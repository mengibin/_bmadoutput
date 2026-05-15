# Story 14.6: SAM Golden Path and Generic SDK Adapter Contract

Status: done

<!-- Created after Story 14.5 reached done under the trusted MCP governance design. Story 14.6 validates the end-to-end SDK Wiki + governance path using SAM fixtures without hardcoding SAM into Runtime. -->

## Story

作为 **SDK Integration Developer**，
我希望用 SAM 验证 SDK Wiki 与 trusted MCP governance 的完整路径，
以便 Abaqus、Ansys、OpenFOAM 等未来 SDK 能复用同一套导入、检索、规划、工具 metadata 与审计合同。

## Acceptance Criteria

### AC-1: SAM SDK Wiki import and list

**Given** a valid SAM SDK Wiki Pack exists
**When** Runtime imports it through `SdkWikiService`
**Then** `sdk_wiki.list_sdks` includes `sam`
**And** the renderer-facing summary does not expose private RuntimeStore absolute paths.

### AC-2: Pressure-load planning golden path

**Given** the SAM pack contains Pressure、Surface、StaticStep and apply-pressure workflow pages
**When** Runtime plans API usage for `给当前板架模型施加向下均布压力载荷`
**Then** `sdk_wiki.plan_api_usage` returns `taskType=apply_pressure_load`
**And** required APIs include Pressure、Surface、StaticStep
**And** every recommended API and plan step has source refs
**And** no missing information is reported.

### AC-3: No invented APIs when knowledge is missing

**Given** the installed SAM Wiki does not match an unsupported intent
**When** Runtime plans API usage
**Then** Runtime returns `missingInformation`
**And** does not invent SDK API names.

### AC-4: Trusted MCP governance golden path

**Given** a mock SDK adapter maps an underlying Runtime tool to SAM `model_write` or `solve` governance metadata
**When** ToolHost executes the tool
**Then** Runtime dispatches without SDK confirmation
**And** audit logs include SDK id, SDK tool name, risk, purpose, governance state, status, duration, and error details when failed.

### AC-5: Generic adapter contract

**Given** another SDK provides the same pack structure and governance metadata
**When** Runtime imports and executes through the same hooks
**Then** Runtime behavior does not branch on `sam`
**And** the documented contract is reusable for non-SAM SDKs.

### AC-6: Tests and review

**Given** Story 14.6 is implemented
**When** running focused verification
**Then** tests cover SAM import/list/search/read/plan, missing-knowledge behavior, governance-audited mock SAM model_write/solve, TypeScript compile, lint, and BMAD code review.

## Design

### Summary

- Use existing `SdkWikiService` and `SdkWikiIndexReader` to validate the SAM knowledge path.
- Use existing `FileSystemToolHost.resolveSdkToolRiskEnvelope` to validate SDK governance audit without adding a real SAM MCP server.
- Add a developer-facing generic SDK adapter contract artifact for future SDK integrations.
- Keep SAM as a fixture/test case only; production Runtime must not branch on SAM-specific names.
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- Adapter Contract: `_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`

### Out of Scope

- No real SAM MCP implementation.
- No hidden LLM calls inside SDK services.
- No Runtime SDK execution confirmation gate.
- No SDK-specific production code branch for SAM.

## Tasks / Subtasks

- [x] Task 1: Story and tech spec (AC: 1-6)
  - [x] 1.1 Create Story 14.6 document
  - [x] 1.2 Create Story 14.6 tech spec
  - [x] 1.3 Move sprint status to in-progress

- [x] Task 2: Generic adapter contract artifact (AC: 5)
  - [x] 2.1 Document SDK Wiki Pack requirements
  - [x] 2.2 Document MCP execution safety boundary
  - [x] 2.3 Document governance metadata and audit events
  - [x] 2.4 Document SAM pressure-load example and non-SAM reuse rules

- [x] Task 3: SAM Wiki golden path tests (AC: 1-3)
  - [x] 3.1 Verify import/list for SAM fixture
  - [x] 3.2 Verify pressure-load plan returns Pressure/Surface/StaticStep with source refs
  - [x] 3.3 Verify missing unsupported intent returns `missingInformation`
  - [x] 3.4 Verify registry system message is compact and does not preload page content

- [x] Task 4: Governance golden path tests (AC: 4-5)
  - [x] 4.1 Verify mock SAM `model_write` audit path
  - [x] 4.2 Verify mock SAM `solve` audit path without confirmation
  - [x] 4.3 Verify audit events carry SDK metadata and no Runtime confirmation artifacts

- [x] Task 5: Verification and review (AC: 6)
  - [x] 5.1 Run focused tests
  - [x] 5.2 Run targeted lint and TypeScript
  - [x] 5.3 Create BMAD code review and move status to done

## Dev Notes

### Codebase Findings

- `SdkWikiService` already supports import/list/search/read/resolve/expand/plan and has inline fixtures suitable for SAM golden path tests.
- `addPressurePlanningFixture()` already models Pressure、Surface、StaticStep and `workflow.apply-pressure-load`.
- `FileSystemToolHost` already supports optional SDK governance metadata resolver and audit events from Story 14.5.
- The correct 14.6 implementation should mainly add explicit golden-path coverage and documentation; production SAM branching is intentionally avoided.

### References

- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic: `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- Development plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Story 14.5: `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `npm test -- electron/services/sdkWikiService.test.ts -t "Story 14.6"` passed: 2 focused tests.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "Story 14.6"` passed: 1 focused test.
- `npm exec eslint -- electron/services/sdkWikiService.test.ts electron/services/fileSystemToolHost.test.ts --max-warnings 0` passed.
- `npm exec tsc -- --noEmit` passed.
- `git diff --check` passed for touched 14.6 docs and runtime test files.

### Completion Notes List

- Added SAM pressure-load golden path coverage for import/list/search/read/plan.
- Verified Pressure、Surface、StaticStep are surfaced with source refs.
- Verified missing unsupported intent returns `missingInformation` instead of invented APIs.
- Added mock SAM adapter governance-audit coverage for `model_write` and `solve`.
- Added generic SDK adapter contract artifact documenting pack rules, MCP safety boundary, governance metadata, and audit events.
- No production SAM-specific branching was added.

### File List

- `_bmad-output/implementation-artifacts/14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`
- `_bmad-output/implementation-artifacts/code-review-14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`

### Change Log

- 2026-05-15 - Created Story 14.6 for SAM golden path and generic SDK adapter contract; status set to in-progress.
- 2026-05-15 - Implemented Story 14.6 tests and adapter contract; status moved to done.
