# Tech Spec: Story 4.11 - Agent-First Session (Menu) & Command Routing

**Created:** 2026-01-03  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/4-11-agent-first-session-menu-and-command-routing.md`

## Overview

### Problem Statement

当前 Runtime 已有 Agent/Chat/Workflow 三种会话视图（`WorksPage` / `RunsPage`），但在 “Agent Session” 场景仍缺少 BMAD/Cursor 风格的确定性行为：

- UI 的 Agent 菜单仅用于展示/拼接输入，`handleSend()` 只是把文本写入本地 Conversation（未触发任何后端解析/执行）。
- Electron Main 尚未提供 **命令解析**（resolve）与 **命令调度**（dispatch）的 IPC 合约。
- Chat 模式没有 `chat:send` → `LLMAdapter` 的桥接；RunMode 也没有 `runs:continue` → `ExecutionEngine.continue` 的桥接。

这会导致：用户无法 “选 agent → 看菜单 → 输入触发 workflow/exec/action/聊天”，也无法在 RunMode 中把用户输入交给 ExecutionEngine 继续执行。

### Solution Summary

按 `_bmad-output/tech-spec/agent-menu-command-contract.md` 实现一个 Main 侧的确定性路由链路：

1) **CommandRouter**：把用户输入解析为 `ResolvedCommand`（数字选择/trigger/alias/handler/fuzzy/ClarifyChoice/Chat）。  
2) **Dispatch**：根据 `ResolvedCommand.kind` 调用 `RunManager.createRun` / `LLMAdapter.chatCompletions` / 内建动作，并返回 UI 可消费的 event。  
3) **IPC**：暴露 `agent:resolveCommand`、`agent:dispatch`、`chat:send`、`runs:continue`（统一 `{success,error}` 包装）。  
4) **UI**：`WorksPage` 在 AgentMode 不再直接拼接文本，而是调用 resolve+dispatch 并将结果写回 conversation / activeRun。

## Scope

### In Scope

- Electron Main 新增：
  - `crewagent-runtime/electron/services/commandRouter.ts`（解析逻辑严格对齐 contract）。
  - `agent:resolveCommand` IPC（返回 `ResolvedCommand`）。
  - `agent:dispatch` IPC（执行/调度并返回 UI event）。
  - `chat:send` IPC（Agent/Chat mode 的非流式 LLM 请求，返回 assistant message）。
  - `runs:continue` IPC（RunMode 输入 → `ExecutionEngine.continue`）。
- Renderer UI 修改：
  - `WorksPage.tsx`：AgentMode 下的 `handleSend()` 改为 resolve+dispatch，并处理 `ClarifyChoice` / `ShowMenu` / `RunStarted` / `ChatResponse`。
- targetSurface：
  - Electron-only：Main 侧固定 `targetSurface='electron'`，执行 `ide-only/web-only` gating（web-only 在 Electron 下隐藏）。

### Out of Scope / Deferred

- LLM streaming（SSE/partial tokens）与 UI 流式渲染。
- 完整 `ScriptRun`（可恢复 state / logs / tool loop）。MVP 可将 `ExecScript` 降级为 “chat + 注入 exec 内容”，不做可恢复 run。
- Web 端（`targetSurface='web'`）支持。

## Technical Decisions

### 1) ClarifyChoice 不引入跨轮次状态

为避免 “澄清候选列表” 需要额外保存临时映射，本实现约定：

- `ClarifyChoice.candidates[].index` 使用 **可见菜单项的 1-based index**（与数字选择一致）。
- multi-handler 只参与 trigger/fuzzy 的命中与评分；若需要澄清，返回的是 **父 menuItem** 维度的候选（避免同 index 多 handler 的歧义）。

### 2) UI 选中菜单 + “补充描述” 的输入协议

`WorksPage` 当前允许用户点选一个 menu 按钮，再输入补充描述。MVP 建议：

- 把菜单选择作为 `input`（优先使用 `trigger`，无 trigger 则用 `description`）。
- 把补充描述作为 `details`（由 UI 拼成 `input + "\n" + details` 发送到 resolve；Router 只用第一行做匹配；Dispatch 在 Chat/ExecScript 时将第二行作为 userInput 的补充）。

### 3) StartWorkflow workflowRef 的落地方式（与 RunManager 对齐）

契约允许 workflowRef 为 `{type:'workflowId'}` 或 `{type:'packagePath'}`。但当前 `RunManager.createRun` 只接受 `workflowRef?: string`（workflow id）。

因此在 `CommandRouter`/dispatch 中统一把 workflowRef 归一化为 **workflowId**：

- 若 menuItem.workflow 为 `bmad.json.workflows[].id` → 使用该 id。
- 若 menuItem.workflow/exec 为 `.../workflow.md` 路径 → 通过 `RuntimeStore.getPackageWorkflows(packageId)` 反查匹配的 workflowId。
- 找不到则返回 `UNKNOWN_WORKFLOW`。

### 4) Tools in Agent/Chat mode（MVP）

由于 `FileSystemToolHost.executeToolCall` 需要 `workflowId` 才能加载 graph，MVP 的 `chat:send` 默认 **不下发 tools**（避免 tool calls）。

后续若要支持 chat 下 `fs.read`，可引入一个不依赖 workflow graph 的 “ChatToolHost”（仅支持 `@project/@pkg` 的 read/list/search）。

## IPC Contracts (Implementation-Ready)

> 详见故事文件 `## Design -> Return Shapes & Error Model`。本 Tech Spec 只强调实现点与落地文件。

- `agent:resolveCommand`
  - input: `{ projectRoot, packageId, agentId, input, context:{activeRunId?} }`
  - output: `{ success:true, command }` 或 `{ success:false, error }`
- `agent:dispatch`
  - input: `{ projectRoot, packageId, agentId, command, conversationId? }`
  - output: `{ success:true, event }` 或 `{ success:false, error }`
- `chat:send`
  - input: `{ projectRoot, packageId, agentId, conversationId?, input }`
  - output: `{ success:true, assistant, usage? }` 或 `{ success:false, error }`
- `runs:continue`
  - input: `{ projectRoot, packageId, workflowId, runId, activeAgentId, userInput }`
  - output: `EngineResult`（`success/phase/assistantText/error`）

## Files & Modules (Planned)

- Main / Services
  - `crewagent-runtime/electron/services/commandRouter.ts` (new)
  - `crewagent-runtime/electron/services/agentDispatcher.ts` (optional split; can be in main.ts first)
  - `crewagent-runtime/electron/main.ts` (add IPC handlers + instantiate `ExecutionEngineImpl` + llmAdapter + toolHost)
- Preload / Types
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/electron/electron-env.d.ts`
- Renderer
  - `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
  - `crewagent-runtime/src/stores/appStore.ts`（若需要把 dispatch 结果写入 run/会话）
- Tests
  - `crewagent-runtime/electron/services/commandRouter.test.ts` (new)
  - `crewagent-runtime/electron/main.*`（可选：IPC shape smoke test）

## Implementation Plan

### Step 1 — CommandRouter (Resolve Only)

- [ ] 定义 `ResolvedCommand`/`CommandKind` 与契约一致（可直接引用契约文件中的类型结构）。
- [ ] 实现候选集构建：
  - 可见菜单项（按顺序，1-based）
  - handler candidates（`triggers[].type === 'handler'`）
  - alias triggers（`triggers[].type === 'alias'`）
  - cmd（若存在，兼容 BMAD）
- [ ] 匹配优先级（必须）：
  1) `input.trim() === ''` → `ShowMenu`
  2) `/^[0-9]+$/` 数字选择（越界 → `ClarifyChoice`）
  3) 精确 trigger 命中（大小写不敏感；支持前导 `*`）
  4) 模糊匹配（MVP：substring；并给出 score 排序）
  5) 未命中 → `Chat`

### Step 2 — Dispatch (Main)

- [ ] `StartWorkflow`：
  - resolve workflowId（如上“Technical Decision 3”）
  - 调用 `RunManager.createRun({projectRoot, packageId, workflowRef: workflowId, activeAgentId})`
  - 返回 `{type:'RUN_STARTED', run}`
- [ ] `ResumeRun`：
  - 调用 `RunManager.listRuns(projectRoot)`，取最后一个（或按 `createdAt` 排序后取最新）
  - 返回 `{type:'RUN_RESUMED', run}`
- [ ] `RunAction` builtin：
  - `menu.show` → `{type:'SHOW_MENU'}`
  - `agent.dismiss` → `{type:'AGENT_DISMISSED'}`
  - `run.resume` → `ResumeRun`
- [ ] `RunAction` promptId / inline：
  - promptId 未找到 → `UNKNOWN_PROMPT_ID`
  - 找到 prompt / inline → 走 `chat:send`（把 prompt content 作为 userInput）
- [ ] `ExecScript`（MVP）：
  - 读取 exec markdown（`@pkg/...` 或 `@project/...`）
  - 将 exec 内容与 data（若有）作为额外上下文注入到 `PromptComposer`（或以 user message 方式注入）
  - 调用 `LLMAdapter.chatCompletions`（不提供 tools）
  - 返回 `{type:'CHAT_RESPONSE', assistant, usage?}`

### Step 3 — IPC Wiring + UI Hookup

- [ ] `ipcMain.handle('agent:resolveCommand', ...)`
- [ ] `ipcMain.handle('agent:dispatch', ...)`
- [ ] `ipcMain.handle('chat:send', ...)`
- [ ] `ipcMain.handle('runs:continue', ...)`
- [ ] `preload.ts` + `electron-env.d.ts` 同步暴露类型化方法。
- [ ] `WorksPage.tsx` 改为：
  - AgentMode：先 resolve 再 dispatch；按 event 更新 UI（追加 assistant message / setActiveRun / show clarify）
  - ChatMode：直接 `chat:send` 并追加 assistant message
  - RunMode：调用 `runs:continue` 并把 `assistantText` 追加到对话

## Test Plan

- Unit（vitest）：`commandRouter.test.ts`
  - 空输入 → ShowMenu
  - 数字选择（in-range/out-of-range）
  - trigger/cmd/alias/handler.match 的精确与模糊匹配优先级
  - `ide-only/web-only` gating 对候选集与 index 的影响（Electron 下 web-only 必须隐藏）
  - 多命中 → ClarifyChoice（候选 index 为 menu index）
- Integration（vitest 可选）：dispatch
  - StartWorkflow 能调用 RunManager.createRun 并返回 RUN_STARTED
  - run.resume 能从 runsIndex 取最新并返回 RUN_RESUMED
  - chat:send 返回 assistant message，并对 LLM error code 做透传/包装

## Acceptance Criteria Checklist

- [ ] Agent Session 输入可被解析为 `ResolvedCommand`（数字/trigger/fuzzy/handler/clarify/chat）。
- [ ] `StartWorkflow` 走 `RunManager.createRun` 并返回 run metadata，UI 进入 RunMode。
- [ ] `chat:send` 可在 Agent/Chat mode 返回 assistant 消息（MVP 非流式）。
- [ ] `runs:continue` 可把 RunMode 的用户输入交给 `ExecutionEngine.continue`。
- [ ] Electron-only gating 生效：`web-only` 菜单项在 Electron 隐藏。
