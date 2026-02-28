# Story MDE-1.1: Multimodal Model Configuration + Capability Guard

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime user**,
I want to configure a dedicated multimodal model profile and validate capability before extraction,
So that image/PDF extraction can run on a supported model and fail deterministically when unsupported.

## Acceptance Criteria

1. **Multimodal profile persistence**
   - **Given** I open Runtime Settings and edit multimodal model fields
   - **When** I save configuration
   - **Then** `provider/baseUrl/model/apiKey/timeout` are persisted in Runtime settings
   - **And** values are restored after app restart

2. **Configuration validation**
   - **Given** I submit invalid multimodal settings (e.g. missing provider/model or invalid timeout)
   - **When** save is triggered
   - **Then** save is rejected with actionable validation messages
   - **And** previous valid configuration remains unchanged

3. **Capability guard before extraction**
   - **Given** a multimodal extraction request is about to run
   - **When** configured model does not support multimodal input
   - **Then** Runtime returns structured error `LLM_MULTIMODAL_NOT_SUPPORTED`
   - **And** no outbound extraction request is sent to the provider

4. **Clear runtime observability**
   - **Given** capability check executes
   - **When** extraction starts or is blocked
   - **Then** logs include `runId/provider/model/capabilityCheck/result/errorCode`
   - **And** API key is never logged

5. **Backward compatibility**
   - **Given** existing text-only chat/workflow runs
   - **When** multimodal profile is introduced
   - **Then** current `llm` config behavior remains unchanged
   - **And** no regression occurs for non-multimodal paths

## Design

### Summary
- Add a dedicated `multimodalLlm` profile in settings, separate from existing `llm`.
- Add `MultimodalCapabilityService` to validate provider/model support before extraction.
- Introduce a single guard entry (`assertMultimodalReady`) for extraction path.
- Standardize unsupported-model failure as `LLM_MULTIMODAL_NOT_SUPPORTED`.

### UX / UI
- Settings page adds a new section/tab: `Multimodal LLM`.
- Form fields: `provider`, `baseUrl`, `model`, `apiKey`, `timeout`.
- Save feedback includes explicit field-level error hints.
- Optional "Test Capability" button can check profile health without running extraction.

### API / Contracts
```ts
export interface RuntimeSettings {
  llm: LLMConfig
  multimodalLlm?: LLMConfig
}

export interface CapabilityGuardError {
  code: 'LLM_MULTIMODAL_NOT_SUPPORTED'
  message: string
  details?: unknown
}
```

### Data / Storage
- Persist `multimodalLlm` in `runtime-store/settings.json`.
- Keep `llm` and `multimodalLlm` independent to avoid cross-overwrite.
- Runtime defaults can initialize `multimodalLlm` from `llm` only once (migration step), then decouple.

### Errors / Edge Cases
- Missing multimodal config: fail before extraction with structured error.
- Provider supports multimodal but model unknown: fail closed (not supported) unless explicit allowlist/probe confirms.
- Timeout or network errors during capability probe: treat as non-supported for current run and provide actionable details.

### Test Plan
- Unit:
  - settings validation for `multimodalLlm`
  - capability guard returns `LLM_MULTIMODAL_NOT_SUPPORTED` and blocks outbound call
- Integration:
  - settings save/load round-trip for multimodal profile
  - extraction entrypoint uses guard before provider request
- Regression:
  - existing text-only `llm` path unchanged

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | **MOD**: extend `RuntimeSettings` with `multimodalLlm`, defaults, migration, validation |
| `crewagent-runtime/src/stores/appStore.ts` | **MOD**: app state + save/load flow for `multimodalLlmConfig` |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | **MOD**: add multimodal settings section and validation UI |
| `crewagent-runtime/electron/services/multimodalCapabilityService.ts` | **ADD**: provider/model capability resolver and guard helpers |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | **MOD**: invoke capability guard at extraction entrypoint |
| `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` | **MOD**: add guard tests for unsupported model path |
| `crewagent-runtime/electron/stores/runtimeStore.test.ts` | **MOD**: add multimodal settings persistence/validation tests |

## Tasks / Subtasks

- [x] 1) Settings model extension (AC: 1,2,5)
  - [x] 1.1 Add `multimodalLlm` in main/renderer settings types
  - [x] 1.2 Add default + migration strategy
  - [x] 1.3 Add schema/field validation and error messages

- [x] 2) Settings UI support (AC: 1,2)
  - [x] 2.1 Add `Multimodal LLM` section in Settings page
  - [x] 2.2 Bind form state + save flow to `settings:update`
  - [x] 2.3 Keep existing `llm` section behavior unchanged

- [x] 3) Capability guard service (AC: 3,4)
  - [x] 3.1 Implement `MultimodalCapabilityService` (allowlist/probe)
  - [x] 3.2 Implement guard helper returning `LLM_MULTIMODAL_NOT_SUPPORTED`
  - [x] 3.3 Integrate guard into extraction start path

- [x] 4) Observability and error propagation (AC: 3,4)
  - [x] 4.1 Emit structured log fields for capability check
  - [x] 4.2 Guarantee no secret logging (`apiKey` redaction)
  - [x] 4.3 Expose stable structured error to UI/agent loop

- [x] 5) Testing and regression (AC: 1~5)
  - [x] 5.1 Unit tests for settings validation and guard behavior
  - [x] 5.2 Integration test for "unsupported model -> blocked request"
  - [x] 5.3 Regression test for existing text-only LLM flow

## Dev Notes

- This story only delivers configuration and capability-guard foundation.
- `media.extract` first-class exposure and `fs.read` binary fallback are covered in Story MDE-1.2.
- Drag-and-drop batch flow and schema orchestration are covered in Story MDE-1.3.
- Keep error shape aligned with existing runtime tool/LLM errors: `{ code, message, details? }`.

### Project Structure Notes

- Reuse existing settings IPC channels: `settings:get`, `settings:update`.
- Keep guard logic in main process services; renderer is display/persistence only.
- Do not add provider-specific hard dependency in renderer layer.

### References

- Epic: `_bmad-output/epics-runtime-multimodal-data-extraction.md` (Story MDE-1.1)
- PRD Addendum: `_bmad-output/prd-runtime-multimodal-data-extraction.md`
- Architecture Addendum: `_bmad-output/architecture/runtime-multimodal-data-extraction-architecture.md`
- Parent PRD FR mapping: `_bmad-output/prd.md` (`FR-MULTI-01..04`)
- Design: `_bmad-output/implementation-artifacts/design-mde-1-1-multimodal-model-configuration-capability-guard.md`
- Validation: `_bmad-output/implementation-artifacts/validation-report-story-mde-1-1.md`
- Code Review: `_bmad-output/implementation-artifacts/code-review-mde-1-1-multimodal-model-configuration-capability-guard.md`

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `npm -C crewagent-runtime run test -- electron/services/multimodalCapabilityService.test.ts electron/services/fileSystemToolHost.test.ts electron/stores/runtimeStore.test.ts`
- `npm -C crewagent-runtime run build:ci`

### Completion Notes List
- Added dedicated `multimodalLlm` settings model in main/renderer store with persistence and safe merge.
- Added `Multimodal LLM` settings section in Runtime UI (provider/baseUrl/model/apiKey/timeout/contextWindow).
- Implemented multimodal capability guard service with deterministic `LLM_MULTIMODAL_NOT_SUPPORTED` error.
- Added MDE-1.1 seam in `FileSystemToolHost` (`media.extract` path checks capability before execution path).
- Hardened settings merge so invalid multimodal `timeout/contextWindow` values do not overwrite prior valid values.
- Added tests for capability service, runtime settings persistence/validation, and guard behavior.

### File List
- `_bmad-output/implementation-artifacts/mde-1-1-multimodal-model-configuration-capability-guard.md`
- `_bmad-output/implementation-artifacts/design-mde-1-1-multimodal-model-configuration-capability-guard.md`
- `_bmad-output/implementation-artifacts/validation-report-story-mde-1-1.md`
- `_bmad-output/implementation-artifacts/code-review-mde-1-1-multimodal-model-configuration-capability-guard.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- `crewagent-runtime/electron/services/multimodalCapabilityService.ts`
- `crewagent-runtime/electron/services/multimodalCapabilityService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`

### Change Log
- 2026-02-27: Created story draft for MDE-1.1 (ready-for-design).
- 2026-02-27: Added design artifact and promoted status to ready-for-dev.
- 2026-02-27: Added validate-create-story report (ready-for-dev confirmed).
- 2026-02-27: Implemented dev-story changes and moved story to review.
- 2026-02-27: Aligned invalid multimodal timeout/contextWindow handling with AC2 and synchronized sprint status to review.
- 2026-02-27: Completed code-review artifact for MDE-1.1 (no open issues).

## Dependencies

- Epic MDE-1
- Existing runtime settings infrastructure (Story 5-5)
- Existing LLM adapter/tool loop infrastructure (Story 4-5/4-6)

## Verification Plan

1. Save valid multimodal settings and verify persistence after restart.
2. Save invalid multimodal settings and verify rejection with field errors.
3. Trigger extraction with unsupported model and verify `LLM_MULTIMODAL_NOT_SUPPORTED`.
4. Confirm provider request is not sent in unsupported case.
5. Run text-only chat flow and verify no behavior regression.
