# Story MDE-1.2: First-Class `media.extract` + Binary Fallback Policy

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **System**,  
I want `media.extract` exposed directly in tool registry and `fs.read` constrained to fallback behavior for binaries,  
So that LLM follows the direct multimodal path safely.

## Acceptance Criteria

1. **Tool registry exposure**
   - **Given** a chat/agent run with tool access
   - **When** LLM requests available tools
   - **Then** `media.extract` appears in tool definitions
   - **And** includes args `path/mode/instruction/schema/schemaIntent/page/strict/documentTypeHint`.

2. **Direct extraction contract**
   - **Given** an image or PDF path
   - **When** `media.extract` is called
   - **Then** tool returns mode-aware result (`mode=extract` returns `data`; `mode=read` returns `content`)
   - **And** output contains `schemaSource` and `schemaUsed` when applicable in `mode=extract`.

3. **Binary fallback behavior for `fs.read`**
   - **Given** `fs.read` is called on binary/image/PDF
   - **When** tool processes request
   - **Then** it does not return garbled text
   - **And** returns structured hint metadata pointing to `media.extract`.

4. **Capability guard reuse**
   - **Given** `media.extract` is invoked
   - **When** configured multimodal model is unsupported
   - **Then** runtime returns `LLM_MULTIMODAL_NOT_SUPPORTED`
   - **And** no provider extraction request is sent.

5. **Backward compatibility**
   - **Given** existing text-file workflows using `fs.read`
   - **When** MDE-1.2 is introduced
   - **Then** text read behavior remains unchanged
   - **And** non-multimodal tool paths are not regressed.

## Design

> Required before development. Run `design-story MDE-1.2` and then set status to `ready-for-dev`.

### UX / UI
- N/A (backend/tool contract focused story).
- If fallback hint is surfaced to UI, keep message concise and action-oriented.

### API / Contracts
- Add first-class tool definition:
  - `name`: `media.extract`
  - `args`: `path/mode/instruction/schema/schemaIntent/page/strict/documentTypeHint`
  - rules:
    - `mode=extract` (default) requires `instruction`.
    - `mode=read` allows empty `instruction` and returns readable full content.
    - `mode=read` ignores `page` and emits warning for compatibility.
- Standardize result envelope:
  - success (`mode=extract`): `ok/mode/mime/path/page/data/schemaUsed/schemaSource/confidence/warnings`
  - success (`mode=read`): `ok/mode/mime/path/page/content/source/pages/confidence/warnings`
  - error: `{ code, message, details? }`

### Runtime Extraction Strategy
- `mode=extract` (default): structured extraction pipeline.
- `mode=read`: human-readable content pipeline for image/PDF.
- PDF handling policy:
  - Prefer local PDF text-layer parsing first (`pdf_text` source) via bundled Python (`pypdf`) for `mode=read`.
  - If text-layer is unavailable, render PDF pages to images via bundled Python (`pypdfium2`) and continue with multimodal fallback (`mixed`/`vision` source).
  - PDF extraction/read uses `image_url` payloads for rendered pages; direct `application/pdf` `input_file` mode is disabled.
  - All embedded-Python helper calls are async (non-`spawnSync`) to avoid Electron main-process blocking.

### Data / Storage
- No required persistent schema changes.
- Optional run artifact/log entries should include source path/page and chosen schema source.

### Errors / Edge Cases
- Unsupported format: `MEDIA_UNSUPPORTED_FORMAT`
- Decode failure: `MEDIA_DECODE_FAILED`
- Invalid schema: `MEDIA_SCHEMA_INVALID`
- Provider failure: `MEDIA_EXTRACTION_FAILED`
- Unsupported multimodal model: `LLM_MULTIMODAL_NOT_SUPPORTED`

### Test Plan
- Unit:
  - `media.extract` appears in visible tools with expected schema.
  - `fs.read` on binary returns structured hint (no garbled payload).
- Integration:
  - `media.extract` with supported model returns structured extraction envelope.
  - unsupported model path is blocked by capability guard.
- Regression:
  - text `fs.read` remains unchanged.

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | **MOD**: expose/dispatch first-class `media.extract`; add `mode=read` flow; implement PDF text-first + image fallback; keep `fs.read` binary fallback policy |
| `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` | **MOD**: add read-mode contract tests, PDF local text-read tests, and PDF compatibility fallback assertions |
| `crewagent-runtime/electron/services/llmAdapter.ts` | **MOD**: normalize tool-name dot/hyphen compatibility in response decoding to avoid `media-extract` mismatch on OpenAI-compatible providers |
| `crewagent-runtime/electron/services/llmAdapter.test.ts` | **MOD**: add non-stream + stream regression tests for `media-extract` → `media.extract` decoding |
| `crewagent-runtime/electron/services/toolHost.ts` | **MOD**: extend media result typing to include read-mode envelope (`content/source/pages`) |
| `crewagent-runtime/electron/services/multimodalCapabilityService.ts` | **REUSE**: keep guard as pre-flight gate from MDE-1.1 |
| `crewagent-runtime/electron/services/pythonService.ts` | **REUSE**: use bundled Python locator (`getBundledPythonPath`) for PDF helper execution |

## Tasks / Subtasks

- [x] 1) Expose first-class `media.extract` in tool registry (AC: 1)
  - [x] 1.1 Add function definition schema in visible tool set
  - [x] 1.2 Ensure tool policy gating aligns with existing tool enablement controls

- [x] 2) Implement `media.extract` execution path (AC: 2,4)
  - [x] 2.1 Parse/validate args (`path/mode/instruction/schema/schemaIntent/page/strict/documentTypeHint`)
  - [x] 2.2 Reuse MDE-1.1 capability guard before provider request
  - [x] 2.3 Return normalized success/error envelope with `schemaSource/schemaUsed`
  - [x] 2.4 Add embedded-Python dependency diagnostics (`PYTHON_MODULE_MISSING` surfaces missing module + pip package hint)

- [x] 3) Enforce binary fallback policy in `fs.read` (AC: 3,5)
  - [x] 3.1 Detect binary/image/PDF input deterministically
  - [x] 3.2 Return structured hint metadata instead of raw/garbled content
  - [x] 3.3 Keep text-read behavior unchanged

- [x] 4) Harden tool-name compatibility across providers (AC: 1,2)
  - [x] 4.1 Decode tool names by request-time lookup map (not hardcoded prefixes only)
  - [x] 4.2 Accept known hyphen aliases in dispatch (`media-extract` → `media.extract`) as runtime safety net

- [x] 5) Tests and regression safety net (AC: 1~5)
  - [x] 5.1 Unit tests for tool exposure and argument validation
  - [x] 5.2 Integration tests for extraction happy path + unsupported-model path
  - [x] 5.3 Regression tests for text `fs.read`, tool-name alias decoding, and non-multimodal tool flows

## Dev Notes

- This story is the implementation counterpart of architecture decisions:
  - AD-MULTI-01 (`media.extract` first-class)
  - AD-MULTI-02 (`fs.read` binary fallback-only)
- Keep schema generation orchestration minimal here; batch flow and drag-and-drop orchestration are completed in MDE-1.3.
- Preserve stable error shape used across runtime tool paths: `{ code, message, details? }`.

### Project Structure Notes

- Main-process tool host remains the single execution entry for `media.extract`.
- Do not move multimodal provider-specific logic into renderer.
- Preserve existing mount sandbox constraints (`@project/@pkg/@state`).

### References

- Epic: `_bmad-output/epics-runtime-multimodal-data-extraction.md` (Story MDE-1.2)
- PRD Addendum: `_bmad-output/prd-runtime-multimodal-data-extraction.md`
- Architecture Addendum: `_bmad-output/architecture/runtime-multimodal-data-extraction-architecture.md`
- Parent PRD FR mapping: `_bmad-output/prd.md` (`FR-MULTI-01..04`)
- Dependency Story: `_bmad-output/implementation-artifacts/mde-1-1-multimodal-model-configuration-capability-guard.md`
- Design: `_bmad-output/implementation-artifacts/design-mde-1-2-first-class-media-extract-binary-fallback-policy.md`
- Validation: `_bmad-output/implementation-artifacts/validation-report-story-mde-1-2.md`
- Code Review: `_bmad-output/implementation-artifacts/code-review-mde-1-2-first-class-media-extract-binary-fallback-policy.md`

## Change Log

- 2026-02-27: Created story draft for MDE-1.2 (ready-for-design).
- 2026-02-27: Added design artifact and promoted status to ready-for-dev.
- 2026-02-27: Added validate-create-story report (ready-for-dev confirmed).
- 2026-02-27: Implemented dev-story changes and moved story to review.
- 2026-02-27: Added code-review report with MEDIUM/LOW follow-up items.
- 2026-02-27: Fixed code-review follow-up items and refreshed review report (no open issues).
- 2026-02-27: Extended implementation with `media.extract mode=read` and PDF text-first/image-fallback strategy.
- 2026-02-27: Switched PDF helper execution to bundled Python async subprocess, read-mode full-content behavior, and disabled PDF `input_file` payloads.
- 2026-02-27: Applied follow-up fixes from second code review (abort semantics, read-mode full-page PDF rendering fallback, fs.read FD-safe binary sniff).
- 2026-02-28: Added OpenAI-compatible tool-name compatibility hardening (`media-extract` decode + dispatch alias safety net) to avoid misrouting away from `media.extract`.
- 2026-02-28: Improved PDF render failure diagnostics: missing embedded Python module now reports concrete dependency (`module/pipPackage/suggestion`) instead of generic failure.

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `npm -C crewagent-runtime run test -- electron/services/fileSystemToolHost.test.ts`
- `npm -C crewagent-runtime run test -- electron/services/llmAdapter.test.ts`
- `npm -C crewagent-runtime run test -- electron/services/multimodalCapabilityService.test.ts electron/services/fileSystemToolHost.test.ts electron/stores/runtimeStore.test.ts`
- `npm -C crewagent-runtime run build:ci`

### Completion Notes List
- Exposed `media.extract` as first-class tool in runtime tool registry with full argument schema.
- Implemented `media.extract` execution path with path validation, capability guard reuse, provider request, and normalized structured output.
- Added schema handling (`schema` provided / `schemaIntent` generated) and strict validation failure path (`MEDIA_SCHEMA_INVALID`).
- Added deterministic `fs.read` binary fallback (`FS_READ_BINARY_FALLBACK`) with structured hint metadata pointing to `media.extract`.
- Added/updated unit + integration coverage for unsupported model, successful extraction envelope, tool visibility, and binary fallback.
- Addressed code-review findings: provider-aware PDF handling, structured response-format compatibility retry, non-media binary hint branching, and missing branch tests.
- Added `media.extract mode=read` to support plain reading for media files, with PDF text-layer first and rendered-image fallback.
- `mode=read` now returns full readable content and ignores `page` argument with explicit warning.
- PDF conversion/extraction helper uses bundled Python asynchronously (`pypdf`/`pypdfium2`) to avoid blocking runtime.
- PDF requests no longer send `input_file`; provider payload uses rendered page `image_url` inputs only.
- Embedded Python helper now returns explicit aborted error semantics when run signal is cancelled.
- `mode=read` PDF image fallback now renders all pages (not first N pages) when local text layer extraction is unavailable.
- `fs.read` binary detection now guarantees file descriptor close via `try/finally`.
- Fixed OpenAI-compatible tool-name decoding for dotted tools (e.g. `media.extract`) in both non-stream and stream responses.
- Added runtime dispatch alias normalization so `media-extract` is safely routed to `media.extract`.
- PDF render dependency errors now surface actionable diagnostics (`missing module`, `pipPackage`, install suggestion) for faster environment repair.

### File List
- `_bmad-output/implementation-artifacts/mde-1-2-first-class-media-extract-binary-fallback-policy.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/llmAdapter.ts`
- `crewagent-runtime/electron/services/llmAdapter.test.ts`
- `crewagent-runtime/electron/services/toolHost.ts`

## Dependencies

- Epic MDE-1
- Story MDE-1.1 (capability guard baseline)
- Existing runtime tool host and LLM adapter infrastructure (Story 4-5 / 4-6 / 4-10)

## Verification Plan

1. Confirm `media.extract` appears in tool list and can be selected by LLM directly.
2. Run extraction on image/PDF and verify normalized structured output.
3. Trigger unsupported-model configuration and verify `LLM_MULTIMODAL_NOT_SUPPORTED`.
4. Call `fs.read` on binary file and verify hint metadata instead of garbled text.
5. Re-run text-file `fs.read` scenarios to ensure no regressions.
6. Verify OpenAI-compatible tool-calls with hyphenated names (`media-extract`) are decoded/routed to `media.extract`.
7. Simulate missing `pypdfium2` and verify `MEDIA_DECODE_FAILED` details include module + pip installation hint.
