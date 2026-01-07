# Tech-Spec: Parse Frontmatter to Determine Current Node

**Created:** 2025-12-29
**Status:** Done
**Source Story:** _bmad-output/implementation-artifacts/4-3-parse-frontmatter-to-determine-current-node.md

## Overview

### Problem Statement
The Runtime needs to know the current state of a workflow execution (e.g. which step to run next, what variables are set) to resume or start a session. This state is stored in the YAML Frontmatter of the `workflow.md` file (for BMad v1.1). We need a reliable way to parse this.

### Solution
Integrate `gray-matter` into the Electron Main process (`RuntimeStore`) to parse the run-scoped `@state/workflow.md` file. Validate frontmatter against `schemas/workflow-frontmatter.schema.json` (v1.1), extract `currentNodeId/stepsCompleted/variables/decisionLog/artifacts`, and return a normalized `WorkflowState`. If `currentNodeId` is empty, default to `workflow.graph.json.entryNodeId`. If `currentNodeId` does not exist in the graph, block execution with a fix hint.

### Scope (In/Out)
- **In**: Parsing run state `@state/workflow.md`, schema validation, graph-based fallback/validation.
- **Out**: Modifying/Writing frontmatter (handled in Story 4.7), parsing Markdown body.

## Context for Development

### Codebase Patterns
- `electron/stores/runtimeStore.ts` handles file I/O.
- IPC is used to expose this to the frontend (though this story focuses on the backend logic, effectively supporting Story 5.1 View State).

### Files to Reference
- `spec/bmad-package-spec/v1.1/examples/create-story-micro/workflow.md` (Example format)

### Technical Decisions
- **Library**: `gray-matter` is the de-facto standard for parsing frontmatter in JS ecosystem. It is robust and handles YAML engine internally.
- **Validation**: Use `ajv/dist/2020` to validate `workflow-frontmatter.schema.json` and `workflow-graph.schema.json`.
- **Error Handling**: Missing/invalid frontmatter or invalid `currentNodeId` should return a clear, actionable error (block execution).

## Implementation Plan

### Tasks

- [x] **Setup**: Install `gray-matter` as dependency.
- [x] Define `WorkflowState` interface in `RuntimeStore.ts`:
    ```typescript
    interface WorkflowState {
        currentNodeId: string
        variables: Record<string, any>
        stepsCompleted: string[]
        // ... other fields
    }
    ```
- [x] Implement frontmatter parsing + normalization logic.
- [x] Implement `getWorkflowState(packageId: string, workflowId: string, runId: string, projectRoot: string)` in `RuntimeStore`.
    - Logic: Load `workflow.graph.json` (for entry node + validation) then read run state `runs/<runId>/state/workflow.md` and parse/validate.
- [x] **Test**: Unit tests for valid/invalid frontmatter.

### Acceptance Criteria

- [x] **Valid**: Parsing example `workflow.md` returns correct `currentNodeId`.
- [x] **Empty**: Empty `currentNodeId` falls back to `graph.entryNodeId`.
- [x] **Missing**: Missing `@state/workflow.md` returns a specific error and blocks execution.

## Additional Context

### Dependencies
- `gray-matter` (npm install gray-matter)
- `vitest`

### Testing Strategy
- **Fixture**: Use `spec/bmad-package-spec/v1.1/examples/create-story-micro`.
- **Unit Tests**: Create `electron/stores/runtimeStore.test.ts`.
- **Workflow**:
    1. Import the micro package (mock zip or direct path).
    2. Call `loadWorkflow`.
    3. Create a run-scoped `workflow.md` under `runs/<runId>/state/`.
    4. Call `getWorkflowState`.
    5. Assert `currentNodeId` is `step-01-select-story`.
