# Story 5.14: Tool Policy + ChatMode Alignment (SystemToolPolicy + mode=chat)

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime user**,
I want Chat / Agent / Workflow 的工具规则与提示注入逻辑一致且可配置,
so that 我能安全、可预期地使用 tools，并避免文档/实现不一致造成的误用与调试成本。

## Acceptance Criteria

1. **ChatMode prompt 对齐**
   - **Given** UI 处于 `ConversationType=chat`
   - **When** 用户通过 `chat:send` 发送消息
   - **Then** Runtime 使用 `PromptComposer.compose({ mode: 'chat' })`
   - **And** 不注入 agent persona（保持通用聊天体验）
   - **And** 仍发送 ToolPolicy（来自合并后的有效策略）

2. **Tools 默认开启 + 系统级配置**
   - **Given** 未显式配置 `agent.tools`
   - **When** 进入 Chat/Agent/Run
   - **Then** 默认可用 tools 为开启（至少 builtin `fs.*`、`ui.ask_user`）
   - **And** Settings 的 SystemToolPolicy 可系统级禁用/限额工具能力

3. **`agent.tools` 只能收紧**
   - **Given** SystemToolPolicy 启用某类 tool
   - **When** `agent.tools` 禁用该 tool 或降低限额
   - **Then** 有效策略以“更严格者”为准（enabled=AND；limits=min）
   - **And** 若 SystemToolPolicy 禁用，则 `agent.tools` 不能重新开启

4. **声明与执行一致**
   - **Given** Prompt 中声明的 ToolPolicy（system message）
   - **When** LLM 发起 tool call
   - **Then** ToolHost 的实际可执行性与限额与该 ToolPolicy 一致（避免“声明允许但执行拒绝/反之亦然”）

5. **Settings 持久化与 UI 可配置（MVP）**
   - **Given** 用户在 Settings 中调整 SystemToolPolicy
   - **When** 保存设置并再次发起 Chat/Run
   - **Then** 新策略生效并持久化到 RuntimeStore settings

## Design

Design complete. Status set to `ready-for-dev`.

### UX / UI
- **Settings → Tools（SystemToolPolicy）**
  - **fs.\***：
    - Checkbox：`Enable filesystem tools (fs.*)`（全局开关，默认开启）。
    - Limit inputs：
      - `Max read bytes`（默认 `524288` = 512KB）
      - `Max write bytes`（默认 `2097152` = 2MB）
    - 交互约束：不允许把 limit 保存成 `undefined`（空值视为“恢复默认值”，或保留旧值），避免运行时退回到更小的内部默认（`50_000/100_000`）。
    - 文案提示：关闭 `fs.*` 后，Chat/Agent/Run 都将无法执行文件读写（但 `ui.ask_user` 仍可用）。
  - **mcp**：
    - Checkbox：`Enable MCP tools`（默认关闭）。
    - MVP 不提供 `allowedServers` 的 UI；启用 MCP 的 allowlist 作为后续增强（安全起见，空列表视为不放行任何 server）。

- **Works → ChatMode**
  - ChatMode 只体现“通用聊天”：不显示/不依赖 persona；但工具可用性由 Settings 的 SystemToolPolicy 控制。
  - ChatMode 仍支持 widgets：当需要结构化输入时可触发 `ui.ask_user` 并在聊天流中嵌入 widget。

### API / Contracts
- `RuntimeSettings` 扩展 `tools`（SystemToolPolicy）。
- 合并规则：SystemToolPolicy 作为上限基线，`agent.tools` 只能收紧（enabled=AND；limits=min）。
- PromptComposer 的 ToolPolicy 声明使用“合并后的有效策略”。
- ToolHost enforcement 读取同一份“合并后的有效策略”。

**Effective ToolPolicy（必须同源）**
- `effectivePolicy = mergeToolPolicies({ system: settings.tools, agent: agentDefinition.tools })`
- 规则（MVP）：
  - `enabled`: `system.enabled && (agent.enabled ?? true)`
  - `maxReadBytes/maxWriteBytes`: 取 `min(system, agent)`（忽略 `undefined`）
  - `allowedServers`: `intersection(system.allowedServers, agent.allowedServers)`（任一缺失则回退另一侧；两者都有则取交集）

**EntryPoints**
- ChatMode：`ipcRenderer.sendChat` → `ipcMain('chat:send')` → `callAgentChat({ mode:'chat' })`
- AgentMode fallback chat：`agent:dispatch` 的 `ResolvedCommand.Chat` → `callAgentChat({ mode:'agent' })`
- RunMode：ExecutionEngine 内部使用 `effectiveAgentId`（node agentId 优先）来合并 tools 并暴露工具。

### Data / Storage
- Settings 持久化：`RuntimeStoreRoot/settings.json`（字段：`settings.tools`）。
- Back-compat：`loadSettings()` 必须用 `defaultSettings` 合并（缺字段回填默认）。
- 更新语义：`settings:update` 仅做字段级 merge，且对 `tools.*` 需忽略 `undefined`（空值不覆盖现有值），避免运行中出现意外降级。

### Errors / Edge Cases
- `fs.enabled=false` 时，所有 `fs.*` tool call 返回清晰拒绝错误。
- settings 旧版本无 `tools` 字段时，自动补 defaults，不影响启动。
- ChatMode 不注入 persona；AgentMode 仍注入 persona；RunMode 保持 RUN_DIRECTIVE + NODE_BRIEF。
- `ui.ask_user` 必须在 `fs.enabled=false` 时仍可用（避免 widget 能力被误伤）。
- LLM 仍可能 hallucinate `fs.*` 工具：即使 prompt 不暴露，ToolHost 也必须拒绝（双保险）。

### Test Plan
- **Unit**
  - `mergeToolPolicies`：enabled=AND；limits=min（忽略 undefined）；allowedServers 交集。
  - `PromptComposer`：
    - `mode=chat` 不包含 persona；`mode=agent`/`mode=run` 包含 persona。
    - ToolPolicy 文本与 merged policy 一致。
  - `FileSystemToolHost`：
    - fs disabled：`getVisibleTools()` 仅暴露 `ui.ask_user`；执行 `fs.*` 返回 `TOOL_NOT_AVAILABLE`。
    - fs enabled：限额生效（read/write 超限行为按既有实现）。
  - `electron/main.ts`：
    - `chat:send` 走 `mode='chat'`（不注入 persona），但仍可触发 `ui.ask_user`。

## Tasks / Subtasks

- [x] 1) 增加 SystemToolPolicy 到 RuntimeSettings（AC: 2,5）
  - [x] Electron RuntimeStore：`RuntimeSettings.tools` schema + defaults + load/merge（缺字段时回填 defaults）
    - [x] 默认值：`fs.enabled=true`（含 `maxReadBytes/maxWriteBytes`），`mcp.enabled=false`
    - [x] `settings:update` 仅允许“合并覆盖”，避免把其他 settings 字段覆盖丢失（并修复 tools limits 显式 undefined 的 in-memory 降级）
  - [x] Renderer AppStore：增加/维护 `systemToolPolicy` 状态并通过 IPC 持久化（读取 `settings:get`，保存 `settings:update`）
  - [x] SettingsPage：新增 Tools UI（fs 开关/限额、mcp 开关）并保存到 settings
- [x] 2) 统一 ToolPolicy 合并逻辑（AC: 2,3,4）
  - [x] 抽取 `mergeToolPolicies({ system, agent })`（enabled=AND；limits=min；`allowedServers`=intersection）
  - [x] Prompt 侧：Chat/Agent/Run 都注入 “合并后的 ToolPolicy”（同一个 merge 函数），确保声明与执行一致
    - [x] Agent/Chat：`callAgentChat` 先 merge，再把 merged policy 写入传入 PromptComposer 的 `agentDefinition.tools`
    - [x] Run：ExecutionEngine 使用 `effectiveAgentId` 对应 agent.tools 与 SystemToolPolicy merge
  - [x] ToolHost enforcement：FileSystemToolHost 读 `runtimeStore.getSettings().tools` 并与 agent.tools 合并后再决定：
    - [x] `getVisibleTools()`：fs disabled 时仅保留 `ui.ask_user`
    - [x] `executeToolCall()`：fs disabled 时所有 `fs.*` 返回 `TOOL_NOT_AVAILABLE`
- [x] 3) ChatMode entrypoint 对齐（AC: 1）
  - [x] `chat:send` 使用 `PromptComposer.compose({ mode: 'chat' })`（不注入 persona，但仍注入 ToolPolicy）
  - [x] AgentSession 的 fallback chat 仍使用 `mode='agent'`（注入 persona）
- [x] 4) 测试与回归（AC: 1~5）
  - [x] 单测：`mergeToolPolicies`（enabled=AND；limits=min；allowedServers intersection）
  - [x] 单测：PromptComposer `mode=chat` 不包含 persona；`mode=agent` 包含 persona；toolPolicy 文本与 merged policy 一致
  - [x] 单测：FileSystemToolHost 在 fs disabled 时仅暴露 `ui.ask_user`，且拒绝 `fs.*`
  - [x] 单测：`updateSettings()` 收到显式 undefined 不会导致 tools limits 在内存中降级
  - [x] 单测：Agent chat / chat:send 走 `mode=chat`（不注入 persona）但仍可触发 `ui.ask_user`
  - [x] 运行：`cd crewagent-runtime && npm test`；`npm run lint`

## Dev Notes

### Context / Current Gap（为什么要做）

- 入口层目前存在三种“对话模式”：Chat / Agent / Run（workflow）。
- 历史问题主要来自两个不一致：
  - Prompt 里声明的 ToolPolicy ≠ ToolHost 实际执行策略（导致“看起来能用但实际被拒/反之亦然”）
  - ChatMode 被错误当成 AgentMode（persona 被注入，违背文档对 ChatMode 的定义）

### Implementation Guardrails（避免回归）

- ToolPolicy 只允许“越收越紧”：SystemToolPolicy 为系统上限基线，agent.tools 只能收紧；严禁 agent 反向放开系统禁用项。
- 同一个 merge 函数必须同时用于：
  - PromptComposer 的 ToolPolicy 声明
  - ToolHost 的可见 tools + enforcement
- ChatMode：
  - `PromptComposer mode=chat`：不注入 persona / 不绑定 workflow state（不发 RUN_DIRECTIVE/NODE_BRIEF）
  - 仍可用 tools（至少 `ui.ask_user`，fs 受 policy 控制）
- `ui.ask_user` 需在 fs disabled 时仍可用（保证 widget/交互能力不被误伤）。

- 主要改动点：
  - `crewagent-runtime/electron/services/toolPolicy.ts`
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

### References

- Spec / docs
  - `crewagent-runtime/docs/entrypoints-agent-vs-workflow.md`
  - `crewagent-runtime/docs/prompt-composer-examples.md`
  - `crewagent-runtime/docs/runtime-spec.md`
  - `crewagent-runtime/docs/runtime-architecture.md`
  - `crewagent-runtime/docs/llm-conversation-protocol-openai.md`
  - `crewagent-runtime/docs/agent-menu-command-contract.md`
- Related runtime code
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/services/chatToolLoop.ts`
  - `crewagent-runtime/electron/services/promptComposer.ts`
  - `crewagent-runtime/electron/services/toolPolicy.ts`
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `crewagent-runtime/electron/services/executionEngine.ts`
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/src/stores/appStore.ts`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
  - `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- `crewagent-runtime`: `npm test`
- `crewagent-runtime`: `npm run lint`

### Completion Notes List

- ChatMode 入口（`chat:send`）使用 `PromptComposer mode=chat`：不注入 persona，但仍按合并后的 ToolPolicy 暴露/执行工具。
- 新增 SystemToolPolicy（Settings）作为系统级工具开关/限额，并与 `agent.tools` 按“只能收紧”规则合并（enabled=AND；limits=min）。
- ToolPolicy 的“对模型声明”与 ToolHost 的“硬执行”使用同一份合并策略，避免声明与执行不一致。
- 修复 Settings 保存时 tools limits 为空导致的 in-memory 降级（显式 `undefined` 会重置到默认值而非落到 ToolHost 内部默认）。

### File List

- `_bmad-output/implementation-artifacts/5-14-tool-policy-and-chatmode-alignment.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/services/toolPolicy.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/electron/services/chatToolLoop.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/promptComposer.test.ts`
