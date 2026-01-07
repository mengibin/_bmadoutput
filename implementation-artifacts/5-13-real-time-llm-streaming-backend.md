# Story 5-13: Real-Time LLM Streaming Backend (SSE + IPC)

## Overview
**Epic**: 5 – Observability, Settings & Recovery  
**Priority**: High  
**Status**: `review`

## Goal
实现真正的后端流式输出（SSE → IPC → Renderer），让 Chat/Run 在 LLM 生成期间持续推送 token，并保持现有工具调用与最终消息持久化行为不变。

## Story

As a **Consumer**,  
I want the runtime backend to stream LLM responses token-by-token,  
So that the chat UI updates in real time without waiting for the full response.

## Context / Current Gap
- Renderer 侧（Story 5-11）已完成 `llm:stream-*` 事件监听与 UI 渲染，但目前：
  - 后端仍走非流式（`llmAdapter` 固定 `stream: false`）
  - 主进程未广播任何 `llm:stream-*` 事件（导致 UI 收不到真实 token 流）
  - 工具调用与“思考/内部事件”尚未按 Iceberg Model 产出对应的 stream 事件

## Business Value
- **即时反馈**：用户可以看到模型正在生成，提高信任感与可用性。
- **更好的控制**：在长响应时可中断/停止，不必等待完整返回。
- **为 5-11 UI 流式体验补齐后端**：前端已有监听，但缺少真实数据流。

## Acceptance Criteria

### 1) 流式响应（SSE → IPC）
- **Given** 选择的 LLM provider 支持 streaming（OpenAI / OpenAI-compatible）
- **When** runtime 发起 `chat.completions` 且 `stream: true`
- **Then** 后端解析 SSE 并按序发出：
  - `llm:stream-start`（稳定的 `messageId`）
  - `llm:stream-chunk`（`delta` + 可选 `partType`）
  - `llm:stream-end`（响应完成）

### 2) 工具调用事件
- **Given** assistant 触发 tool calls
- **When** tool 调用开始/结束
- **Then** 后端发出 `llm:stream-tool-start` 和 `llm:stream-tool-end`
- **And** end 事件包含 `{ result, duration }`

### 2.1) Widget 触发兼容（Story 5.12）
- **Given** tool call 为 `ui.ask_user`（或等价 widget 请求）
- **When** 需要等待用户交互
- **Then** 发出 `llm:stream-tool-start`
- **And** 同步触发 `widget:request`（由 5.12 处理）
- **And** 本次流式输出结束或进入等待态（不再继续追加 chunk）

### 3) 兼容回退
- **Given** provider 不支持 streaming 或流式解析出错
- **When** 请求失败或需要降级
- **Then** 自动回退到非流式路径
- **And** 仍返回完整 assistant message（不破坏 UI 与持久化）

### 4) 中断与清理
- **Given** 用户中止/停止运行
- **When** streaming 正在进行
- **Then** 流式连接被安全终止，且不残留监听/计时器

### 5) Iceberg Model 事件映射
- **Given** UI 依赖 `MessagePartType`（`content`/`thinking`/`tool_start`/`tool_result`）
- **When** 后端接收到 tool call / 产生内部执行细节
- **Then** 需要能通过 `llm:stream-*` 事件表达这些片段（至少覆盖 tool_start/tool_end）
- **And** payload 结构与 `electron-env.d.ts`/preload 暴露的方法保持一致

## Technical Components / Changes
1. **LLM Adapter Streaming**
   - 在 `llmAdapter` 中增加 streaming 支持（SSE 解析，`data:` 行与 `[DONE]` 处理）
   - 解析 `delta.content`、`delta.tool_calls`，累积形成最终 assistant message
   - 支持 `AbortSignal` 与超时逻辑

2. **ExecutionEngine + chatToolLoop**
   - 接收 streaming 回调（start/chunk/end）
   - 在 tool 执行前后发出 tool-start/tool-end
   -（可选）将关键内部事件映射为 `partType='thinking'`（例如：规划/路由/关键决策），避免 UI 只能显示 “Thinking...” 却无数据源
   - 保证最终聚合的 assistant message 仍返回用于持久化

3. **Electron IPC 广播**
   - 在 `electron/main.ts` 增加 `sendIpcEvent` + `createLlmStreamBroadcaster`（可按调用方窗口 sender 发送，避免多窗口/多页面串流）
   - 发送 `llm:stream-*` 事件到发起调用的 renderer（缺省可回退为 broadcast）
   - `messageId` 需稳定（推荐：`{runId}:{turn}:{seq}` 或 `chat-{conversationId}:{turn}`）

4. **类型与协议**
   - 补充 OpenAI streaming delta 类型（`openaiProtocol.ts`）
   - 若 payload 结构变化，更新 `electron-env.d.ts`

## Verification Plan

### Manual Verification
1. Chat 模式发送长消息：UI 实时流式渲染 + 自动滚动。
2. Run 模式执行 workflow：流式响应 + 工具调用指示器正常触发。
3. 触发 `ui.ask_user`（5.12）：应弹出 widget，并进入等待态。
4. 中断运行：流式停止，不再追加内容。
5. 断网/解析错误：自动回退到非流式，UI 不崩溃。

### Automated Tests
- `llmAdapter` SSE 解析单测（包含 `delta.content` 与 `delta.tool_calls`）。
- `executionEngine` / `chatToolLoop`：流式回调触发顺序与最终 message 聚合。

## Design

### UX / UI
- UI 已在 5-11 完成监听与渲染，本故事不改 UI 组件，仅保证事件可用。
- 失败回退时 UI 仍需能显示完整消息（无流式）。

### API / Contracts
- IPC 事件协议：
  - `llm:stream-start` / `llm:stream-chunk` / `llm:stream-end`
  - `llm:stream-tool-start` / `llm:stream-tool-end`
- **Widget 兼容**（Story 5.12）：
  - `ui.ask_user` tool call → `widget:request` IPC
  - 触发后进入 `waiting-user`（不继续流式 chunk）
  - 同时仍可发 `llm:stream-tool-start` 作为 UI 指示器
- `messageId` 稳定格式：Run `run:{runId}:{turn}` / Chat `chat:{conversationId}:{turn}`。
- `stream-chunk` payload：`{ messageId, delta, partType? }`。

### Data / Storage
- 持久化保持现有路径：最终聚合 message 仍写入 `messages.json`。
- Streaming 仅为运行时 UI 事件，不写入持久化。
- `ui.ask_user` 的 widget payload 由 5.12 负责持久化，本故事仅负责触发。

### Errors / Edge Cases
- Provider 不支持 streaming → 自动回退非流式。
- SSE 解析错误 / 超时 → 中止流式并回退。
- Abort/cancel → 立即终止流式连接并停止事件推送。
- `ui.ask_user` 触发后必须暂停继续输出，避免 UI 混乱。

### Test Plan
- 单测：SSE 解析、tool_calls 聚合、`[DONE]` 终止。
- 集成：ExecutionEngine/chatToolLoop 回调顺序与 tool-start/end 触发。

## Tasks / Subtasks
- [x] 1. 设计 streaming 接口（`llmAdapter` 回调/API 形式）
  - [x] 1.1 定义 SSE delta 类型与聚合规则
  - [x] 1.2 确定 `messageId` 生成策略
  - [x] 1.3 定义 `partType` 映射策略（已覆盖 `content`，tool 通过 `llm:stream-tool-*` 事件表达；`thinking` 暂不输出）
- [x] 2. 实现 LLM streaming（OpenAI / OpenAI-compatible）
  - [x] 2.1 在 `llmAdapter` 中解析 SSE 与 `[DONE]`
  - [x] 2.2 支持 Abort/超时/错误回退（解析失败/非支持自动回退非流式）
- [x] 3. 打通 ExecutionEngine / chatToolLoop
  - [x] 3.1 绑定 onStream 回调并转发 IPC
  - [x] 3.2 工具调用前后发送 tool-start/tool-end（含 result + duration）
- [x] 4. Electron IPC 广播
  - [x] 4.1 `main.ts` 添加 stream 事件发送器（可 sender-scoped）
  - [x] 4.2 连接到 Run/Chat 入口（覆盖两条路径）
- [x] 5. 测试
  - [x] 5.1 streaming 单测（SSE content/tool_calls + fallback）
  - [x] 5.2 engine/toolloop 流式顺序与 tool-start/end 单测

## Dev Notes
- 重点改动路径：
  - `crewagent-runtime/electron/services/llmAdapter.ts`
  - `crewagent-runtime/electron/services/openaiProtocol.ts`
  - `crewagent-runtime/electron/services/executionEngine.ts`
  - `crewagent-runtime/electron/services/chatToolLoop.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`（仅需确认事件已对外）

## Related Artifacts
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-13-real-time-llm-streaming-backend.md`

## Dev Agent Record

### Agent Model Used
GPT-5.2 (Codex CLI)

### Debug Log References
`crewagent-runtime`: `npm test`, `npm run lint`

### Completion Notes List
- LLM Adapter 支持 OpenAI / OpenAI-compatible 的 SSE streaming，并聚合最终 assistant message（content + tool_calls）。
- ExecutionEngine/chatToolLoop 通过回调转发 `llm:stream-*`，并在工具调用前后发送 `llm:stream-tool-start/end`（含 duration + result）。
- IPC 事件默认发送到发起调用的 renderer（`event.sender`），减少多窗口/多页面串流干扰；无 sender 时可广播。

### File List
- crewagent-runtime/electron/services/openaiProtocol.ts
- crewagent-runtime/electron/services/llmAdapter.ts
- crewagent-runtime/electron/services/executionEngine.ts
- crewagent-runtime/electron/services/chatToolLoop.ts
- crewagent-runtime/electron/main.ts
- crewagent-runtime/electron/services/llmAdapter.test.ts
- crewagent-runtime/electron/services/chatToolLoop.test.ts
- crewagent-runtime/electron/services/executionEngine.test.ts

## Change Log
- 2026-01-05: 实现 SSE → IPC 流式后端（含工具事件、回退与单测）。
