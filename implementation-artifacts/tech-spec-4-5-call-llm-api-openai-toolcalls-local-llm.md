# Tech Spec: Story 4.5 - Call LLM API (OpenAI ToolCalls / Local LLM)

**Created:** 2026-01-02  
**Status:** Implemented (Ready for Review)  
**Source Story:** `_bmad-output/implementation-artifacts/4-5-call-llm-api-openai-toolcalls-local-llm.md`

## 1. Objective

Implement the **LLM calling layer** and the **ToolCalls execution loop** inside the Electron Main process so a Run can progress via:

`PromptComposer → LLMAdapter → (tool_calls?) → ToolHost → tool messages → LLMAdapter → … → stop point`

This story is the protocol “spine” for later capabilities (ToolHost sandbox, state mutation validation, MCP drivers).

## 2. Scope

### In Scope
- OpenAI-compatible `chat.completions` calls using `messages/tools/tool_calls/tool` (DeepSeek compatible).
- ExecutionEngine internal loop (LLM ↔ ToolHost) with:
  - stop point semantics (`WaitingUser` vs `Completed`)
  - cancellation (`stop/pause`) via `AbortController`
  - infinite-loop protection (max turns + repeated failing tool calls)
- Structured error model (`{ code, message, details? }`).
- Unit/integration tests with mocks (no real network).

### Out of Scope (Deferred)
- Streaming tokens/toolcall deltas to UI (can be a later enhancement).
- MCP tool execution (Story 4.10).
- Full filesystem sandbox tools (Story 4.6) and graph transition validation on state write (Story 4.7).
- Run creation/init (`@state/workflow.md` copy) (Story 4.8).

## 3. Architecture

### 3.1 Modules (Electron Main)

- `electron/services/llmAdapter.ts`
  - provider-agnostic wrapper around OpenAI-compatible API
  - handles auth header, timeout, retries, response parsing

- `electron/services/executionEngine.ts`
  - owns per-run message history (assistant + tool)
  - orchestrates loop until stop point
  - emits log/events for UI and audit

- `electron/services/toolHost.ts` (interface only in this story; implementation in Story 4.6/4.10)
  - provides tool defs for LLM
  - executes tool calls and returns structured `ToolResult`

### 3.2 Key Dependencies
- Runtime settings: `RuntimeStore.getSettings()` → `RuntimeSettings.llm` (`provider/baseUrl/model/apiKey/timeout`)
- Prompt composer: `PromptComposer.compose(input)` (Story 4.4)
- Protocol spec: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Stop semantics: `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`

## 4. Contracts

### 4.1 OpenAI-compatible request/response (minimum)

Follow `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` as the single source of truth. Minimum required fields:

- Request: `POST {baseUrl}/chat/completions`
  - `model`
  - `messages`
  - `tools` (function tools)
  - `tool_choice: "auto"`

- Response: `choices[0].message`
  - `content` (string|null)
  - `tool_calls?: [{ id, type:"function", function:{ name, arguments } }]`

### 4.2 LLMAdapter API

```ts
export interface LlmAdapterConfig {
  provider: 'openai' | 'openai-compatible' | 'azure' | 'ollama'
  baseUrl: string
  model: string
  apiKey: string
  timeoutSeconds: number
}

export type LlmAdapterError = {
  code:
    | 'LLM_AUTH_FAILED'
    | 'LLM_TIMEOUT'
    | 'LLM_RATE_LIMITED'
    | 'LLM_HTTP_ERROR'
    | 'LLM_BAD_RESPONSE'
    | 'UNKNOWN'
  message: string
  details?: unknown
}

export interface LlmAdapterResult {
  success: boolean
  message?: { role: 'assistant'; content: string | null; tool_calls?: unknown[] }
  usage?: unknown
  error?: LlmAdapterError
}

export interface LlmAdapter {
  chatCompletions(params: {
    messages: unknown[]
    tools?: unknown[]
    config: LlmAdapterConfig
    signal?: AbortSignal
  }): Promise<LlmAdapterResult>
}
```

Provider notes:
- `openai`/`openai-compatible`: identical request shape; use `Authorization: Bearer <apiKey>` when `apiKey` is set.
- `azure`/`ollama`: treat as planned variants; implement only if the endpoint supports OpenAI-compatible routes in MVP.

### 4.3 ToolHost API (interface)

```ts
export type ToolResult = { ok: true; data: unknown } | { ok: false; error: { code: string; message: string; details?: unknown } }

export interface ToolHost {
  getVisibleTools(context: { packageId: string; agentId: string }): unknown[] // OpenAI function tool list
  executeToolCall(params: {
    toolCallId: string
    name: string
    argumentsJson: string
    context: { projectRoot: string; packageId: string; runId: string }
    signal?: AbortSignal
  }): Promise<ToolResult>
}
```

**Tool message backfill rule (hard requirement):**
- Append `role="tool"` with:
  - `tool_call_id` (exactly `toolCallId`)
  - `content = JSON.stringify(toolResult)`

### 4.4 ExecutionEngine API + phases

```ts
export type RunPhase = 'Running' | 'WaitingUser' | 'Completed' | 'Failed'

export type EngineError = { code: string; message: string; details?: unknown }

export interface EngineResult {
  success: boolean
  phase: RunPhase
  assistantText?: string | null
  error?: EngineError
}

export interface ExecutionEngine {
  start(params: { projectRoot: string; packageId: string; workflowId: string; runId: string; activeAgentId: string }): Promise<EngineResult>
  continue(params: { projectRoot: string; packageId: string; workflowId: string; runId: string; userInput: string }): Promise<EngineResult>
  stop(runId: string): void
}
```

Stop point semantics:
- assistant has `tool_calls` → keep `Running` and continue loop
- no `tool_calls`:
  - `isWorkflowComplete(...)` true → `Completed`
  - else → `WaitingUser`

## 5. Engine Loop Details

### 5.1 Message Construction Strategy

ExecutionEngine maintains a per-run `history[]` containing only:
- assistant messages returned by LLM
- tool messages produced by ToolHost

For each loop iteration, construct:
1) `baseMessages = PromptComposer.compose(...)` (system + RUN_DIRECTIVE/NODE_BRIEF/USER_INPUT as needed)
2) `messages = [...baseMessages, ...history]`

Optimization (recommended):
- only include `RUN_DIRECTIVE` + `NODE_BRIEF` when:
  - start/resume
  - user just provided input
  - `currentNodeId` changed (state advanced)

### 5.2 Loop Limits

- Hard cap: `maxTurns` (default 20; configurable later)
- Repeated failure heuristic:
  - fingerprint each tool call: `name + stableStringify(args)`
  - if identical fingerprint repeats N times (default 3) with `ToolResult.ok=false` → stop with `WaitingUser` and show warning

### 5.3 Cancellation

- Each LLM call uses an `AbortController` tied to:
  - UI stop
  - timeout (`timeoutSeconds`)
- Stop only between turns (never interrupt in-progress file writes once ToolHost exists); if interrupted mid LLM call → return `Failed` with `ENGINE_ABORTED`.

## 6. Testing

### 6.1 LLMAdapter tests (unit)
- mock `globalThis.fetch`
- assert:
  - correct URL (`{baseUrl}/chat/completions`)
  - headers include Authorization when apiKey set
  - body includes `messages/tools/tool_choice`
  - timeout abort triggers `LLM_TIMEOUT`
  - malformed JSON/shape triggers `LLM_BAD_RESPONSE`

### 6.2 ExecutionEngine tests (integration with mocks)
- mock LLMAdapter returning:
  - tool call → tool result → final assistant message (no tool calls) → `WaitingUser`
- mock ToolHost returning:
  - ok tool result (and ensure tool message uses `tool_call_id` + JSON string content)
- heuristic test:
  - same failing tool call 3x → engine stops with `WaitingUser` and `ENGINE_LOOP_DETECTED`
- abort test:
  - stop invoked during in-flight request → returns `ENGINE_ABORTED`

Run via: `npm -C crewagent-runtime run test`

## 7. Implementation Checklist (Dev)

- [x] Add `electron/services/llmAdapter.ts`
- [x] Add `electron/services/executionEngine.ts`
- [x] Define `ToolHost` interface (`electron/services/toolHost.ts`)
- [x] Add unit tests for `llmAdapter`
- [x] Add integration tests for `executionEngine` (mock adapter + tool host)
- [x] Add structured log events (wire to existing `RuntimeStore.addLog` for UI)
