# Story 4.9: Save All Artifacts to ProjectRoot (Project-First)

Status: in-progress

> **Tech Spec:** `_bmad-output/implementation-artifacts/tech-spec-4-9-save-all-artifacts-to-project-folder.md`

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **Runtime**,
I want all user-visible generated files to be saved within the user ProjectRoot by default,
so that outputs are organized and persisted.

## Acceptance Criteria

### AC-1: Default write location (Project-First)

**Given** the LLM generates a new file via `fs.write`  
**When** the runtime FileSystemToolHost writes the file  
**Then** the file is written under `@project/...` (ProjectRoot)  
**And** the default artifact output location is `@project/artifacts/...` (per system prompt + `artifactsRoot`)  
**And** parent directories are created if missing  
**And** writes remain sandboxed (no traversal outside ProjectRoot)

### AC-2: Record created artifacts in run state (`@state/workflow.md`)

**Given** a successful `fs.write` that creates a new file under `@project/artifacts/...`  
**When** the write completes  
**Then** the artifact path is appended to `@state/workflow.md` frontmatter `artifacts[]`  
**And** the recorded value is a **project-relative** path (e.g. `artifacts/foo.md`, i.e. strip the `@project/` prefix)  
**And** duplicates are not added

### AC-3: Artifact appears in execution log UI

**Given** an artifact file is created via `fs.write`  
**When** Execution Log UI renders the run log (`@state/logs/execution.jsonl`)  
**Then** the artifact write is visible in the tool call list (at minimum: the `fs.write` entry includes the written `@project/...` path)

## Technical Context

- **Primary module**: `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `fs.write` is the single tool used for creating files.
  - Tool execution is logged to `@state/logs/execution.jsonl`.
  - `fs.apply_patch` supports `updateFrontmatter` for `@state/workflow.md` (can be used internally to append `artifacts[]`).
- **Schema (artifact list)**: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
- **Reference docs**:
  - `_bmad-output/architecture/runtime-architecture.md` (Project-First + `artifacts: string[]` stores project-relative paths)
  - `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` (mounts + artifactsRoot)

## Design

### Summary

- Treat `@project/artifacts/...` as the default user-visible output directory.
- On successful `fs.write` that creates a new file under `@project/artifacts/...`:
  - Compute the project-relative path (e.g. `artifacts/foo.md`).
  - Append it to `@state/workflow.md` frontmatter `artifacts[]` (dedupe).
  - Prefer reusing the existing `updateFrontmatter` patch logic to preserve schema + transition validation.

### API / Contracts

- No tool API changes required (existing `fs.write` + `fs.apply_patch`).
- Artifact record format:
  - Store project-relative path strings inside `@state/workflow.md` frontmatter `artifacts: string[]`.

### Errors / Edge Cases

- If `@state/workflow.md` is missing or invalid, the tool host must return a structured error (do not silently drop artifact records).
- If the write succeeds but the state patch fails, treat the overall operation as failure (atomicity at the “artifact created + recorded” level).

### Test Plan

- Unit (vitest): `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
  - `fs.write` to `@project/artifacts/a.md` creates the file and appends `artifacts/a.md` to `@state/workflow.md` frontmatter.
  - Re-writing the same artifact does not duplicate `artifacts[]`.
  - `fs.write` outside artifacts root (e.g. `@project/src/a.ts`) does not append to `artifacts[]`.
  - Failure to patch `@state/workflow.md` returns an error and does not leave inconsistent state.

## Tasks / Subtasks

- [x] Implement artifact recording on `fs.write` (AC-2)
  - [x] Detect “new file created” under `@project/artifacts/...`
  - [x] Compute project-relative path (`artifacts/...`)
  - [x] Append into `@state/workflow.md` via `updateFrontmatter` (dedupe)
  - [x] Ensure failure modes are structured + consistent (no partial success)
- [x] Confirm execution log contains `fs.write` entries with `@project/...` path (AC-3)
- [x] Add vitest coverage for artifact recording behavior (AC-2/AC-3)

### Review Follow-ups (AI)

- [ ] [AI-Review][HIGH] Execution Log UI is not backed by `@state/logs/execution.jsonl` (UI reads in-memory runtime logs only), so AC-3 isn’t fully satisfied. [crewagent-runtime/src/hooks/useLogs.ts:1]
- [ ] [AI-Review][MEDIUM] Artifact recording hardcodes `@project/artifacts/` and ignores configurable `ProjectConfig.artifactsDir`, so custom artifacts roots won’t be tracked. [crewagent-runtime/electron/services/fileSystemToolHost.ts:1388]
- [ ] [AI-Review][LOW] Paths like `@project/./artifacts/a.md` won’t be recorded because the artifacts check uses `startsWith('@project/artifacts/')` without normalizing internal `./` segments. [crewagent-runtime/electron/services/fileSystemToolHost.ts:1388]

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- `npm -C crewagent-runtime test -- electron/services/fileSystemToolHost.test.ts`
- `npm -C crewagent-runtime test`
- `npm -C crewagent-runtime run lint` (fails: pre-existing `react-hooks/exhaustive-deps` warnings)

### Completion Notes List

- `fs.write` now records newly created `@project/artifacts/...` files into `@state/workflow.md` frontmatter `artifacts[]` as project-relative paths (e.g. `artifacts/a.md`).
- Recording uses the existing `updateFrontmatter` path (schema + transition validation) and dedupes entries.
- If state update fails, the newly created artifact file is rolled back (no partial success).
- Added tests for artifact recording, dedupe, rollback behavior, and execution log entry presence.
- Repo lint currently fails due to pre-existing React hooks dependency warnings (unrelated to this change).

### File List

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`

## Change Log

- 2026-01-08: Code review completed; 1 HIGH / 1 MEDIUM / 1 LOW issue logged; status set to in-progress.

## Senior Developer Review (AI)

### Review Outcome

- Date: 2026-01-08
- Outcome: Changes Requested
- Issues: 3 (1 HIGH, 1 MEDIUM, 1 LOW)
- Acceptance Criteria: PARTIAL (AC-3 not fully backed by JSONL in UI)

### Action Items

- [ ] [AI-Review][HIGH] Execution Log UI must read from `@state/logs/execution.jsonl` (or explicitly persist/reload logs from it) so AC-3 is truly “backed” by JSONL. [crewagent-runtime/src/hooks/useLogs.ts:1]
- [ ] [AI-Review][MEDIUM] Respect configurable artifacts root (`ProjectConfig.artifactsDir`) when deciding whether to append to `artifacts[]`. [crewagent-runtime/electron/services/fileSystemToolHost.ts:1388]
- [ ] [AI-Review][LOW] Normalize `@project` paths before artifact-root matching to handle `./` segments. [crewagent-runtime/electron/services/fileSystemToolHost.ts:1388]
