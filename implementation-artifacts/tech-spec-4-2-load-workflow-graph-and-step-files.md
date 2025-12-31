# Tech-Spec: Load Workflow Graph and Step Files

**Created:** 2025-12-29
**Status:** Ready for Development
**Source Story:** _bmad-output/implementation-artifacts/4-2-load-workflow-graph-and-step-files.md

## Overview

### Problem Statement
The Runtime needs to transition from package metadata (Story 4.1) to actual execution readiness. This requires parsing the `workflow.graph.json` to understand the workflow structure and loading the content of all step files (Markdown) into memory so the Execution Engine can process them.

### Solution
Implement `RuntimeStore.loadWorkflow(packageId, workflowId)` which:
1. Locates the graph file for the given workflow.
2. Validates the graph structure against the v1.1 schema using `ajv`.
3. Performs logical validation (e.g. entryNodeId checks).
4. Iterates through the graph nodes to resolve and load referenced `.md` step files.
5. Returns a comprehensive `WorkflowDefinition` object.

### Scope (In/Out)
- **In**: Graph parsing, schema validation, step file loading (read-only), returning definition object.
- **Out**: Execution logic, variable interpolation, frontend visualization (other than verification).

## Context for Development

### Codebase Patterns
- **Store**: `RuntimeStore` handles backend logic and file system access.
- **Validation**: Uses `ajv` (already set up for packages) to validate JSON schemas.
- **Safety**: Uses `resolveInsideDir` to prevent path traversal when reading package files.

### Files to Reference
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
- `crewagent-runtime/electron/stores/runtimeStore.ts`

### Technical Decisions
- **Validation Library**: Use `ajv/dist/2020` consistent with Story 4.1.
- **Return Type**: `WorkflowDefinition { graph: WorkflowGraph, steps: Record<NodeId, Content> }`. This separates structure from content but keeps them bundled for the execution engine.
- **Concurrent Loading**: Use `Promise.all` or synchronous `fs` (since Electron Main is Node)? `fs` synchronous is safer for atomicity in simplified MVP, but async is better for UI responsiveness. Given `importPackage` was async, `loadWorkflow` should also be async.

## Implementation Plan

### Tasks

- [ ] **Setup**: Install `vitest` and configure `vite.config.ts` for testing.
- [ ] Define `WorkflowDefinition` interface in `RuntimeStore.ts`.
- [ ] Implement `validateWorkflowGraph` private method using `ajv`.
- [ ] Implement `loadWorkflow(packageId, workflowId)` in `RuntimeStore`.
- [ ] **Test**: Create `electron/stores/runtimeStore.test.ts` to unit test `loadWorkflow` (mocking fs/adm-zip if needed, or using integration style with temp files).
- [ ] Expose `workflow:load` via IPC.
- [ ] Add temporary logging in `PackagePage.tsx` to verify loading.

### Acceptance Criteria

- [ ] **Unit Tests Pass**: `npm run test` passes for `RuntimeStore`.
- [ ] **Valid**: Loading `create-story-micro` works and returns graph + all step contents.
- [ ] **Invalid Schema**: Loading a graph with missing `entryNodeId` fails.
- [ ] **Missing File**: Loading a graph referencing a non-existent `.md` file fails.

## Additional Context

### Dependencies
- `ajv`, `ajv-formats` (installed).
- `vitest` (new devDependency).

### Testing Strategy
- **Automated**: Unit tests for `RuntimeStore` logic (Validation, Graph Parsing, File Resolution).
- **Manual**: Verification using the provided `create-story-micro` package in the running app.
- Corrupting the package to test failure modes manually.

## Traceability

- Story: _bmad-output/implementation-artifacts/4-2-load-workflow-graph-and-step-files.md
- Design: N/A (Backend logic only)
