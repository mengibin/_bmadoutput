---
storyId: CCV2-1.1
epic: CCV2-1
status: done
date: '2026-05-15'
---

# Story CCV2-1.1: Compression Observability and Protocol Foundation

## Overview

**Epic**: CCV2-1 – Protocol-Aware Historical Context Compression  
**Priority**: First implementation slice  
**Status**: `done`

## Goal

修正压缩结果语义和日志表达，并建立 tool-call group classifier 与 protocol validator，为后续更激进的历史工具调用摘要提供安全基础。

## Business Value

- 避免 logs 把“触发压缩”误报为“有效压缩”。
- 让上下文压缩结果可以被测试和调试。
- 为旧 tool-call group 摘要化建立协议边界，降低后续破坏 OpenAI-compatible tool protocol 的风险。

## Acceptance Criteria

### 1. Accurate Compression State

- **Given** context usage exceeds threshold
- **When** compression reduces token usage
- **Then** result reports `attempted = true`, `effective = true`, and `status = attempted_effective`.

### 2. No-Op Compression Detection

- **Given** context usage exceeds threshold
- **When** compression cannot reduce token usage
- **Then** result reports `attempted = true`, `effective = false`, and `status = attempted_noop`.
- **And** logs say `Context compression attempted but no effective reduction`.

### 3. Under-Threshold State

- **Given** context usage is under threshold
- **When** compression is evaluated
- **Then** result reports `attempted = false`, `effective = false`, and `status = not_needed`.

### 4. Tool-Call Group Classification

- **Given** assistant `tool_calls` followed by matching tool results
- **When** classifier runs
- **Then** it returns a complete `ToolCallGroup` with tool message indexes and classification.

### 5. Protocol Validation

- **Given** outgoing messages contain orphan tool messages or missing tool results
- **When** validator runs
- **Then** validation fails with a specific error code.

## Implementation Summary

- Extended compression result metadata with `attempted`, `effective`, and `status`.
- Changed `compressed` to be a backwards-compatible alias for effective reduction.
- Added no-op detection when a strategy runs but token usage does not shrink.
- Updated Chat, Agent, Chat Tool Loop, and ExecutionEngine log handling to distinguish effective compression from no-op attempts.
- Added `classifyToolCallGroups` for protocol group detection.
- Added `validateToolCallProtocol` for outgoing message protocol validation.
- Added `repairToolCallProtocol` so invalid historical tool protocol can be made safe before sending to the LLM.
- Integrated protocol validation and repair into `CompressionPipeline` so invalid compressed/final output no longer returns invalid fallback messages.
- Added outgoing tool protocol repair in `chatToolLoop` before compression and again after compression, covering both no-compression and failed-fallback paths.
- Added final outgoing tool protocol repair in `ExecutionEngine` before run-mode LLM requests, covering malformed groups that survive context building.
- Extracted shared compression telemetry label helpers and reused them from Chat Tool Loop, ExecutionEngine, and main chat logging.
- Added unit tests for classifier, validator, repair, effective compression, no-op compression, fallback repair, outgoing repair, and compression audit/log labels.

## Files Changed

- `crewagent-runtime/electron/core/context-compression/types.ts`
- `crewagent-runtime/electron/core/context-compression/CompressionPipeline.ts`
- `crewagent-runtime/electron/core/context-compression/compressionTelemetry.ts`
- `crewagent-runtime/electron/core/context-compression/index.ts`
- `crewagent-runtime/electron/core/context-compression/protocol/protocolRepair.ts`
- `crewagent-runtime/electron/core/context-compression/protocol/toolCallGroupClassifier.ts`
- `crewagent-runtime/electron/core/context-compression/protocol/protocolValidator.ts`
- `crewagent-runtime/electron/core/context-compression/context-compression.test.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.ts`
- `crewagent-runtime/electron/services/agentContextBuilder.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/main.ts`

## Verification

```bash
npm test -- electron/core/context-compression/context-compression.test.ts electron/services/chatToolLoop.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/executionEngine.test.ts
npx tsc --noEmit
git diff --check
```

Result:

- 106 focused tests passed.
- TypeScript check passed.
- Diff whitespace check passed.

## Remaining Work

- CCV2-1.2 will implement provider-aware reasoning policy and deterministic old tool-call group compression.
- CCV2-1.3 will implement LLM rolling summary and cross-mode regression hardening.
