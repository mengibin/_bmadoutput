---
stepsCompleted: [1, 2, 3, 4, 5, 6]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-context-compression-v2.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/runtime-context-compression-v2-architecture.md'
workflowType: 'epics'
lastStep: 6
project_name: 'CrewAgent Runtime Context Compression V2'
user_name: 'Mengbin'
date: '2026-05-14'
---

# CrewAgent Runtime Context Compression V2 - Epic and Development Plan

## Overview

本文件将 Context Compression V2 拆解为一个核心 Epic 和四条可独立交付的 Story。目标是从当前“触发式启发压缩”升级到“协议安全、provider-aware、可观测、可 LLM 摘要”的通用历史压缩能力，并在历史压缩仍不足时处理当前工具链过长的问题。

来源文档：

- `_bmad-output/prd-runtime-context-compression-v2.md`
- `_bmad-output/architecture/runtime-context-compression-v2-architecture.md`

## Delivery Status

- `CCV2-1.1` Compression Observability and Protocol Foundation: `done`
- `CCV2-1.2` Protocol-Aware Structural Compression: `done`
- `CCV2-1.3` LLM Summary and Cross-Mode Integration: `done`
- `CCV2-1.4` Active Tool-Loop Compression and Final Fallback: `done`
- Closure artifact: `_bmad-output/implementation-artifacts/epic-ccv2-retrospective.md`

## Requirements Inventory

### Functional Requirements

- FR-CCV2-01: 区分 attempted、effective、no-op、failed。
- FR-CCV2-02: 输出完整压缩 metadata。
- FR-CCV2-03: 保证 tool-call protocol 安全。
- FR-CCV2-04: 识别并分类 tool-call group。
- FR-CCV2-05: provider-aware 处理 `reasoning_content`。
- FR-CCV2-06: 实现 deterministic structural compression。
- FR-CCV2-07: 引入 LLM rolling summary。
- FR-CCV2-08: LLM summary 失败时安全回退。
- FR-CCV2-09: Chat、Agent、Run 共用 V2 pipeline。
- FR-CCV2-10: Plan mode 兼容但不做 plan-only 修复。
- FR-CCV2-11: logs 和 context usage 准确可观测。
- FR-CCV2-12: 提供工具密集 reasoning 密集的 synthetic fixtures。
- FR-CCV2-13: 历史压缩后仍接近上下文上限时，触发当前工具链压缩。
- FR-CCV2-14: 当前工具链压缩必须按完整 tool-call group 处理，不破坏协议。
- FR-CCV2-15: 压缩仍不到位时进入明确 final fallback，不直接发送超预算请求。

### Non-Functional Requirements

- NFR-CCV2-01: Protocol safety。
- NFR-CCV2-02: Reliability。
- NFR-CCV2-03: Performance。
- NFR-CCV2-04: Privacy and data minimization。
- NFR-CCV2-05: Backward compatibility。
- NFR-CCV2-06: Configurability。

## Epic List

### Epic CCV2-1: Protocol-Aware Historical Context Compression

**Goal**: 交付一套跨 Chat、Agent、Run/Workflow 的 V2 历史上下文压缩能力，使工具密集、reasoning 密集的长会话能够被有效压缩，同时保持 provider 和 OpenAI tool protocol 兼容。

**FRs covered**: FR-CCV2-01 ~ FR-CCV2-12, NFR-CCV2-01 ~ NFR-CCV2-06

**Deliverables**:

- Accurate attempted/effective/no-op compression metadata。
- Tool-call group classifier。
- Outgoing message protocol validator。
- Provider-aware reasoning policy。
- Deterministic historical tool group summary。
- LLM rolling `CONVERSATION_SUMMARY` service。
- Chat、Agent、Run/Workflow integration。
- Synthetic regression fixtures。
- Active tool-loop `TOOL_LOOP_SUMMARY` and final fallback。

## Recommended Implementation Order

1. **CCV2-1.1 Compression Observability and Protocol Foundation**
   先修 logs / metadata，并建立 tool-call group classifier 与 protocol validator。
2. **CCV2-1.2 Protocol-Aware Structural Compression**
   再处理 DeepSeek/Kimi reasoning policy，并实现旧 tool-call group 的 deterministic summary。
3. **CCV2-1.3 LLM Summary and Cross-Mode Integration**
   最后加入 LLM rolling summary，并接入 Chat、Agent、Run/Workflow、Plan Chat。
4. **CCV2-1.4 Active Tool-Loop Compression and Final Fallback**
   在历史压缩仍不足时，压缩当前用户请求内部已经完成的旧工具调用组，并提供仍超限时的最终兜底。

## Story CCV2-1.1: Compression Observability and Protocol Foundation

As a **Runtime Developer**,  
I want compression telemetry and tool-call protocol boundaries to be explicit,  
So that Runtime can report real compression results and future compression changes cannot create invalid message sequences.

### Acceptance Criteria

**Given** context usage exceeds threshold  
**When** compression runs and output tokens are lower than input tokens  
**Then** metadata sets `attempted = true` and `effective = true`.

**Given** context usage exceeds threshold  
**When** compression runs but output tokens do not decrease  
**Then** metadata sets `attempted = true` and `effective = false`  
**And** logs say `Context compression attempted but no effective reduction`.

**Given** context usage is under threshold  
**When** compression is evaluated  
**Then** metadata sets `attempted = false`  
**And** no compression success log is emitted.

**Given** an assistant message with `tool_calls` followed by matching tool messages  
**When** the classifier runs  
**Then** it returns a complete `ToolCallGroup`.

**Given** a tool message without a preceding assistant tool call  
**When** validator runs  
**Then** validation fails with an orphan tool message error.

**Given** compression summarizes an old tool-call group  
**When** validator runs  
**Then** no summarized message contains `tool_calls`  
**And** no removed tool result remains.

### Technical Scope

- Extend `CompressionResult` or introduce `CompressionAttemptResult`.
- Update `CompressionPipeline.compressIfNeeded`.
- Update chat and run logs.
- Update context usage emission to use real after usage.
- Add `protocol/toolCallGroupClassifier.ts`.
- Add `protocol/protocolValidator.ts`.
- Add group classification fields:
  - `current_loop`
  - `recent`
  - `historical`
  - `invalid`
- Integrate validator into V2 pipeline.
- Add tests for no-op compression and protocol validation.

### Verification

- Unit test: no-op strategy returns `effective = false`.
- Integration test: log message changes from misleading applied log to attempted/no-op log.
- Unit tests for complete, incomplete, duplicate, invalid assistant tool-call id, and orphan tool protocols.
- Integration test that compressed output passes validator.

## Story CCV2-1.2: Protocol-Aware Structural Compression

As a **Runtime User**,  
I want old tool-heavy and reasoning-heavy history compressed structurally,  
So that long tool-use sessions can continue without breaking DeepSeek/Kimi or OpenAI tool-call protocol.

### Acceptance Criteria

**Given** provider/model indicates DeepSeek thinking mode  
**When** recent tool-call groups are compressed  
**Then** their `reasoning_content` is preserved.

**Given** a historical non-tool assistant message with `reasoning_content`  
**When** compression runs  
**Then** reasoning can be stripped while visible assistant content remains.

**Given** an old complete tool-call group  
**When** it is summarized  
**Then** original `reasoning_content` is removed because the original tool protocol is no longer sent.

**Given** Kimi `thinking.keep: "all"` is enabled  
**When** compression pressure requires reasoning reduction  
**Then** the policy decision is logged clearly.

**Given** a history with hundreds of old complete tool-call groups  
**When** compression runs  
**Then** old groups outside the recent preservation window are converted into ordinary summary messages.

**Given** a summarized tool group  
**When** outgoing messages are inspected  
**Then** the summary message has no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.

**Given** old tool result content is large  
**When** deterministic summary is generated  
**Then** summary includes tool name, relevant path/args preview, result preview, and outcome when inferable.

**Given** structure compression completes  
**When** token usage is recalculated  
**Then** token count decreases significantly or metadata explains why target was not reached.

### Technical Scope

- Add `protocol/providerReasoningPolicy.ts`.
- Derive policy from LLM settings and model name.
- Apply policy during structural compression.
- Preserve current loop and recent tool-call reasoning.
- Add deterministic tool group summary builder.
- Replace old group with `TOOL_CALL_HISTORY_SUMMARY`.
- Drop corresponding tool messages.
- Strip old reasoning.
- Preserve recent groups.

### Verification

- Unit tests for DeepSeek, Kimi default, Kimi keep-all, unknown provider.
- Fixture test with 90 percent reasoning-bearing tool-call messages.
- Synthetic 300-group fixture drops below target or records targetExceeded reason.
- Protocol validator passes.
- Existing context compression tests continue passing.

## Story CCV2-1.3: LLM Summary and Cross-Mode Integration

As a **Runtime User**,  
I want old history summarized semantically and V2 compression used consistently across Runtime modes,  
So that Chat, Agent, Plan Chat, and Run/Workflow can all survive long contexts without losing core intent.

### Acceptance Criteria

**Given** structural compression removes many historical messages  
**When** LLM summary is available  
**Then** the pipeline generates or updates `CONVERSATION_SUMMARY`.

**Given** an existing `CONVERSATION_SUMMARY` exists  
**When** new old history is summarized  
**Then** the summary is merged rather than duplicated.

**Given** summary LLM fails or times out  
**When** the main request is prepared  
**Then** deterministic fallback summary is used  
**And** the main request still proceeds.

**Given** generated summary is parsed  
**When** it lacks required headings  
**Then** it is rejected and deterministic fallback is used.

**Given** Chat mode has tool-heavy history  
**When** a request is built  
**Then** V2 compression runs and emits accurate metadata.

**Given** Agent mode has equivalent history  
**When** a request is built  
**Then** V2 compression behavior is equivalent to Chat mode.

**Given** Run/Workflow mode has long tool history  
**When** ExecutionEngine builds context  
**Then** V2 compression runs before request submission.

**Given** Plan Chat injects plan context  
**When** history compression runs  
**Then** metadata separates history compression from plan/system prefix tokens.

### Technical Scope

- Add `summaries/ConversationSummaryService.ts`.
- Add fixed summary prompt.
- Add parser/validator for required headings.
- Add timeout and bounded input.
- Add deterministic fallback.
- Integrate V2 in `chatContextBuilder`.
- Integrate V2 in `agentContextBuilder`.
- Integrate V2 in `chatToolLoop`.
- Integrate V2 in `executionEngine`.
- Update context usage UI payload if needed.
- Add regression tests.

### Verification

- Unit test successful summary parse.
- Unit test malformed summary fallback.
- Unit test timeout fallback.
- Integration test summary inserted into outgoing context.
- Run context compression unit tests.
- Run chat tool loop tests.
- Run execution engine tests covering compression.
- Add plan chat integration test for history compression metadata.

## Story CCV2-1.4: Active Tool-Loop Compression and Final Fallback

As a **Runtime User**,  
I want the current tool-call loop to be compressed when historical compression is still not enough,  
So that a long-running task that creates hundreds of function calls in one user turn does not fail just because there is no old history left to compress.

### Acceptance Criteria

**Given** historical compression has already run or been evaluated  
**When** the outgoing context remains near or above target  
**Then** Runtime evaluates active tool-loop compression.

**Given** the active loop contains complete older tool-call groups  
**When** active tool-loop compression runs  
**Then** those older groups may be replaced by ordinary `TOOL_LOOP_SUMMARY` assistant messages.

**Given** a tool-call group is incomplete  
**When** active tool-loop compression runs  
**Then** that group is preserved exactly and is not summarized.

**Given** recent complete tool-call groups exist  
**When** active tool-loop compression runs  
**Then** the recent preservation window remains intact.

**Given** a group is summarized  
**When** the outgoing messages are inspected  
**Then** the summary has no `tool_calls`, no `tool_call_id`, and no `reasoning_content`.

**Given** DeepSeek or Kimi recent tool-call reasoning should be preserved by policy  
**When** active tool-loop compression runs  
**Then** preserved recent groups keep the provider-required reasoning fields.

**Given** historical compression plus active tool-loop compression still cannot reduce enough  
**When** Runtime prepares the next LLM request  
**Then** it uses deterministic final fallback or stops with `CONTEXT_STILL_TOO_LARGE` instead of blindly sending an over-budget request.

### Technical Scope

- Add active loop compression stage after historical compression.
- Reuse the current loop boundary already passed through `compressHistoricalMessages`.
- Classify complete, incomplete, recent, and older current-loop tool groups.
- Build deterministic `TOOL_LOOP_SUMMARY` messages.
- Recalculate usage after active-loop compression.
- Extend telemetry metadata for active-loop compression.
- Add final fallback behavior for still-over-target requests.
- Add synthetic first-session first-turn 300+ tool-call fixture.

### Verification

- Unit tests for active-loop group classification.
- Unit tests for `TOOL_LOOP_SUMMARY` field safety.
- Integration test with first session and first user request producing 300+ complete tool groups.
- Regression test that short normal tool loops do not trigger active-loop compression.
- Protocol validation must pass after active-loop compression and final fallback.

## Delivery Scope by Layer

### Core Runtime Scope

- New compression result model.
- Tool-call classifier.
- Protocol validator.
- Provider reasoning policy.
- Deterministic summary builders.
- LLM summary service.

### Chat / Agent Scope

- Use V2 compression during context building.
- Use V2 loop compression during multi-turn tool use.
- Emit accurate context usage and logs.

### Run / Workflow Scope

- Replace old compression calls with V2 result handling.
- Preserve run directive and current node context.
- Emit audit events with V2 metadata.

### Plan Mode Scope

- Continue injecting plan context.
- Compress history using V2.
- Report plan context token pressure separately.

## Dependencies

- Existing `ConversationMessage` supports `tool`, `tool_calls`, `tool_call_id`, and `reasoning_content`.
- Existing `CompressionPipeline` and `ContextUsageMonitor`.
- Existing `llmAdapter` for optional summary call.
- Existing context builders for Chat, Agent, and Run.

## Risks and Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | Compression breaks tool protocol | Add protocol validator and conservative fallback |
| R2 | DeepSeek rejects missing reasoning | Preserve recent tool-call reasoning; summarize whole old groups only |
| R3 | Kimi keep-all users expect preserved thinking | Log policy and support configuration |
| R4 | LLM summary drops important details | Fixed schema, recent window preservation, deterministic facts |
| R5 | Summary LLM latency | Timeout and bounded input |
| R6 | Plan context still exceeds budget | Separate telemetry and follow-up plan context compression story if needed |

## Acceptance Checklist

- [x] Compression logs distinguish attempted, effective, no-op, and failed fallback.
- [x] Tool-call group classifier has unit coverage.
- [x] Protocol validator runs before outgoing request.
- [ ] Historical non-tool reasoning can be stripped.
- [ ] Old complete tool-call groups can be summarized into ordinary messages.
- [ ] Recent DeepSeek/Kimi tool-call reasoning is preserved.
- [ ] LLM rolling summary is available with deterministic fallback.
- [ ] Chat, Agent, and Run all use V2 pipeline.
- [ ] Plan mode reports history compression separately from plan context.
- [ ] Synthetic tool-heavy reasoning-heavy fixture proves effective reduction.

## Suggested Test Command Set

```bash
npm test -- electron/core/context-compression/context-compression.test.ts
npm test -- electron/services/chatToolLoop.test.ts
npm test -- electron/services/chatContextBuilder.test.ts
npm test -- electron/services/agentContextBuilder.test.ts
npm test -- electron/services/executionEngine.test.ts
```
