# CCV2-1.4 Active Tool-Loop Compression and Final Fallback

## Status

Completed

## Story

As CrewAgent Runtime, I want the current tool-call loop to be compressed only when historical compression is still not enough, so long-running tasks that generate hundreds of function calls in a single user turn can continue without breaking tool-call protocol or sending over-budget model requests.

## Problem

CCV2-1.1 to CCV2-1.3 solve historical context compression. They work when the context is large because old conversation history, old tool calls, old tool results, and old `reasoning_content` are taking space.

There is still an extreme case:

- A new conversation starts.
- The first user request triggers a long-running task.
- The model performs hundreds of function calls in the same user turn.
- Historical compression triggers, but there is little or no old history to compress.
- The current tool-call loop itself becomes the main source of context growth.

In this case, history compression can report an attempted no-op or remain above target. The system needs a second-stage compression path for the current tool loop, plus a clear final fallback when compression is still not enough.

## Goals

- Trigger current tool-loop compression only after historical compression still leaves the request near or above the context target.
- Compress completed older tool-call groups inside the current user turn without breaking OpenAI-compatible tool-call protocol.
- Preserve the current user request, recent complete tool-call groups, and any incomplete tool-call group.
- Remove old `reasoning_content` from compressed tool-loop groups.
- Add clear logs and metadata for historical compression, tool-loop compression, and final fallback.
- Prevent over-budget requests from being sent blindly after compression fails to reduce enough.

## Non-Goals

- Do not compress incomplete tool calls.
- Do not split an assistant `tool_calls` message from its matching tool results.
- Do not persist request-local `TOOL_LOOP_SUMMARY` into conversation storage unless a later story explicitly changes persistence behavior.
- Do not expose provider-specific reasoning controls to normal users.
- Do not replace CCV2 historical compression.

## User-Facing Behavior

When a single user request creates a very long tool-call chain, Runtime should behave like this:

1. Build the next LLM request.
2. If context is too large, compress old history first.
3. Recalculate context size.
4. If the request is still near or above the target, compress older completed tool calls from the current loop.
5. Recalculate context size again.
6. If still too large, run a more aggressive fallback or stop the loop with a clear message instead of sending an over-budget request.

The outgoing context should look like:

```text
system instructions
CONVERSATION_SUMMARY, if any
compressed older history

current user request

TOOL_LOOP_SUMMARY
- completed older tool calls summarized here
- important files, commands, results, errors, and do-not-repeat notes

recent complete tool-call groups
incomplete/current tool-call group, if any
```

## Acceptance Criteria

1. Historical compression remains the first compression stage.
2. Active tool-loop compression is triggered only when the context remains near or above the configured target after historical compression.
3. Tool-loop compression only compresses complete older tool-call groups from the current user turn.
4. Recent complete tool-call groups remain intact.
5. Incomplete tool-call groups remain intact.
6. A compressed tool-call group is replaced by a normal assistant message containing `TOOL_LOOP_SUMMARY`.
7. `TOOL_LOOP_SUMMARY` contains no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.
8. Compressed older tool groups do not carry `reasoning_content` into the outgoing request.
9. DeepSeek/Kimi recent reasoning policy remains respected for the preserved recent groups.
10. The final outgoing request passes tool-call protocol validation.
11. If historical compression plus tool-loop compression still cannot reduce enough, Runtime enters a final fallback path.
12. Final fallback must either produce a smaller protocol-valid context or stop the automatic loop with a clear `CONTEXT_STILL_TOO_LARGE` result.
13. Logs show historical compression result, tool-loop compression result, token before/after, number of summarized tool groups, and final fallback result.
14. Synthetic tests cover first-session first-turn tasks where a long function-call chain makes the active loop itself the main context pressure.

## Technical Design

### Trigger

Current tool-loop compression should not use a simple "number of tool calls" trigger.

The trigger is:

```text
historical compression was attempted or evaluated
AND request usage remains near or above target
AND there are complete older tool-call groups inside the current loop
```

Recommended internal threshold:

```text
usage after historical compression > targetUsage
```

The exact comparison should use the same token estimator and budget used by `CompressionPipeline`.

### Current Loop Boundary

Runtime already tracks a loop boundary so current tool calls are not accidentally folded into historical compression. CCV2-1.4 should reuse that boundary:

```text
messages before loopStartIndex -> historical section
messages from loopStartIndex onward -> current tool loop section
```

Tool-loop compression only operates on the current loop section.

### Tool Group Rules

The compressor must classify current-loop tool groups:

- `incomplete`: assistant has `tool_calls`, but matching tool results are not all present.
- `recent_complete`: latest complete groups to preserve verbatim.
- `older_complete`: completed groups eligible for summary.
- `invalid`: protocol-broken fragments; do not summarize aggressively unless repair logic already defines a safe outcome.

Only `older_complete` groups can be compressed.

### Summary Format

Compressed older groups become one or more normal assistant messages:

```text
TOOL_LOOP_SUMMARY
- scope: current user request
- tools summarized: <count>
- actions:
  - <toolName>: <important args/path/command/query> -> <result summary> (outcome: ok/error/unknown)
- files and paths: <paths touched or read>
- current findings: <facts learned>
- errors and blockers: <important failures>
- do not repeat: <calls that should not be repeated unless inputs changed or user asks>
```

The summary is ordinary assistant content. It must not contain tool protocol fields.

### Reasoning Content Policy

For tool groups that are summarized:

- Drop `reasoning_content`.
- Do not copy reasoning text into `TOOL_LOOP_SUMMARY`.
- Summarize only visible facts: tool name, important args, result preview, error, path, command, and outcome.

For recent groups that remain verbatim:

- Keep existing provider-aware reasoning behavior.
- DeepSeek/Kimi recent tool-call reasoning can remain if current policy says to preserve it.

### Final Fallback

After tool-loop compression, recalculate usage.

If usage is still above the safe target, Runtime must not blindly send the over-budget request.

Final fallback options:

1. More aggressive compression:
   - Keep fewer recent complete tool groups.
   - Shorten `TOOL_LOOP_SUMMARY`.
   - Keep only the current user request, summaries, and incomplete group.
2. Stop automatic loop:
   - Return a clear `CONTEXT_STILL_TOO_LARGE` error or controlled stop status.
   - Add a log explaining that the current task generated too much tool context.
   - Keep summaries available in the request context so the next user turn can continue.

The first implementation should prefer a deterministic aggressive fallback before stopping.

### Metadata

Extend compression metadata with active-loop fields:

- `activeLoopCompressionAttempted`
- `activeLoopCompressionEffective`
- `activeLoopSummarizedToolGroupCount`
- `activeLoopSummaryTokens`
- `activeLoopPreservedRecentToolGroupCount`
- `activeLoopDroppedMessageIds`
- `finalFallbackUsed`
- `finalFallbackReason`

### Logs

Logs should clearly distinguish:

```text
Context compression effective ...
Active tool-loop compression effective ...
Context still too large after compression; using final fallback ...
Context still too large after final fallback; stopping tool loop ...
```

## Implementation Plan

1. Add synthetic fixtures for first-session first-turn long tool loops.
2. Add tests showing historical compression alone cannot reduce the current-loop-only case.
3. Add active tool-loop group classification based on current loop boundary.
4. Add deterministic `TOOL_LOOP_SUMMARY` builder.
5. Add active tool-loop compression step after historical compression in Chat tool loop.
6. Recalculate usage after active-loop compression.
7. Add final fallback behavior for still-over-target contexts.
8. Extend compression metadata and telemetry formatter.
9. Add Run/Workflow integration after Chat tool-loop behavior is stable.
10. Run protocol validator after active-loop compression and final fallback.

## Verification Plan

- Unit test: complete current-loop tool groups are summarized only when historical compression remains above target.
- Unit test: incomplete tool group is never summarized.
- Unit test: recent complete tool groups remain intact.
- Unit test: summarized groups contain no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.
- Unit test: DeepSeek/Kimi recent preserved groups keep expected reasoning fields.
- Integration test: first conversation, first user request, many complete tool groups compress below target or enter controlled final fallback.
- Integration test: outgoing messages pass protocol validation after active-loop compression.
- Integration test: logs include active-loop compression metadata.
- Regression test: normal short tool loops do not trigger active-loop compression.

## Implementation Summary

Implemented in `crewagent-runtime`:

- Added active tool-loop compression after historical compression in `CompressionPipeline`.
- Added deterministic `TOOL_LOOP_SUMMARY` generation for older completed tool-call groups in the current user turn.
- Preserved recent complete tool-call groups verbatim during the normal active-loop compression pass.
- Kept summarized active-loop groups out of the outgoing tool protocol by replacing them with ordinary assistant content.
- Ensured summarized active-loop messages contain no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.
- Added an aggressive final fallback pass that can preserve zero recent complete groups when the normal pass is still above target.
- Added `context_still_too_large` metadata and Chat tool-loop handling that stops before the LLM request with `CONTEXT_STILL_TOO_LARGE`.
- Extended compression metadata and telemetry details with active-loop and final-fallback fields.
- Exported `TOOL_LOOP_SUMMARY` helpers for tests and future reuse.

## Verification Results

- Passed: `npm test -- --run electron/services/chatToolLoop.test.ts`
  - 28 tests passed after Chat MCP forced-retry final-request fallback fix.
- Passed: `npm test -- --run electron/core/context-compression/context-compression.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`
  - 149 tests passed after Chat MCP forced-retry final-request fallback fix.
- Passed: `npm run build:ci`
  - TypeScript and Vite production builds completed.
  - Existing warnings observed: npm unknown user config warnings, `jspreadsheet-ce` eval warning, Vite large chunk warnings, and one `mcpInstallService` mixed dynamic/static import warning.
- Passed: `git diff --check` in `crewagent-runtime`.
- Passed: `git diff --check` in repo root.
- Passed: `git diff --check` in `_bmad-output`.

## Acceptance Mapping

1. Historical compression remains first because active-loop compression is only called from `compressHistoricalMessages` after the historical stage returns.
2. Active-loop compression is gated by post-history usage remaining above the configured target while the full request is already over the compression trigger.
3. Only complete older current-loop tool groups are summarized.
4. Recent complete tool groups remain intact in the normal pass.
5. Incomplete or invalid groups are not summarized by the active-loop compressor.
6. Summarized groups are replaced by a normal assistant `TOOL_LOOP_SUMMARY` message.
7. `TOOL_LOOP_SUMMARY` carries no tool protocol fields and no reasoning payload.
8. Summarized older active-loop groups drop their old `reasoning_content`.
9. DeepSeek/Kimi recent reasoning policy remains attached to preserved recent groups because those messages stay verbatim.
10. Final outgoing messages run through protocol validation and repair fallback.
11. When the normal pass remains above target, an aggressive final fallback pass is attempted.
12. If the request remains above the compression trigger after fallback, Chat and Run/Workflow request paths stop with `CONTEXT_STILL_TOO_LARGE`.
13. Compression telemetry now includes active-loop and final-fallback details.
14. Synthetic tests cover first-turn long active tool loops and controlled stop behavior when compression cannot reduce enough.

## Code Review Notes

- Review fix applied: inactive active-loop summary attempts no longer report summarized group counts, summary token counts, or dropped message ids. Metadata now counts only the summary pass that was actually adopted into the outgoing context.
- Review fix applied: Run/Workflow mode now stops before the LLM request when final fallback reports `context_still_too_large`, matching the Chat path behavior.
- Review fix applied: Chat and Run/Workflow modes now emit `compressing` usage status before awaiting async compression, so the UI reflects long-running compression while it is happening.
- Review fix applied: Run/Workflow mode now checks the fully assembled final request after protocol repair and stops with `CONTEXT_STILL_TOO_LARGE` if system prompts or step context push it back over the compression trigger.
- Review fix applied: Chat skill activation now keeps the current-loop boundary aligned when injected system messages change the leading system prefix before the next compression pass.
- Review fix applied: recent complete tool-call group preservation now counts only complete, non-current-loop groups, so trailing incomplete or invalid groups no longer consume the recent-group preservation budget.
- Review fix applied: Chat mode now checks the fully assembled final request after compression and protocol repair, stopping with `CONTEXT_STILL_TOO_LARGE` if preserved system or skill prefix context keeps the request above the compression trigger.
- Review fix applied: Chat MCP forced retry now reuses the final request budget guard before the retry LLM call, restoring the original assistant message and stopping with `CONTEXT_STILL_TOO_LARGE` if the forced retry prompt would exceed the context trigger.

## Development Notes

This story should be implemented after CCV2-1.3. It should reuse existing protocol helpers where possible instead of creating a separate tool protocol model.

The most important rule is:

```text
Never delete half of a tool-call protocol group.
```

An assistant tool-call message and its matching tool results must either remain together or be replaced together by an ordinary assistant summary.
