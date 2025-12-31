# Tech-Spec: Parse Frontmatter to Determine Current Node

**Created:** 2025-12-29
**Status:** Ready for Development
**Source Story:** _bmad-output/implementation-artifacts/4-3-parse-frontmatter-to-determine-current-node.md

## Overview

### Problem Statement
The Runtime needs to know the current state of a workflow execution (e.g. which step to run next, what variables are set) to resume or start a session. This state is stored in the YAML Frontmatter of the `workflow.md` file (for BMad v1.1). We need a reliable way to parse this.

### Solution
Integrate `gray-matter` into the Electron Main process (`RuntimeStore`) to parse the `workflow.md` file. Extract `currentNodeId` and `variables` and structure them into a `WorkflowState` object.

### Scope (In/Out)
- **In**: Parsing existing `workflow.md` files, extracting specific frontmatter fields.
- **Out**: Modifying/Writing frontmatter (handled in a future story), parsing Markdown body.

## Context for Development

### Codebase Patterns
- `electron/stores/runtimeStore.ts` handles file I/O.
- IPC is used to expose this to the frontend (though this story focuses on the backend logic, effectively supporting Story 5.1 View State).

### Files to Reference
- `spec/bmad-package-spec/v1.1/examples/create-story-micro/workflow.md` (Example format)

### Technical Decisions
- **Library**: `gray-matter` is the de-facto standard for parsing frontmatter in JS ecosystem. It is robust and handles YAML engine internally.
- **Error Handling**: If file is missing or empty, return default state (entry node from graph).

## Implementation Plan

### Tasks

- [ ] **Setup**: Install `gray-matter` as dependency.
- [ ] Define `WorkflowState` interface in `RuntimeStore.ts`:
    ```typescript
    interface WorkflowState {
        currentNodeId: string
        variables: Record<string, any>
        stepsCompleted: string[]
        // ... other fields
    }
    ```
- [ ] Implement `parseWorkflowState(content: string): WorkflowState` helper.
- [ ] Implement `getWorkflowState(packageId: string): Promise<WorkflowState | null>` in `RuntimeStore`.
    - Logic: Resolve `@pkg/workflow.md`, read, parse.
- [ ] **Test**: Unit tests for valid/invalid frontmatter.

### Acceptance Criteria

- [ ] **Valid**: Parsing example `workflow.md` returns correct `currentNodeId`.
- [ ] **Empty**: Parsing file without frontmatter returns safe defaults (or error if strictly required).
- [ ] **Missing**: Handling missing file gracefully (or throwing specific error).

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
    3. Call `getWorkflowState`.
    4. Assert `currentNodeId` is `step-01-select-story`.
