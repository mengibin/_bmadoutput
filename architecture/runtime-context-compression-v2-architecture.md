---
stepsCompleted: [1, 2, 3, 4, 5]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-context-compression-v2.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/unified-conversation-context.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/7-4-create-context-compression-module.md'
workflowType: 'architecture'
lastStep: 5
project_name: 'CrewAgent Runtime Context Compression V2'
user_name: 'Mengbin'
date: '2026-05-14'
---

# Runtime Context Compression V2 Architecture

## Implementation Status

- Status: `done`
- Completed scope: `CCV2-1.1 ~ CCV2-1.4`
- Closure artifact: `_bmad-output/implementation-artifacts/epic-ccv2-retrospective.md`
- Plan context decision: no separate `CCV2-1.5` is needed now; Plan context remains a follow-up candidate only if a single current plan or remarks file becomes large enough to require its own budget guard.

## 1. Summary

Context Compression V2 upgrades the current heuristic-only context compression into a protocol-aware, provider-aware, summary-capable pipeline.

The design keeps the existing module boundary under `electron/core/context-compression/`, but changes the pipeline contract from “return selected messages” to “return a compression attempt result with accurate effectiveness metadata”. The new pipeline first performs deterministic structural compression, then optionally calls an LLM to produce a rolling `CONVERSATION_SUMMARY`, then validates the final outgoing message protocol before handing it back to Chat, Agent, or Run mode.

After CCV2-1.3, a remaining edge case exists: the first user turn of a new session can produce hundreds of function calls before there is any historical context to compress. CCV2-1.4 adds a second-stage active tool-loop compression path. It runs only when historical compression is still not enough and summarizes older completed tool-call groups from the current loop into protocol-safe `TOOL_LOOP_SUMMARY` messages.

## 2. Current Architecture

Current path:

```text
Chat / Agent / Run
  -> CompressionPipeline
  -> ContextUsageMonitor
  -> KeyMessageExtractionStrategy
  -> OpenAIChatMessage[]
```

Current characteristics:

- Threshold-based trigger at usage > 80%.
- Target after compression around 50%.
- Preserves all user messages.
- Preserves complete tool-call groups.
- Supports basic truncation and oversized tool-call summary only in narrow fallback cases.
- Reports `compressed: true` when compression is triggered, not when reduction is effective.

## 3. Design Goals

- Preserve OpenAI-compatible tool protocol.
- Avoid provider-specific request failures for DeepSeek/Kimi thinking models.
- Reduce old tool-heavy history aggressively but safely.
- Use LLM summaries for semantic continuity.
- Keep compression deterministic and safe if summary LLM fails.
- Reuse one core pipeline across Chat, Agent, and Run.

## 4. Proposed Module Layout

```text
electron/core/context-compression/
├── index.ts
├── types.ts
├── tokenEstimator.ts
├── ContextUsageMonitor.ts
├── CompressionPipeline.ts
├── protocol/
│   ├── toolCallGroupClassifier.ts
│   ├── protocolValidator.ts
│   └── providerReasoningPolicy.ts
├── summaries/
│   ├── deterministicSummary.ts
│   ├── conversationSummaryPrompt.ts
│   ├── conversationSummaryParser.ts
│   ├── ConversationSummaryService.ts
│   └── toolLoopSummary.ts
├── strategies/
│   ├── CompressionStrategy.ts
│   ├── ProtocolAwareCompressionStrategy.ts
│   └── KeyMessageExtractionStrategy.ts
└── context-compression.test.ts
```

`KeyMessageExtractionStrategy` can remain for compatibility and fallback, but new default behavior should be `ProtocolAwareCompressionStrategy`.

## 5. Core Types

### 5.1 CompressionAttemptResult

```ts
export type CompressionStatus = 'not_needed' | 'attempted_effective' | 'attempted_noop' | 'failed_fallback'

export interface CompressionAttemptResult {
  messages: ConversationMessage[]
  status: CompressionStatus
  attempted: boolean
  effective: boolean
  beforeUsage: ContextUsage
  afterUsage: ContextUsage
  metadata: CompressionMetadata
}

export interface CompressionMetadata {
  inputCount: number
  outputCount: number
  droppedCount: number
  inputTokens: number
  outputTokens: number
  targetTokens: number
  targetExceeded: boolean
  strategyUsed: string
  droppedMessageIds: string[]
  summarizedToolGroupCount: number
  strippedReasoningCount: number
  deterministicSummaryCount: number
  llmSummaryUsed: boolean
  summaryTokens: number
  reason?: string
}
```

### 5.2 ToolCallGroup

```ts
export interface ToolCallGroup {
  id: string
  assistantIndex: number
  toolIndexes: number[]
  toolCallIds: string[]
  complete: boolean
  hasReasoning: boolean
  hasLargeArgs: boolean
  hasLargeResults: boolean
  totalTokens: number
  classification: 'current_loop' | 'recent' | 'historical' | 'invalid'
}
```

### 5.3 ProviderReasoningPolicy

```ts
export interface ProviderReasoningPolicy {
  provider: 'deepseek' | 'kimi' | 'openai-compatible' | 'unknown'
  preserveRecentToolReasoning: boolean
  preserveAllHistoricalReasoning: boolean
  stripNonToolHistoricalReasoning: boolean
  oldToolGroupAction: 'preserve' | 'summarize'
  reason: string
}
```

The policy should be derived from runtime LLM settings, model name, provider base URL, and explicit thinking options when available.

## 6. Compression Pipeline

### 6.1 High-Level Flow

```text
build request messages
  -> calculate usage
  -> if under threshold, return not_needed
  -> classify tool-call groups
  -> define preservation window
  -> strip safe historical reasoning
  -> summarize old tool-call groups
  -> merge or create CONVERSATION_SUMMARY
  -> if still over target, run LLM summary
  -> if LLM summary fails, deterministic fallback
  -> if still over target, summarize older completed active-loop tool groups
  -> if still over target, use final fallback or stop with CONTEXT_STILL_TOO_LARGE
  -> validate tool protocol
  -> if invalid, conservative fallback
  -> return attempt result
```

### 6.2 Preservation Window

Preserve fully:

- System prompt prefix built for the current request.
- Current loop messages.
- Latest user input.
- Recent N conversation rounds.
- Recent M tool-call groups.
- Existing `SUMMARY` / `CONVERSATION_SUMMARY` messages.
- Messages carrying explicit important markers such as `ARTIFACT_SAVED` and `NODE_COMPLETE`.

Default suggested values:

```ts
recentRounds = 4
recentToolGroups = 6
minRecentMessages = 10
targetUsage = 0.5
triggerThreshold = 0.8
summaryTargetTokens = 2000
summaryTimeoutMs = 15000
```

### 6.3 Tool-Call Group Classification

The classifier scans messages and creates groups:

```text
assistant with tool_calls
  -> collect following tool messages whose tool_call_id matches
  -> stop at next assistant/user/system
  -> mark complete if all tool_call_ids have results
```

Invalid groups must not be aggressively rewritten until validator behavior is defined. The safe default is to keep invalid current groups and drop invalid historical tool messages only if they are already protocol-broken and not needed.

### 6.4 Reasoning Handling

Rules:

- Current loop tool-call reasoning stays.
- Recent tool-call reasoning stays for DeepSeek/Kimi safety.
- Non-tool historical assistant reasoning may be stripped.
- Historical tool-call group reasoning is removed only when the whole group is converted to a normal summary message.
- Kimi `thinking.keep: "all"` should be treated as a costful preservation mode. If compression pressure is critical, V2 may summarize old reasoning and log a policy downgrade.

### 6.5 Historical Tool-Call Summary

An old complete group:

```text
assistant(content, reasoning_content, tool_calls)
tool(tool_call_id, content)
tool(tool_call_id, content)
```

becomes:

```text
assistant(content = "TOOL_CALL_HISTORY_SUMMARY\n...")
```

The replacement assistant message must not contain:

- `tool_calls`
- `reasoning_content`
- `tool_call_id`

The summary format:

```text
TOOL_CALL_HISTORY_SUMMARY
- timeframe: <createdAt range>
- purpose: <short purpose inferred from assistant content or args>
- tools:
  - <toolName>: <args summary> -> <result summary>
- outcome: <what changed or was learned>
- do_not_repeat: <optional>
```

The deterministic summary should be conservative and factual, based on tool names, safe argument previews, file paths, and result previews.

### 6.6 LLM Rolling Summary

LLM summary is an optional second pass.

Inputs:

- Existing `CONVERSATION_SUMMARY`, if any.
- Deterministic summaries of removed tool groups.
- Dropped user/assistant messages that are outside the preservation window.
- Current mode metadata.
- Current plan/run state summary when available.

### 6.7 Active Tool-Loop Compression

Historical compression must remain the first stage. Active tool-loop compression is only evaluated after historical compression has completed or been considered and the request is still near or above the target budget.

The active loop is the current user request and the tool-call chain generated after it. The boundary is already tracked by `loopStartIndex` in current runtime paths.

```text
all request messages
  -> historical section before loopStartIndex
  -> current loop section from loopStartIndex onward
```

Active tool-loop compression only operates on the current loop section.

Rules:

- Preserve the current user request.
- Preserve incomplete tool-call groups exactly.
- Preserve the latest complete tool-call groups exactly.
- Summarize only older complete tool-call groups.
- Replace summarized groups with ordinary assistant messages.
- The summary message must not include `tool_calls`, `tool_call_id`, or `reasoning_content`.

Summary format:

```text
TOOL_LOOP_SUMMARY
- scope: current user request
- tools summarized: <count>
- actions:
  - <toolName>: <path/query/command/args preview> -> <result preview> (outcome: ok/error/unknown)
- files and paths: <paths>
- current findings: <facts learned>
- errors and blockers: <important failures>
- do not repeat: <calls not to repeat unless inputs changed>
```

The summary is intentionally an assistant text message, not a tool result. This avoids creating orphan `tool` messages or assistant `tool_calls` without matching tool results.

### 6.8 Final Fallback

After active tool-loop compression, usage is recalculated.

If the request still exceeds the safe target, Runtime should not send the request blindly. It must either:

1. Apply a more aggressive deterministic fallback while preserving protocol validity.
2. Stop the automatic loop with a clear `CONTEXT_STILL_TOO_LARGE` status and log.

The first implementation should try deterministic fallback before stopping. A stop is preferable to a provider-level context-window failure because it gives the UI and logs an explicit recovery point.

### 6.9 Reasoning Handling in Active Tool-Loop Compression

For summarized active-loop groups:

- Drop `reasoning_content`.
- Do not copy reasoning text into `TOOL_LOOP_SUMMARY`.
- Keep only visible execution facts: tool name, important arguments, file path, command, result preview, errors, and outcome.

For preserved recent active-loop groups:

- Continue to use provider-aware reasoning policy.
- DeepSeek/Kimi recent tool-call reasoning can remain if current policy requires it.

This keeps active-loop compression aligned with the historical compression rules from CCV2-1.2.

Output:

```text
CONVERSATION_SUMMARY
- User goal:
- Current status:
- Confirmed constraints:
- Key decisions:
- Files and paths:
- Tool results:
- Completed actions:
- Pending tasks:
- Risks and blockers:
- Do not repeat:
```

The summary should be a normal system summary message or a summary-marked conversation message that `KeyMessageExtractionStrategy` already knows how to preserve.

### 6.7 Summary LLM Call Isolation

The summary call must:

- Use a small bounded input.
- Have a timeout.
- Avoid tools.
- Avoid streaming unless already supported cleanly.
- Not mutate conversation history unless persistence is explicitly implemented.
- Return failure without blocking main request.

## 7. Protocol Validator

Validator checks outgoing messages after compression:

1. For each assistant `tool_calls`, collect ids and require matching following `tool` messages.
2. For each `tool` message, require an open expected `tool_call_id`.
3. Reject orphan tool messages.
4. Reject duplicate tool results for the same id unless provider supports it.
5. Reject assistant summary messages that still contain `tool_calls`.
6. Allow current in-flight assistant tool calls only before tool execution, not before sending a user turn.

Failure handling:

- First fallback: restore recent complete groups and revalidate.
- Second fallback: no LLM summary, deterministic only.
- Final fallback: keep original messages and mark `failed_fallback`.

## 8. Integration Points

### 8.1 Chat

`chatContextBuilder` should call V2 pipeline before system message injection where appropriate, and `chatToolLoop` should apply V2 historical compression inside multi-turn tool loops.

Care is required because `chatToolLoop` currently preserves system prefix and compresses non-system messages. V2 should keep this behavior but report usage for full outgoing context and non-system compressed region separately.

### 8.2 Agent

Agent chat uses the same builder pattern as chat and should pass agent identity and tool policy metadata into the compression context.

### 8.3 Run / Workflow

`ExecutionEngine` should use V2 compression for persisted run history before composing run context. Current step and node directive must be treated as system/run prefix and preserved.

### 8.4 Plan Mode

Plan mode adds `PLAN_CONTEXT`, authoring rules, and execution state messages. These are not historical chat messages and should be budgeted separately.

Recommended V2 behavior:

- Preserve current plan state injection.
- Compress historical conversation using V2.
- Emit separate metadata:
  - `historyBefore/historyAfter`
  - `systemPrefixTokens`
  - `planContextTokens`

## 9. Logging and Telemetry

### Effective Compression Log

```text
Context compression effective
- before: 297745/100000 tokens, messages=634
- after: 48200/100000 tokens, messages=72
- target: 50000 tokens
- summarizedToolGroups: 240
- strippedReasoning: 260
- llmSummary: true
```

### No-Op Log

```text
Context compression attempted but no effective reduction
- before: 98500/100000 tokens
- after: 98480/100000 tokens
- reason: protocol-critical history exceeded target
```

### Failed Fallback Log

```text
Context compression failed; using conservative fallback
- reason: protocol validation failed after summary
- validatorError: orphan tool message
```

## 10. Testing Strategy

### Unit Tests

- `toolCallGroupClassifier` groups complete and incomplete sequences.
- `protocolValidator` rejects orphan tool messages.
- `providerReasoningPolicy` derives DeepSeek/Kimi behavior.
- deterministic summary removes `tool_calls`, `tool`, and `reasoning_content` for old groups.
- metadata reports no-op correctly.

### Integration Tests

- Chat path compresses tool-heavy history effectively.
- Agent path compresses tool-heavy history effectively.
- Run path compresses tool-heavy history effectively.
- Plan mode reports history compression separately from plan context injection.
- LLM summary failure falls back without failing the main request.

### Fixture Tests

Synthetic fixture:

- 300 assistant tool-call messages.
- 300 matching tool results.
- 90 percent of assistant tool-call messages include `reasoning_content`.
- Large tool args and large tool results.
- Expected output drops below target or records targetExceeded with clear reason.

## 11. Migration Plan

1. Add V2 types while keeping old `CompressionResult` compatibility.
2. Introduce V2 strategy behind a feature flag or internal config.
3. Update logs to use attempted/effective semantics first.
4. Add classifier and validator.
5. Add deterministic structural compression.
6. Add provider-aware reasoning policy.
7. Add LLM summary service.
8. Switch Chat/Agent/Run to V2 default after tests pass.

## 12. Risks and Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | Breaking tool-call protocol | Add validator before every outgoing request |
| R2 | DeepSeek rejects missing reasoning | Preserve recent tool-call reasoning and summarize only whole old groups |
| R3 | Kimi preserved thinking becomes ineffective | Log policy downgrade and make `thinking.keep` behavior explicit |
| R4 | LLM summary loses important details | Use fixed schema, deterministic facts, and recent window preservation |
| R5 | Summary LLM call adds latency | Timeout, bounded input, deterministic fallback |
| R6 | Plan context still dominates context | Report plan context tokens separately and handle as a later plan-context optimization |

## 13. Source Notes

- DeepSeek thinking mode requires tool-call turn reasoning to be passed back in subsequent requests: https://api-docs.deepseek.com/guides/thinking_mode
- Kimi `thinking.keep` controls whether historical reasoning is preserved; preserved reasoning consumes context: https://platform.kimi.ai/docs/guide/use-kimi-k2-thinking-model
