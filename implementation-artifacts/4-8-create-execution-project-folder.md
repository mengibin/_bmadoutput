# Story 4.8: Create Run Folder (RuntimeStore) & Initialize State

Status: review

## Story

As a **Consumer**,
I want a new run folder created for each execution,
so that each run has isolated state/logs while writing artifacts into my ProjectRoot.

## Acceptance Criteria

1. **Run Folder Creation**
   - **Given** I click "Start Execution" (or API call to `RunManager.createRun`)
   - **Then** a new Run Folder is created under RuntimeStore (derive `projectId` from `projectRoot`):
     - Path: `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/`
     - Directories: `state/` and `state/logs/` (created)
     - File: `state/logs/execution.jsonl` exists (empty if new)

2. **Workflow Resolution (Multi-Workflow)**
   - **Given** `bmad.json.workflows[]` exists
   - **Then** `workflowRef` must match a workflow `id` and its `workflow`/`graph` paths are used.
   - **Given** `bmad.json.workflows[]` is missing/empty
   - **Then** use `bmad.json.entry.workflow` and `bmad.json.entry.graph` (single-workflow fallback).

3. **State Initialization (`@state/workflow.md`)**
   - **Then** `<RuntimeStoreRoot>/.../<runId>/state/workflow.md` is created.
   - **Content**: Copied from the selected workflow's `workflow.md`, then Frontmatter is updated/initialized with:
     - `runId`: The generated UUID.
     - `activeAgentId`: The selected agent ID.
     - `workflowRef`: The selected workflow ID.
     - `updatedAt`: ISO timestamp.
     - `currentNodeId`: (Empty or entry node, depends on Story 4.3 logic, usually empty at creation).
     - `stepsCompleted`: `[]` (when missing).
     - `variables`: `{}` (when missing).
     - `decisionLog`: `[]` (when missing).
     - `artifacts`: `[]` (when missing).
   - **And** the resulting frontmatter is valid against `workflow-frontmatter.schema.json` (v1.1).

4. **Runs Index Update**
   - **Then** the Project-scoped metadata is updated at:
     - `<RuntimeStoreRoot>/projects/<projectId>/runsIndex.json`
   - **Entry schema**:
     - `{ runId, projectId, packageId, workflowRef, activeAgentId, phase, createdAt, lastUpdatedAt }`
   - **Notes**:
     - `createdAt` is the run start time (equivalent to `startedAt` in Epic text).
     - `phase` uses: `idle | running | waiting-user | completed | failed`.

5. **Path Mapping (Definition)**
   - The Run structure MUST support the following mounts for Story 4.6:
     - `@state` -> `<RuntimeStoreRoot>/.../<runId>/state/`
     - `@pkg` -> `<RuntimeStoreRoot>/packages/<packageId>/` (ReadOnly)
     - `@project` -> The user's actual ProjectRoot.
   - `@project/artifacts/` is created if missing.
   - `@state` file operations require a runId context (explicit parameter or active-run binding); missing context must be rejected.

6. **Validation**
   - If `projectRoot` or `packageId` are invalid/missing, `createRun` fails.
   - If `bmad.json.workflows[]` exists:
     - `workflowRef` is required and must match a workflow `id`, otherwise `createRun` fails.
   - If `bmad.json.workflows[]` is missing/empty:
     - `workflowRef` may be empty; use `entry.workflow/entry.graph`.
     - If `workflowRef` is provided, it must match the derived entry workflow id (from `entry.workflow`), otherwise `createRun` fails.
   - If the resolved `workflow.md` path does not exist, `createRun` fails.
   - If the resolved `workflow.graph.json` path does not exist, `createRun` fails.
   - If `activeAgentId` is missing or not found in `agents.json`, `createRun` fails.

## Technical Context

- **Module**: `crewagent-runtime/electron/services/runManager.ts` (New Service)
- **Dependencies**:
  - `RuntimeStore` (to derive projectId from projectRoot, ensure artifacts dir, manage `runsIndex`).
  - `PackageManager` (to read `bmad.json` and the selected workflow `workflow.md` from the cached package).
- **Architecture**: `_bmad-output/architecture/runtime-architecture.md` (Section 3: Project / Package / Run & Storage Layout).

## Design

### Summary
- Create a run-scoped state/logs folder under RuntimeStore: `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/state/`.
- Resolve `workflowRef` against `bmad.json.workflows[]` (strict) with fallback to `entry.workflow/entry.graph` when no index.
- Initialize `@state/workflow.md` from the selected workflow `workflow.md` and inject `runId/activeAgentId/workflowRef/updatedAt` while resetting progress fields.
- Append a run record into `<RuntimeStoreRoot>/projects/<projectId>/runsIndex.json` atomically, and expose `runs:create` / `runs:list` via IPC.
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-8-create-execution-project-folder.md`

### UX / UI
- Package Page “Start Workflow / Start Agent” must call IPC `runs:create` (do not generate `runId` in Renderer).
- On success: update `appStore.activeRun` from returned metadata and navigate to `/runs`.
- On failure: show `error.message` and keep `activeRun` unchanged.

### API / Contracts
- `RunConfig`: `{ projectRoot: string; packageId: string; workflowRef?: string; activeAgentId: string }`
- `RunMetadata`: `{ runId; projectId; packageId; workflowRef; activeAgentId; phase; createdAt; lastUpdatedAt }`
  - `phase`: `'idle' | 'running' | 'waiting-user' | 'completed' | 'failed'`
- IPC
  - `runs:create` → `{ success: true, run: RunMetadata } | { success: false, error: { code, message, details? } }`
  - `runs:list` → `{ success: true, runs: RunMetadata[] } | { success: false, error: { code, message, details? } }`
- `RunManager.getRunStatePath(projectRoot, runId)` returns an absolute path used for mounting `@state` (never exposed to LLM).

### Data / Storage
- Run folders:
  - `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/state/workflow.md`
  - `<RuntimeStoreRoot>/projects/<projectId>/runs/<runId>/state/logs/execution.jsonl`
- Run index:
  - `<RuntimeStoreRoot>/projects/<projectId>/runsIndex.json` (append new entry; update via tmp → rename)
- Project artifacts:
  - Ensure `@project/artifacts/` exists before execution starts.

### Errors / Edge Cases
- `projectRoot` invalid/missing → fail fast.
- `packageId` not found → fail fast.
- `workflows[]` exists but `workflowRef` missing/unknown → fail fast.
- Resolved `workflow.md` or `workflow.graph.json` missing → fail fast.
- Workflow frontmatter parse/write failures → return structured error (no partial state).
- `runsIndex.json` update failures → return structured error (do not report run as created).

### Test Plan
- Unit (vitest): `RunManager.createRun`
  - Multi-workflow: requires `workflowRef`, selects correct workflow paths.
  - Single-workflow fallback: allows empty `workflowRef`; rejects mismatched provided `workflowRef`.
  - Creates run state dir and `state/logs/execution.jsonl`.
  - Writes `@state/workflow.md` with injected fields and reset `stepsCompleted/variables/decisionLog/artifacts`.
  - Appends to `runsIndex.json` atomically.
  - Rejects `activeAgentId` not found in `agents.json`.
- Optional IPC smoke: `runs:create`/`runs:list` return shape and error codes.

## Tasks / Subtasks

- [x] Implement `RunManager` service skeleton.
- [x] Implement `createRun` logic:
  - [x] Directory creation.
  - [x] Create `@state/logs/execution.jsonl` if missing.
  - [x] Resolve `workflowRef` via `bmad.json` (support `workflows[]` or fallback to `entry`).
  - [x] `runsIndex.json` update (atomic write).
  - [x] `@state/workflow.md` initialization (via `gray-matter`).
- [x] Implement `listRuns(projectRoot)` reading from `runsIndex.json`.
- [x] Wire IPC + preload (`runs:create`, `runs:list`).
- [x] Update UI run creation to call `runs:create` (Package + Works).
- [x] Add unit tests for `RunManager` (vitest).

## Verification

- **Unit Test**: `runManager.test.ts`
  - Call `createRun`.
  - Verify file system structure exists.
  - Verify `workflow.md` content has correct injected frontmatter.
  - Verify `runsIndex.json` has the new entry.
  - Verify `state/logs/execution.jsonl` is created.

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-8-create-execution-project-folder.md`
