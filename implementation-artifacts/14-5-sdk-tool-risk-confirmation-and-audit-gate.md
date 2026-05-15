# Story 14.5: Trusted SDK Tool Governance and Audit

Status: done

<!-- Revised on 2026-05-15 after the trust-boundary decision: MCP/integration software owns SDK execution safety. Runtime does not add an SDK risk confirmation gate. -->

## Story

作为 **Runtime User**，
我希望 SDK tool call 在可信 MCP 边界下被风险标注、自动执行并记录审计，
以便 Agent 能自主推进 SDK 操作，同时 CrewAgent 保留可追溯的治理日志。

## Acceptance Criteria

### AC-1: Governance metadata contract

**Given** Runtime 接收到 SDK execution tool call
**When** SDK/MCP adapter 将其暴露给 ToolHost
**Then** tool call 可提供 `sdkId`、`toolName`、`risk`、可选 `purpose`、`targetPath`、`targetObject` 的 governance envelope
**And** `risk` 只能是 `read`、`model_write`、`file_write`、`solve`、`destructive`。

### AC-2: Runtime does not confirm SDK risk

**Given** SDK tool call classified as `file_write`、`solve` or `destructive`
**When** effective Runtime tool policy permits the underlying MCP/local tool
**Then** Runtime must not return `SDK_TOOL_CONFIRMATION_REQUIRED`
**And** Runtime must not require a `confirmationToken` or `WIDGET_SUBMIT` before dispatch.

### AC-3: MCP owns execution safety

**Given** MCP exposes an SDK execution tool
**When** Runtime dispatches the tool
**Then** Runtime treats the tool as safe within existing effective tool policy
**And** path safety、destructive safety、solve resource/license guard、domain confirmation belong to MCP/integration software.

### AC-4: Metadata warnings do not block execution

**Given** SDK governance metadata is incomplete or invalid
**When** Runtime observes the tool call
**Then** Runtime logs a metadata warning
**And** Runtime must not reject the tool solely because metadata is missing `purpose`、`targetPath` or has an invalid risk value.

### AC-5: Existing effective tool policy remains authoritative

**Given** system/agent effective tool policy disables an MCP/local tool
**When** SDK governance metadata says the SDK action is safe
**Then** Runtime must still deny the underlying tool through existing policy
**And** SDK metadata must never grant a disabled tool.

### AC-6: Audit logging

**Given** SDK governance metadata is available or fails to resolve
**When** Runtime handles the tool call
**Then** execution JSONL must include SDK audit events with tool name, SDK id when available, risk, purpose, target path/object, governance state, warnings, execution status, duration, and error details where available.

### AC-7: Tests

**Given** Story 14.5 is implemented
**When** running focused tests
**Then** tests must cover autonomous execution for solve/destructive-style metadata, metadata warning behavior, ToolHost audit events, TypeScript compile, and lint.

## Design

### Summary

- Keep shared SDK tool risk names because they are useful observability metadata.
- Rework the previous `SdkToolRiskPolicyService` into an audit-only governance evaluator.
- Keep the optional ToolHost SDK risk envelope resolver, but use it to observe and log rather than block.
- Remove Runtime SDK confirmation token issuance/validation and chat/run `WIDGET_SUBMIT` interception.
- Keep SDK Wiki query tools read-only and separate from SDK execution tools.
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-5-sdk-tool-risk-confirmation-and-audit-gate.md`

### Design Decisions

- MCP/集成软件是执行安全边界；Runtime 不在 SDK risk 层重复确认。
- Runtime existing effective tool policy still controls whether a tool/server is visible and executable.
- `model_write` missing `purpose` is a metadata warning, not a rejection.
- `file_write` missing `targetPath` is a metadata warning, not a confirmation request.
- `solve` and `destructive` are auditable risk labels, not Runtime confirmation triggers.
- SDK metadata resolver failure is logged as `sdk_tool.metadata_warning` and does not stop dispatch by itself.
- Audit events use `sdk_tool.requested` / `sdk_tool.metadata_warning` before dispatch and `sdk_tool.executed` / `sdk_tool.failed` after dispatch.

### Out of Scope

- No real SAM execution tool implementation; Story 14.6 adds the SAM golden path.
- No SDK-specific API hardcoding.
- No new UI confirmation surface.
- No replacement for MCP-side execution safety policy.
- No change to SDK Wiki read/query tools.

## Tasks / Subtasks

- [x] Task 1: BMAD redesign documents (AC: 1-7)
  - [x] 1.1 Revise source requirement archive for trusted MCP governance
  - [x] 1.2 Revise PRD/Epic/Architecture/development plan trust boundary
  - [x] 1.3 Revise Story 14.5 and tech spec

- [x] Task 2: Shared governance contract and evaluator (AC: 1-4)
  - [x] 2.1 Remove Runtime confirmation request/token types
  - [x] 2.2 Keep risk metadata and argument fingerprinting for audit
  - [x] 2.3 Convert policy service to allow/observe decisions with warnings

- [x] Task 3: ToolHost boundary and audit (AC: 2-6)
  - [x] 3.1 Remove `approveSdkToolConfirmationFromInput`
  - [x] 3.2 Remove chat/run SDK confirmation submit interception
  - [x] 3.3 Dispatch underlying tools after metadata observation
  - [x] 3.4 Append SDK governance audit events
  - [x] 3.5 Preserve existing effective tool policy behavior

- [x] Task 4: Tests and verification (AC: 7)
  - [x] 4.1 Update policy service tests
  - [x] 4.2 Update ToolHost boundary audit tests
  - [x] 4.3 Run focused tests, lint, TypeScript, and build verification

## Dev Notes

### Codebase Findings

- `FileSystemToolHost.executeToolCall()` is the central boundary before local tools, MCP tools, terminal tools, SDK Wiki tools, and FS tools execute.
- Existing audit logs append JSONL records to `@state/logs/execution.jsonl`.
- The previous 14.5 implementation added a two-stage confirmation token flow; this is now intentionally removed because it places execution safety in Runtime rather than MCP.
- Chat/run continuations should no longer consume SDK-risk `WIDGET_SUBMIT` input or inject a confirmation token into user input.

### References

- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic: `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- Development plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Requirements: `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/03-tool-risk-confirmation-and-logging.md`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `npm test -- electron/services/sdkToolRiskPolicyService.test.ts` passed: 1 test file, 6 tests.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK-risked"` passed: 1 focused test.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "Story 14.5"` passed: 2 focused tests.
- `npm exec eslint -- shared/sdkToolRiskTypes.ts electron/services/sdkToolRiskPolicyService.ts electron/services/sdkToolRiskPolicyService.test.ts electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/main.ts --max-warnings 0` passed.
- `npm exec tsc -- --noEmit` passed.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "ui.ask_user"` passed: 1 focused test.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK Wiki"` passed: 2 focused tests.
- `npm run build:ci` passed; remaining warnings are existing Vite eval/chunk-size/dynamic-import warnings.
- `git diff --check` passed for touched Story 14.5 files.

### Completion Notes List

- Revised BMAD source requirement, PRD, Epic, Architecture, development plan, Story, and tech spec to reflect the trusted MCP execution boundary.
- Removed Runtime SDK confirmation request/token decision types.
- Converted `SdkToolRiskPolicyService` into an audit-only governance evaluator.
- Removed `approveSdkToolConfirmationFromInput()` and chat/run SDK-risk `WIDGET_SUBMIT` interception.
- ToolHost now dispatches SDK-risked tools immediately after metadata observation when existing effective tool policy permits the underlying tool.
- Added `sdk_tool.requested`, `sdk_tool.metadata_warning`, `sdk_tool.executed`, and `sdk_tool.failed` audit events.
- Added tests for autonomous execution and resolver-failure metadata warning behavior.

### File List

- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/03-tool-risk-confirmation-and-logging.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/shared/sdkToolRiskTypes.ts`
- `crewagent-runtime/electron/services/sdkToolRiskPolicyService.ts`
- `crewagent-runtime/electron/services/sdkToolRiskPolicyService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/main.ts`

### Change Log

- 2026-05-15 - Revised Story 14.5 from Runtime confirmation gate to trusted MCP governance and audit.
- 2026-05-15 - Implemented audit-only SDK governance evaluator and ToolHost integration; status moved to done.
