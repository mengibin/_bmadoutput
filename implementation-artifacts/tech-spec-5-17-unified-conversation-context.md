# Tech Spec: Story 5-17 – Unified Conversation Log + Cross-Mode LLM Context (Codex-style)

> Architecture reference: keep this document as a long-term v2 design note for CrewAgent’s conversation/context system.

## 1. Summary

This spec proposes a **Codex-style** approach to conversation handling:

1) **Store everything** in a single Conversation log (user-visible chat + internal LLM/tool/execution events + errors/partial output).
2) For every LLM call (Chat / Agent / Run), build a request from:
   - **Dynamic System Prompt** (depends on current mode)
   - **Conversation context window** (selected from the unified log with strict budget rules)
   - **Run-only injected context** (RUN_DIRECTIVE / NODE_BRIEF / step context) when in workflow execution

Mode switching (Run / Agent / Chat) **does not reset history**; it only changes the system prefix used to build the next request.

---

## 2. Goals / Non-Goals

### Goals
- Single source of truth for all conversation-related records:
  - UI-visible messages
  - LLM assistant messages (including tool calls)
  - Tool results (`role=tool`)
  - LLM failures (timeouts/network), including partial streamed output
- Every LLM call uses **conversation history** (within a budget) so “继续/continue” has an anchor.
- Switching modes updates only the **system prompt**, not the message log.
- Preserve OpenAI tool-call protocol ordering (`assistant(tool_calls)` → `tool(result)` → `assistant(content)`).

### Non-Goals (MVP)
- Perfect long-term memory with unlimited context; budget enforcement is required.
- Cross-conversation memory.
- Full autonomous summarization pipeline (can be added in v2 as optional).

---

## 3. Current State (Observed)

### 3.1 Storage
- Conversations and messages are persisted per project:
  - `.../conversations/index.json`
  - `.../conversations/<conversationId>/messages.json`
- Stored messages are primarily user-visible (`role: user/assistant/system`), not OpenAI protocol-complete.

### 3.2 Context Building
- **Chat mode**: `chat:send` builds base prompt + current input only; does **not** include prior conversation messages.
- **Agent mode**: similar; not consistently contextual across turns.
- **Run mode**: `ExecutionEngine` uses an in-memory `session.history` and not the persisted conversation messages.

Result: failures/partial output may not appear in the UI message timeline; “continue” can be ambiguous after errors.

---

## 4. Proposed Design

### 4.1 Data Model: Persist a Protocol-Complete Conversation Log

#### Key idea
Persist entries that can be deterministically converted to OpenAI `messages[]` **without losing tool-call structure**.

#### Option A (Recommended): Extend `ConversationMessage` to be a superset of `OpenAIChatMessage`
Extend `ConversationMessage.role` to include `tool`, and add the fields required by tool-call protocol:

```ts
type ConversationRole = 'system' | 'user' | 'assistant' | 'tool'

interface ConversationMessage {
  id: string
  role: ConversationRole
  createdAt: string

  // OpenAI-compatible payload (protocol-complete)
  content?: string | null
  tool_calls?: OpenAIToolCall[]          // assistant only
  tool_call_id?: string                  // tool only

  // UI extras (existing)
  partType?: MessagePartType
  toolName?: string
  duration?: number
  isCollapsed?: boolean
  widget?: WidgetPayload

  // Context metadata (new)
  mode?: 'chat' | 'agent' | 'run'
  runId?: string
  workflowId?: string
  agentId?: string
  includeInContext?: boolean             // default true; allows UI-only events
}
```

Back-compat: existing messages without these new fields load as-is; missing fields default safely.

#### Option B: Introduce `ConversationEvent[]` (More flexible, larger refactor)
Store typed events (`user_message`, `assistant_message`, `tool_result`, `mode_switch`, `llm_error`) and generate UI messages + LLM messages from that. This is more robust but touches more code.

MVP recommendation: **Option A**.

---

### 4.2 Context Builder (Core): `buildLlmMessagesFromConversation()`

Create a single builder used by Chat / Agent / Run.

**Inputs**
- `projectRoot`, `packageId`, `conversationId`
- `mode`: `'chat' | 'agent' | 'run'`
- `agentDefinition` + merged tool policy
- `runContext` when mode=`run` (workflowId/runId/current node info)
- `settings`: context budgets

**Outputs**
- `OpenAIChatMessage[]` to send to `llmAdapter.chatCompletions()`
- debug metadata: message counts, estimated tokens, which items were kept/dropped

**Budget model (Codex-style)**
- Hard limits:
  - max messages (e.g. 80)
  - max characters (fallback if token estimator is not used)
  - optionally a token budget estimate
- Always keep:
  - the latest user message
  - tool-call groups that are within the retained window
- When trimming:
  - trim from oldest → newest
  - preserve tool-call protocol groups (don’t keep `tool` without its preceding `assistant(tool_calls)` if it exists in-range)

**Tool output strategy (Chosen)**
- Persist full tool results for debugging and replay, but **do not** inject full payloads into every LLM request.
- Inject **preview + “read logs/path”** into context:
  - Include a short preview (e.g. first N chars / structured summary).
  - Include an **alias path** under `@state/` or `@project/` where the full output was saved (so the LLM can `fs.read` it on demand).
  - Example (conceptual):
    - `TOOL_RESULT_PREVIEW ...`
    - `Full output: @state/logs/tool-results/<toolCallId>.json`

Note: Conversation files under RuntimeStore are not directly readable by tools via `@state`; “read logs/path” refers to files we intentionally write under `@state`/`@project`.

**System prompt handling**
- System prompt is **not** persisted in history (avoid conflicts).
- On every call, prepend:
  - `system-base-rules-(chat|run).md`
  - `system-tool-policy.md`
  - persona (agent mode + run mode only; omit in chat mode)
  - a short “Mode banner” system message:
    - `MODE\n- active: run|agent|chat\n- note: history may include other modes; follow current instructions.`

**Run-only injection**
- For mode=`run`, prepend existing run directive messages (RUN_DIRECTIVE + NODE_BRIEF + step context) as they are deterministic and often large.
- The unified conversation log still stores the assistant/tool exchanges, errors, and user inputs.

---

### 4.3 Persistence Rules (What gets appended, and when)

Persist **both user-visible and internal** messages:

1) User input
   - Append `role=user` message (already done today).

2) LLM response
   - Append `role=assistant` message including `tool_calls` (even if `content=null`).
   - Append final assistant content message (if present).

3) Tool results
   - Append `role=tool` message for each tool result with `tool_call_id` (protocol-complete).
   - Store `toolName`, `duration`, and a UI-friendly collapsed preview.
   - For large tool outputs, write the full output to a dedicated file under `@state/logs/` (or `@project/artifacts/`) so the LLM can read it on demand; store only a preview + alias path in context.

4) Errors (critical for “continue”)
   - On LLM error/timeout/network:
     - Append `role=assistant` message that contains:
       - partial streamed text (if any)
       - a structured error block (`LLM_ERROR ...`)
     - This message must be included in context so retries can resume.

5) Mode switch
   - History is not cleared.
   - No explicit mode-switch events are required in the message log for MVP.
   - Persist the currently active mode via `ConversationMetadata.activeType` and have the context builder use the current mode’s system prefix.
   - Recommendation for correctness: tag messages with `runId` when produced during workflow execution so multiple runs in one conversation can be disambiguated without relying on mode-switch events.

---

## 5. Integration Points

### 5.1 Chat mode (`chat:send`)
- Load conversation messages for `conversationId`
- Use the unified context builder to assemble `messages[]`
- Execute via `chatToolLoop`
- Persist assistant/tool messages and errors into the same conversation log
- UI should show errors as assistant messages (not only `alert`)

### 5.2 Agent mode (`agent:dispatch` chat paths)
- Same as Chat mode, but `PromptComposer.compose(mode='agent')` includes persona.

### 5.3 Run mode (`ExecutionEngine`)
Unified approach (Chosen):
- Replace `session.history` as the source of truth with **conversation-backed history**.
- `ExecutionEngine` reads/writes the conversation log as the canonical history store (an in-memory cache is allowed for performance, but must be reconstructible from persisted messages on restart).

---

## 6. Backwards Compatibility / Migration

- Existing `messages.json` contain messages without `tool_calls/tool_call_id/role=tool`.
- Loading should treat missing fields as:
  - `content` = existing `content`
  - `includeInContext` = true
- New messages can be appended with extended schema without breaking older clients.

---

## 7. Settings (Proposed)

Add context controls (safe defaults, similar to Codex “sliding window”):

```ts
settings.llm.contextMaxMessages = 80
settings.llm.contextMaxChars = 120_000        // fallback
settings.llm.includeToolResultsInContext = true
settings.llm.includeCollapsedUiEventsInContext = false
```

MVP can start with `contextMaxMessages` only and iterate.

---

## 8. Test Plan

### Unit
- Context builder:
  - keeps latest user message
  - trims older messages correctly
  - preserves tool-call groups
  - mode banner and persona rules are correct (chat omits persona)
- Persistence:
  - tool results stored as `role=tool` with `tool_call_id`
  - LLM failures store `LLM_ERROR` + partial output and are included in subsequent request

### Integration
- Chat mode:
  - send → tool calls → final answer; reopen app; conversation still has tool messages; next message includes history.
- Mode switch:
  - chat → agent → run; history remains; system prompt changes; run still functions.
- Failure recovery:
  - simulate LLM timeout; verify a visible assistant error message exists and “continue” resumes from it.

---

## 9. Open Questions

1) Tool outputs in context: **Resolved** → preview + “read logs/path” (full output saved under `@state`/`@project`).
2) Automatic “conversation summary” message: optional v2. Meaning: when older messages are trimmed, insert a short summary message that represents the dropped segment so the LLM keeps continuity without full history.
3) Persist mode switches as explicit history events: **Resolved** → not required; store active mode in `ConversationMetadata.activeType` and use dynamic system prefix.
4) Unified conversation log as the only source for Run resumption: **Resolved** → yes.
