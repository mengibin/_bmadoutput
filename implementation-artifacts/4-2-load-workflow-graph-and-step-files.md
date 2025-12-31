# Story 4.2: Load Workflow Graph and Step Files

Status: ready-for-design

## Story

As a **Runtime System**,
I want to **load and parse the `workflow.graph.json` and referenced step files** for a specific workflow,
so that I have all necessary data in memory to start an execution session.

## Acceptance Criteria

1. **Given** a loaded package and a target `workflowId`
   **When** `RuntimeStore.loadWorkflow(packageId, workflowId)` is called
   **Then** the system locates the `workflow.graph.json` corresponding to that workflow ID.
   **And** parses the JSON content.

2. **Given** the parsed graph JSON
   **When** validating the structure
   **Then** it MUST verify:
   - `schemaVersion` is supported (v1.1).
   - `entryNodeId` exists in the `nodes` list.
   - All `nodes` have `id`, `type`, and `file` properties.
   - All `edges` connect valid `from`/`to` node IDs.

3. **Given** a valid graph structure
   **When** loading step content
   **Then** the system iterates through all `nodes` (regardless of type).
   **And** resolves the `file` path relative to the package root.
   **And** reads the content of each `.md` file.
   **And** fails with a clear error if any file is missing or unreadable.

4. **Given** successful loading
   **When** returning the result
   **Then** the method returns a `WorkflowDefinition` object containing:
   - The original graph structure.
   - A map of `nodeId` -> `StepContent` (raw markdown string).

## Technical Context

- **Schema**: `workflow-graph.schema.json` (v1.1) defines the structure.
- **Path Resolution**: Step files are relative paths in the zip/package structure. `RuntimeStore` knows the package root.
- **Memory**: The loaded definition is ephemeral (per session). It doesn't need to be persisted to disk again, but loaded into the active `Run` state in memory.

## Tasks / Subtasks

- [ ] Define `WorkflowDefinition` and `StepContent` interfaces in `RuntimeStore`.
- [ ] Implement `loadWorkflow(packageId, workflowId)` in `RuntimeStore`.
- [ ] Implement `validateGraphIntegrity` helper (nodes vs edges vs entry).
- [ ] Implement `loadStepFiles` helper (parallel read of .md files).
- [ ] Add error handling for "Graph not found", "Step file missing", "Invalid Schema".
- [ ] Expose via IPC `workflow:load`.

## References

- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-2-load-workflow-graph-and-step-files.md

## Design

### Summary
- Implementation follows the Tech Spec.
- Key logical separation: `RuntimeStore` handles data loading; Execution Engine (future story) handles logic.
- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-2-load-workflow-graph-and-step-files.md

### UX / UI
- N/A (Backend logic).

### API / Contracts
- IPC Channel: `workflow:load`
- Returns: `WorkflowDefinition`

### Data / Storage
- Read-only access to package files.
- In-memory representation of `WorkflowDefinition`.

### Errors / Edge Cases
- Invalid Schema -> Error with details.
- Missing File -> Error "File not found: <path>".
- Invalid JSON -> Error "Parse error".

### Test Plan
### Automated Tests
- **Fixture**: `create-story-micro` (v1.1 reference package).
- **Unit Test**: `electron/stores/runtimeStore.test.ts` > `should load workflow graph and steps`.
- **Assertions**:
  - `loadWorkflow` returns `success: true`.
  - `definition.graph.entryNodeId` matches fixture (`step-01-select-story`).
  - `definition.steps` contains content for `step-01-select-story`.

### Manual Verification
1. Start the app.
2. Import `create-story-micro` zip.
3. Open Developer Tools console.
4. Verify `window.ipcRenderer.invoke('package:loadWorkflow', ...)` returns valid graph object.
