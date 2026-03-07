# Story MDE-1.3: Drag-and-Drop Batch Extraction + Schema Contract + Observability

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime user**,  
I want to drag files/folders into chat and let LLM decide whether to read or extract, with optional schema-constrained batch extraction,  
So that multimodal results are directly usable by downstream systems without forcing extraction on every attachment.

## Acceptance Criteria

1. **Chat drag-and-drop ingestion**
   - **Given** I drag files/folders into chat input
   - **When** I send the message
   - **Then** payload includes attachment metadata (`path/name/mime/isDirectory`)
   - **And** backend receives alias-resolved paths under sandbox roots
   - **And** drag-and-drop itself does **not** force batch extraction; LLM may decide `fs.read` or `media.extract` based on intent.

2. **Recursive discovery + deterministic order**
   - **Given** a dropped folder with nested subfolders
   - **When** explicit extraction ingestion runs
   - **Then** supported files (`.pdf/.png/.jpg/.jpeg/.webp`) are discovered recursively
   - **And** processing order is deterministic (default: relative filename ascending).

3. **Schema source: provided**
   - **Given** `schema` is provided
   - **When** extraction runs
   - **Then** each extraction result marks `schemaSource=provided`
   - **And** exposes `schemaUsed` in result metadata.

4. **Schema source: generated**
   - **Given** `schema` is absent and `schemaIntent` exists
   - **When** extraction runs
   - **Then** schema is generated before extraction
   - **And** result marks `schemaSource=generated`.

5. **Strict validation behavior**
   - **Given** strict mode is enabled
   - **When** extracted data violates schema
   - **Then** runtime returns structured validation error (`MEDIA_SCHEMA_INVALID`)
   - **And** invalid records are not silently written into batch output.

6. **Batch structured result output (default)**
   - **Given** multiple files/pages are processed
   - **When** batch completes
   - **Then** runtime returns structured payload (`runId/generatedAt/records/errors/stats`)
   - **And** each record includes at least `sourceFile/page/data`
   - **And** persistence is optional via request flag (`persistArtifact=true`) rather than mandatory.

7. **Observability contract**
   - **Given** extraction is executed
   - **When** logs/audit events are emitted
   - **Then** each file/page event contains `runId/sourceFile/page/model/status/duration/errorCode`
   - **And** no secrets (e.g. apiKey/raw binary payload) are logged.

8. **E2E deterministic behavior**
   - **Given** a fixture with mixed image + PDF + unsupported model case
   - **When** E2E suite runs
   - **Then** it validates structured output contract, schema compliance, and deterministic unsupported-model error (`LLM_MULTIMODAL_NOT_SUPPORTED`).

## Design

> Design completed. Story is ready for development.

### UX / UI
- Reuse existing chat attachment UX (drag-over highlight + attachment chips).
- Show deterministic processing summary (processed/succeeded/failed/truncated).
- Batch failure should remain partial-success tolerant: failed items are listed with error code/details.

### API / Contracts
- Extend send payload to carry structured attachments:
  - `attachments[]: { path, name, mimeType?, isDirectory? }`
- Batch orchestration contract:
  - input: attachment list + extraction intent (`instruction/schema/schemaIntent/strict/documentTypeHint`)
  - output: in-memory structured extraction payload; optional persisted artifact when `persistArtifact=true`
- Per-record normalized shape:
  - `{ sourceFile, page, data, schemaSource?, schemaUsed?, confidence?, warnings? }`

### Data / Storage
- Default output is structured IPC payload (no mandatory file write).
- If persistence is enabled, use atomic write (temp file + rename) to avoid partial corruption.
- Output ordering must match deterministic processing order.

### Errors / Edge Cases
- Unsupported format: skip with `MEDIA_UNSUPPORTED_FORMAT` record-level error.
- Decode/render failure: record-level `MEDIA_DECODE_FAILED`, continue batch.
- Unsupported model: fail fast with `LLM_MULTIMODAL_NOT_SUPPORTED` before batch execution.
- Empty folder/no supported files: return actionable no-op result (not silent success).

### Test Plan
- Unit:
  - attachment payload normalization
  - recursive discovery and deterministic sorting
  - schema source selection (`provided` vs `generated`)
- Integration:
  - batch execution returns structured payload with required fields
  - optional persistence writes artifact atomically when enabled
  - strict schema violation blocks invalid record write
- E2E:
  - mixed images + PDFs fixture
  - unsupported-model deterministic error path
  - logs include required observability fields

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx` | **MOD**: ensure drag/drop supports file+folder metadata contract and deterministic attachment list handoff |
| `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx` | **MOD**: send structured attachments and optional batch extraction intent |
| `crewagent-runtime/electron/services/agentSessionContract.ts` | **MOD**: extend chat send payload contract for attachments |
| `crewagent-runtime/electron/main.ts` | **MOD**: orchestrate attachment ingestion, recursive discovery, structured result return, and optional artifact write |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | **REUSE**: reuse `media.extract` for per-file/page extraction and strict validation behavior |
| `crewagent-runtime/src/pages/RunsPage/components/ChatInput.test.tsx` | **MOD**: drag/drop payload tests for files/folders |

## Tasks / Subtasks

- [x] 1) Chat attachment payload hardening (AC: 1)
  - [x] 1.1 Normalize attachment metadata for files/folders
  - [x] 1.2 Preserve alias path semantics (`@project/...`) end-to-end
  - [x] 1.3 Reject unsupported roots/traversal inputs at ingestion time

- [x] 2) Recursive file discovery + deterministic scheduling (AC: 2)
  - [x] 2.1 Implement recursive enumeration for dropped folders
  - [x] 2.2 Filter supported formats only
  - [x] 2.3 Sort deterministically by relative path

- [x] 3) Schema contract orchestration (AC: 3,4,5)
  - [x] 3.1 Use provided schema when present (`schemaSource=provided`)
  - [x] 3.2 Generate schema from `schemaIntent` when schema missing (`schemaSource=generated`)
  - [x] 3.3 Enforce strict validation and return structured schema errors

- [x] 4) Batch result aggregation + optional persistence (AC: 6)
  - [x] 4.1 Aggregate record-level outputs (`sourceFile/page/data/...`)
  - [x] 4.2 Return structured payload via chat contract
  - [x] 4.3 Atomically persist artifact only when requested
  - [x] 4.4 Keep partial-success behavior and include failure records

- [ ] 5) Observability + E2E coverage (AC: 7,8)
  - [x] 5.1 Emit file/page-level audit fields and redact secrets
  - [ ] 5.2 Add mixed fixture integration tests
  - [ ] 5.3 Add unsupported-model deterministic path test

## Dev Notes

- Story MDE-1.3 depends on MDE-1.1 and MDE-1.2 foundations already in place:
  - multimodal model capability guard
  - first-class `media.extract` + binary fallback policy
- Keep extraction orchestration in main process; renderer remains input/output UX.
- Maintain existing sandbox boundaries (`@project/@pkg/@state`) and avoid bypass paths.

### Project Structure Notes

- Prefer reusing existing attachment flow from Story 5-17 rather than introducing parallel channels.
- Keep persisted artifact naming stable when persistence is explicitly enabled.
- Do not introduce provider-specific branching into renderer components.

### References

- Epic: `_bmad-output/epics-runtime-multimodal-data-extraction.md` (Story MDE-1.3)
- PRD Addendum: `_bmad-output/prd-runtime-multimodal-data-extraction.md`
- Architecture Addendum: `_bmad-output/architecture/runtime-multimodal-data-extraction-architecture.md`
- Parent PRD FR mapping: `_bmad-output/prd.md` (`FR-MULTI-01..04`)
- Dependency Story: `_bmad-output/implementation-artifacts/mde-1-1-multimodal-model-configuration-capability-guard.md`
- Dependency Story: `_bmad-output/implementation-artifacts/mde-1-2-first-class-media-extract-binary-fallback-policy.md`
- Design: `_bmad-output/implementation-artifacts/design-mde-1-3-drag-and-drop-batch-extraction-schema-contract-observability.md`
- Validation: `_bmad-output/implementation-artifacts/validation-report-story-mde-1-3.md`

## Change Log

- 2026-02-28: Created story draft for MDE-1.3 (ready-for-design).
- 2026-02-28: Added design artifact and promoted status to ready-for-dev.
- 2026-02-28: Added validate-create-story report (ready-for-dev confirmed).
- 2026-02-28: Implemented dev-story core changes (chat attachment contract, recursive batch extraction orchestration, atomic artifact output, and observability logs); moved to review.
- 2026-02-28: Fixed review findings: stop now aborts extraction stage, PDF multi-page records no longer default to `page=1`, and added backend utility tests.
- 2026-02-28: Updated contract direction: drag-and-drop no longer implies mandatory extraction; default output is structured return with optional persistence.

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `npm -C crewagent-runtime run test -- src/pages/RunsPage/components/ChatInput.test.tsx`
- `npm -C crewagent-runtime run build:ci`

### Completion Notes List
- Extended chat attachment contract to include `attachments[]` and `batchExtraction` options.
- Updated drag-and-drop source/target so folders are first-class attachments with normalized metadata.
- Added main-process batch orchestration for recursive discovery, deterministic sorting, multimodal capability fail-fast, and per-file `media.extract` execution.
- Added structured batch result contract (`records/errors/stats`) as primary output; artifact persistence is optional.
- Added file/page-level observability logs with `runId/sourceFile/page/model/status/duration/errorCode`, without secret material.
- Added a renderer test case for directory attachment rendering.
- Added stop-signal propagation for extraction stage (`chat:stop` aborts active batch extraction).
- Adjusted page semantics: PDF records default to `page=0` when no explicit page is returned; read-mode page arrays are expanded into per-page records.
- Added backend utility tests for attachment normalization, media support filtering, deterministic sorting, and page derivation.

### File List
- `crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx`
- `crewagent-runtime/src/pages/RunsPage/components/ChatInput.test.tsx`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/electron/services/agentSessionContract.ts`
- `crewagent-runtime/electron/services/chatBatchExtractionUtils.ts`
- `crewagent-runtime/electron/services/chatBatchExtractionUtils.test.ts`
- `crewagent-runtime/electron/main.ts`
- `_bmad-output/implementation-artifacts/mde-1-3-drag-and-drop-batch-extraction-schema-contract-observability.md`

## Dependencies

- Epic MDE-1
- Story MDE-1.1 (multimodal capability guard)
- Story MDE-1.2 (first-class `media.extract` + binary fallback)
- Existing drag/drop attachment base flow (Story 5-17)

## Verification Plan

1. Drag files/folders in chat and verify backend receives normalized attachment payload.
2. Run recursive ingestion and verify deterministic file ordering.
3. Validate `schema` provided/generated paths and `schemaSource` values.
4. Enable strict mode and verify invalid outputs are rejected with `MEDIA_SCHEMA_INVALID`.
5. Verify structured response contains required fields and stable ordering; if persistence is enabled, verify artifact path and content.
6. Run unsupported-model scenario and verify deterministic `LLM_MULTIMODAL_NOT_SUPPORTED`.
