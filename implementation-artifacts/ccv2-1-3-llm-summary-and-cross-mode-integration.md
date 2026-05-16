# CCV2-1.3 LLM Summary and Cross-Mode Integration

## Status

Completed

## Story

As CrewAgent Runtime, I want old compressed history to be represented by a semantic rolling summary across Chat, Agent, Plan Chat, and Run/Workflow, so long conversations keep their intent, constraints, files, decisions, and pending work even after structural compression removes or condenses historical messages.

## Scope

- Add a summary service under `electron/core/context-compression/summaries/`.
- Parse and validate `CONVERSATION_SUMMARY` content with required headings.
- Generate deterministic fallback summaries when LLM summary is unavailable, malformed, or fails.
- Merge existing `CONVERSATION_SUMMARY` with newly summarized historical material instead of duplicating summary messages.
- Insert the summary as a request-local system summary message before compressed history.
- Surface summary metadata in compression results and logs.
- Keep current synchronous pipeline users compatible; LLM summary is optional and failure-safe.

Out of scope:

- Persisting generated summaries back into conversation history.
- Adding a separate summary model selector in settings.
- Replacing structural compression with LLM-only compression.
- Changing plan context injection semantics beyond preserving history-compression metadata.

## Acceptance Criteria

1. Existing `CONVERSATION_SUMMARY` messages are detected and merged into a single summary message.
2. Valid generated summaries must start with `CONVERSATION_SUMMARY` and include required headings.
3. Malformed LLM summary output is rejected and deterministic fallback is used.
4. Summary generation failure or timeout must not block the main request.
5. When structural compression drops or summarizes historical messages, compressed output includes one summary message when summary material exists.
6. Summary metadata records whether LLM summary was used, whether deterministic fallback was used, summary token count, and summary failure reason when applicable.
7. Chat, Agent, Chat tool-loop, and Run/Workflow continue using the same V2 pipeline metadata and remain protocol-valid.

## Technical Design

### Summary Modules

- `conversationSummaryParser.ts`
  - `isConversationSummaryMessage`
  - `parseConversationSummary`
  - `validateConversationSummary`
- `deterministicSummary.ts`
  - Builds a bounded factual fallback from existing summary, dropped messages, deterministic tool summaries, user goals, paths, and visible assistant conclusions.
- `ConversationSummaryService.ts`
  - Accepts optional async `summarizer`.
  - Uses timeout guard.
  - Validates LLM output before use.
  - Falls back deterministically on unavailable, timeout, exception, or malformed output.

### Pipeline Integration

The current sync `compressIfNeeded` remains deterministic and non-blocking. Summary insertion happens synchronously through deterministic fallback. The service type supports optional async LLM summarizer so a later caller can opt into actual LLM generation without changing parser/fallback/metadata contracts.

The summary pass runs after structural compression and before final protocol validation:

1. Identify existing summary messages.
2. Identify dropped messages and deterministic tool summaries.
3. Build or merge one `CONVERSATION_SUMMARY`.
4. Remove duplicate prior summary messages from the outgoing history.
5. Recompute usage and effective metadata.

### Metadata

Extend compression metadata:

- `llmSummaryUsed`
- `deterministicSummaryUsed`
- `summaryTokens`
- `summaryFailureReason`
- `summaryMessageId`

## Implementation Plan

1. Add summary parser/validator tests.
2. Add deterministic fallback summary builder and tests.
3. Add summary service with malformed/failure fallback tests.
4. Extend compression types and telemetry formatter.
5. Integrate deterministic summary pass into `CompressionPipeline`.
6. Add integration tests for merged summary, protocol validity, and metadata.
7. Run CCV2 test suite and build.

## Verification Plan

- `npm test -- --run electron/core/context-compression/context-compression.test.ts`
- `npm test -- --run electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`
- `npm run build:ci`
- `git diff --check`

## Implementation Summary

- Added `summaries/` parser, deterministic fallback builder, LLM-ready summary prompt, and timeout/fallback summary service.
- Integrated `CONVERSATION_SUMMARY` insertion into `CompressionPipeline` after structural compression and before protocol validation, with sync deterministic fallback and async LLM-summary support.
- Merged duplicate existing summary messages into one request-local system summary.
- Added summary metadata to compression results and telemetry logs.
- Preserved synchronous compression callers; Chat tool-loop and Run/Workflow compression now use the async summary path when a summarizer is configured.

## Verification Results

- `npm test -- --run electron/core/context-compression/context-compression.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`: 117 passed.
- `npm test -- --run electron/core/context-compression/context-compression.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`: 138 passed after review fixes.
- `npm run build:ci`: passed; existing warnings remain for npm user config, `jspreadsheet-ce` eval, large chunks, and `mcpInstallService` mixed import.
- `git diff --check` in `crewagent-runtime`: passed.
- `git diff --check` in repo root: passed.

## Code Review Notes

- No blocking issues found in the CCV2-1.3 implementation review.
- Review fixes applied: no-op/protocol fallback results no longer report an unused generated summary, and only system messages are treated as `CONVERSATION_SUMMARY` messages.
- Review fixes applied after follow-up: dropped `reasoning_content` is no longer copied into summaries, async compression can call configured LLM summarizers, malformed LLM summaries fall back deterministically, Chat tool-loop uses async compression before the LLM request, and Run/Workflow compression uses async summary generation.
- Review fixes applied after second follow-up: Chat tool-loop no longer treats leading `CONVERSATION_SUMMARY` as static system prefix, skill activation preserves generated request-local summaries, and LLM summary prompt construction now bounds dropped-history input with an omitted-count marker.
