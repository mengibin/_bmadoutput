# Story 4.5: Call LLM API (OpenAI ToolCalls / Local LLM)

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime**,
I want to send the composed prompt to the configured LLM and run a ToolCalls-based execution loop,
so that the workflow can progress with tool assistance.

## Acceptance Criteria

1. **LLM API Client & Configuration**
   - Supports OpenAI-compatible API providers (OpenAI, DeepSeek, Local/Ollama).
   - Configurable endpoint, model, and API key (via `RuntimeStore` settings).
   - Uses the unified `messages` format (`system/user/assistant/tool`) per `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` (do not rely on `developer` role).

2. **ToolCalls Execution Loop**
   - Implements the "Agent Loop":
     1. Send `messages` to LLM.
     2. Receive response.
     3. If response contains `tool_calls`:
        - Execute tools via `ToolHost` (Stub/Mock for now, or connect to basic `fs` if ready from Story 4.6).
        - Append `role="tool"` messages with:
          - `tool_call_id = tool_calls[i].id`
          - `content = JSON.stringify(toolResult)` (string-only requirement)
        - Loop back to Step 1.
     4. If response has content (and no tool calls pending):
        - Treat as "stop point" (WaitingUser or Completed).

3. **Anti-Infinite Loop Strategy (Hard Limit + Heuristic)**
   - **Configurable Loop Limit**: Set a hard limit (e.g., max 20 turns) as a failsafe. If reached, enter `Failed` state and notify user.
   - **Consecutive Failure Detection (Heuristic)**:
     - Detect if the Agent makes **identical or effectively identical** tool calls (same tool + same args) for N consecutive times (e.g., 3 times) that result in **Errors** or **No State Change**.
     - If detected:
       - **Automated Intervention** (Optional): Inject a `system` message: "You have failed X times with the same action. Please try a different approach."
       - **Fail-Fast**: If it persists, abort loop and enter `WaitingUser` with a specific warning ("Detected consecutive failures loop").

4. **Stop Point Semantics**
   - Align with `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`:
     - No tool calls + not "completed" -> `WaitingUser` (e.g., asking a clarifying question).
     - "Completed" condition met -> `Completed` (State update triggers this).

5. **Error Handling & Graceful Degradation**
   - Handle API errors (timeout, 401, 500) gracefully (retry or pause execution).
   - Handle max token limits (smart context truncation or error).
   - Allow user to Pause/Stop execution safely (interrupt loop between turns).

## Technical Context

- **Module**: `crewagent-runtime/electron/services/llmAdapter.ts`, `crewagent-runtime/electron/services/executionEngine.ts`
- **Integration**:
  - Receives composed `messages[]` from `PromptComposer` (Story 4.4).
  - Calls `ToolHost` (Story 4.6/4.10) for tool execution.
  - Updates `RuntimeStore` / logs.
- **Protocol**: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`.
- **Dependencies**: `openai` npm package (or standard fetch for custom control).

## Design

### Summary

- **LLMAdapter**: OpenAI-compatible Chat Completions client（DeepSeek 等可复用同一协议）。负责鉴权、超时、重试、响应解析。
- **ExecutionEngine**: 运行期编排器。维护 run 的对话历史（assistant/tool），执行 `LLM ↔ ToolHost` 内循环，并在“停点”返回给 UI。
- **ToolHost（依赖注入）**: 统一的工具执行入口（Story 4.6/4.10 实现），本 Story 只定义接口并在测试中 mock。
- **Loop Detector**: 在 engine loop 内做硬限制（max turns）+ “重复失败 tool call” 侦测，防止无限循环。

### UX / UI
- **Execution Log**: Shows "Calling LLM...", "Tool Call: fs.read...", "Loop Detected: Pausing for User Intervention".
- **Controls**: "Stop" button in UI must trigger an abort signal to the Engine/Adapter.

### API / Contracts
- `LLMAdapter.chatCompletion(messages: Message[], tools: ToolDef[], config: LLMConfig): Promise<LLMResponse>`
- `ExecutionEngine.start(runId: string)`
- `ExecutionEngine.stop(runId: string)`

**OpenAI-compatible ToolCalls 回填规则（必须遵守）**：
- 当 assistant 返回 `tool_calls` 时：
  - Runtime 必须按 `tool_calls[i].id` 执行工具（可串行；保持回填顺序一致）。
  - Runtime 必须追加 `role="tool"` message：
    - `tool_call_id = tool_calls[i].id`
    - `content = JSON.stringify(toolResult)`（tool content 只能是 string）
- 下一次请求的 `messages[]` 必须包含：上一条 assistant（含 tool_calls）+ 对应的 tool messages。

**返回给 UI 的引擎停点语义（对齐 `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`）**：
- 有 `tool_calls` → 继续 loop（不进入用户停点）
- 无 `tool_calls`：
  - `isWorkflowComplete(@state/workflow.md, graph)==true` → `Completed`
  - 否则 → `WaitingUser`

### Data / Storage
- **State**: In-memory `messages[]` during the run (persisted to `RuntimeStore` if we want full history, or just rely on re-construction/log).
- **Logs**:
  - Engine-level audit log persisted to `@state/logs/execution.jsonl` (JSONL; see runtime-spec + protocol doc).
  - UI log panel should also receive structured entries (e.g. via `RuntimeStore.addLog`) so users can see progress in real time.

### Errors / Edge Cases
- **Network Failure**: Retry logic with exponential backoff.
- **Context Window Exceeded**: Strategy to summarize or truncate old history (System Prompt always preserved).
- **Malformatted Tool Calls**: If LLM hallucinates invalid JSON, treat as text or return error to LLM to self-correct.
- **Structured Errors（建议统一 shape）**：`{ code: string, message: string, details?: unknown }`
  - LLM 侧：`LLM_AUTH_FAILED`, `LLM_TIMEOUT`, `LLM_HTTP_ERROR`, `LLM_BAD_RESPONSE`, `LLM_RATE_LIMITED`
  - Tool 侧：`TOOL_ARGS_INVALID_JSON`, `TOOL_NOT_AVAILABLE`, `TOOL_EXEC_FAILED`
  - Engine：`ENGINE_MAX_TURNS_EXCEEDED`, `ENGINE_LOOP_DETECTED`, `ENGINE_ABORTED`

### Test Plan
- Unit tests for `LLMAdapter` (mocking `fetch`/`openai`): verify correct payload structure.
- Integration test for `ExecutionEngine` (mocking LLM): verify loop mechanics (LLM -> Tool -> Result -> LLM).
- **Heuristic Test**: Mock an LLM that returns the same failing tool call 3 times; verify engine stops and enters `WaitingUser`.
- Test cancellation/abort handling.

## Tasks / Subtasks

- [x] Implement `LLMAdapter` service (OpenAI-compatible client wrapper).
- [x] Implement `ExecutionEngine` loop logic (The "Agent Loop").
- [x] Implement **Loop Detector (Consecutive Failure)** logic.
- [x] Define `ToolHost` interface (even if implementation is stubbed for now) to allow the Engine to delegate calls.
- [x] Implement cancellation/pause mechanism (AbortController).
- [x] Add basic unit tests for the Loop logic and Detector.

## Verification

- `npm -C crewagent-runtime run test`
- `npm -C crewagent-runtime run build:ci`

## Validation Notes

Merged from: `_bmad-output/implementation-artifacts/validation-report-story-4-5.md`

- **Verdict**: Approved for `ready-for-dev`.
- **Design must lock in**:
  - ToolCalls 回填：`tool_call_id` + `content = JSON.stringify(toolResult)`
  - Stop point 判定：无 `tool_calls` 时进入 `WaitingUser/Completed`
  - 结构化错误：`{ code, message, details? }`

## References

- Epic: `_bmad-output/epics.md` (Story 4.5)
- Architecture: `_bmad-output/architecture/runtime-architecture.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-5-call-llm-api-openai-toolcalls-local-llm.md`
