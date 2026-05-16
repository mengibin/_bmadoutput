# CCV2-1.2 Protocol-Aware Structural Compression

## Status

Completed

## Story

As CrewAgent Runtime, I want context compression to be protocol-aware and provider-aware, so old tool-heavy history and historical reasoning content can be reduced without breaking OpenAI-compatible tool-call ordering or provider thinking-mode requirements.

## Scope

- Add provider reasoning policy resolution for DeepSeek, Kimi/Moonshot, OpenAI-compatible, and unknown providers.
- Strip safe historical `reasoning_content` from non-tool assistant messages.
- Convert old complete tool-call groups outside the recent preservation window into ordinary `TOOL_CALL_HISTORY_SUMMARY` assistant messages.
- Preserve current-loop and recent tool-call groups, including `reasoning_content`, when provider policy requires it.
- Surface deterministic compression metadata for summarized tool groups, stripped reasoning, and policy decisions.
- Reuse the existing shared compression pipeline across Chat/Agent/Run.

Out of scope:

- LLM rolling `CONVERSATION_SUMMARY`.
- Cross-turn summary persistence.
- Any change to model response generation behavior outside context preparation.

## Acceptance Criteria

1. DeepSeek thinking mode recent tool-call groups preserve `reasoning_content`.
2. Historical non-tool assistant messages with `reasoning_content` can have reasoning stripped while visible content remains.
3. Old complete tool-call groups can be summarized, and the original tool-call assistant `reasoning_content` is removed because the original protocol group is no longer sent.
4. Kimi/Moonshot keep-all reasoning mode records a clear policy decision when compression pressure would otherwise require reasoning reduction.
5. Hundreds of old complete tool-call groups outside the recent preservation window are converted to ordinary summary messages or otherwise reduced safely.
6. Summarized tool groups emit no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.
7. Large old tool results produce deterministic summaries containing tool name, argument/path preview when available, result preview, and outcome/status.
8. Structural compression either significantly decreases estimated tokens or returns metadata explaining why the target was not reached.
9. Protocol validation passes after structural compression.

## Technical Design

### Provider Reasoning Policy

Add `electron/core/context-compression/protocol/providerReasoningPolicy.ts`.

Policy input:

- `provider`
- `baseUrl`
- `model`
- `requestOptions.thinking`

Policy output:

- provider family
- whether recent tool-call reasoning must be preserved
- whether historical reasoning may be stripped
- whether old complete tool-call groups may be summarized
- a machine-readable policy id and human-readable reason

Default policy is conservative for live/recent protocol groups and aggressive for historical non-tool reasoning. Kimi/Moonshot with ordinary user-facing `thinking: "enabled"` is not treated as keep-all; historical standalone reasoning remains eligible for stripping. Explicit keep-all is supported only through internal policy input such as `{ keep: "all" }`, and the policy reason must be visible in metadata/log details when used.

### Structural Compression

Extend `KeyMessageExtractionStrategy` instead of adding a second strategy class for this Story. This keeps the Story scoped to the current default pipeline while using the classifier and validator introduced in CCV2-1.1.

Preprocess before priority selection:

1. Classify tool-call groups.
2. Summarize historical complete groups outside `recentToolGroupCount`.
3. Strip `reasoning_content` from historical non-tool assistant messages if policy allows it.
4. Preserve current/recent groups intact.
5. Preserve invalid groups conservatively.

The summary replacement is a normal assistant message:

```text
TOOL_CALL_HISTORY_SUMMARY
- purpose: ...
- tools:
  - toolName: args/path preview -> result preview
- outcome: ok|error|unknown
```

It must not include `tool_calls`, `tool_call_id`, or `reasoning_content`.

### Metadata

Extend compression metadata with optional fields:

- `summarizedToolGroupCount`
- `strippedReasoningCount`
- `deterministicSummaryCount`
- `reasoningPolicy`
- `reasoningPolicyReason`

## Implementation Plan

1. Add provider reasoning policy module and exports.
2. Extend compression types/config to carry provider, model, baseUrl, requestOptions, and structural-compression metadata.
3. Extend `CompressionStrategy` with optional stats reporting.
4. Add structural preprocessing to `KeyMessageExtractionStrategy`.
5. Wire policy options through `CompressionPipeline`.
6. Pass current LLM settings into Chat/Agent/Run compression pipeline construction.
7. Add focused unit tests for policy resolution, reasoning stripping, summary output shape, DeepSeek recent preservation, Kimi policy metadata, large old tool result summaries, 300-group structural compression, and protocol validation.
8. Run the context compression test suite.

## Verification Plan

- `npm test -- --run electron/core/context-compression/context-compression.test.ts`
- Confirm existing CCV2-1.1 protocol tests still pass.
- Confirm new CCV2-1.2 AC tests fail before implementation and pass after implementation.

## Implementation Summary

Implemented in `crewagent-runtime`:

- Added `protocol/providerReasoningPolicy.ts` with DeepSeek, Kimi/Moonshot, OpenAI-compatible, and unknown-provider policy resolution.
- Extended compression config and metadata with provider/model/requestOptions, `recentToolGroupCount`, policy id/reason, summarized tool group count, stripped reasoning count, and deterministic summary count.
- Extended `KeyMessageExtractionStrategy` with a structural preprocessing pass:
  - complete historical tool-call groups outside the recent window become ordinary `TOOL_CALL_HISTORY_SUMMARY` assistant messages;
  - summarized messages remove `tool_calls`, `tool_call_id`, and `reasoning_content`;
  - historical non-tool assistant reasoning is stripped when policy allows it;
  - recent/current/invalid tool-call groups remain conservative and protocol-safe.
- Wired provider reasoning policy through `CompressionPipeline`.
- Passed current LLM provider, base URL, model, and request options into Chat/Agent/Run compression pipeline construction.
- Added focused tests for provider policy, DeepSeek recent reasoning preservation, default Kimi thinking compression, internal Kimi keep-all metadata and fallback preservation, historical reasoning stripping, deterministic tool group summaries, 300-group tool-heavy reduction, and protocol validation.

## Verification Results

- Passed: `npm test -- --run electron/core/context-compression/context-compression.test.ts`
  - 50 tests passed.
- Passed: `npm test -- --run electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`
  - 66 tests passed.
- Passed: `npm run build:ci`
  - TypeScript and Vite production builds completed.
  - Existing Vite warnings observed: `jspreadsheet-ce` eval warning, large chunk warnings, and one dynamic/static import warning for `mcpInstallService.ts`.

## Review Fixes

- Fixed Kimi policy semantics: ordinary `thinking: "enabled"` no longer maps to keep-all reasoning. Historical non-tool assistant `reasoning_content` remains eligible for compression in default Kimi thinking mode.
- Kept explicit internal `{ keep: "all" }` policy support out of ordinary user-facing settings and routed it through `CompressionConfig.reasoningPolicyInput`.
- Propagated compression metadata through Chat/Agent context builders so page logs can include policy id, structural summary counts, stripped reasoning counts, and compression reason.
- Added `formatCompressionLogDetails` for shared log detail formatting across Chat/Agent/Run paths.
- Moved deterministic summary group outcome into the header so truncation cannot remove it from long multi-tool summaries.
- Enforced explicit keep-all reasoning preservation through both priority selection and fallback trimming, so old standalone assistant reasoning is not silently dropped when target pressure remains.
- Fixed Run-mode compression to pass a stable current-loop boundary into historical compression, preserving tool-call groups produced during the active run loop instead of summarizing them as old history.
- Fixed Chat tool-loop compression logging to log only real compression attempts and include shared policy/count metadata details.

Additional verification after review fixes:

- Passed: `npm test -- --run electron/core/context-compression/context-compression.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts`
  - 124 tests passed.
- Passed: `npm run build:ci`
  - TypeScript and Vite production builds completed.
  - Existing Vite warnings observed: `jspreadsheet-ce` eval warning, large chunk warnings, and one dynamic/static import warning for `mcpInstallService.ts`.

## Acceptance Mapping

1. DeepSeek recent tool-call reasoning preservation covered by `preserves DeepSeek recent tool-call reasoning and protocol group`.
2. Historical non-tool reasoning stripping covered by `strips historical non-tool reasoning while keeping visible assistant content`.
3. Old complete tool-call group reasoning removal covered by `summarizes old complete tool-call groups as ordinary assistant messages`.
4. Kimi keep-all policy visibility and historical reasoning preservation covered by `supports internal Pipeline keep-all reasoning policy and preserves historical reasoning through fallback`.
5. Hundreds of old tool groups covered by `reduces hundreds of historical tool-call groups without violating protocol`.
6. Summary message protocol shape covered by assertions for no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.
7. Large tool result deterministic summary covered by `builds deterministic summaries with tool name, args/path preview, result preview, and outcome` and `keeps deterministic summary outcome visible for long multi-tool groups`.
8. Reduction/target metadata covered by compression metadata and `target_exceeded_after_structural_compression` reason.
9. Protocol safety covered by `validateToolCallProtocol` assertions and existing protocol tests.
