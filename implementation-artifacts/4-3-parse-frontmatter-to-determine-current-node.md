# Story 4.3: Parse Frontmatter to Determine Current Node

Status: ready-for-design

## Story

As a **Runtime Execution Engine**,
I want to **parse the Frontmatter of the active `workflow.md` file**,
so that I can determine the **Current Node ID** and the current state of execution (variables, history).

## Acceptance Criteria

1. **Given** a reference to the active `workflow.md` file (entry point)
   **When** the execution session initializes or resumes
   **Then** the system reads the file content.

2. **When** parsing the content
   **Then** it extracts the YAML Frontmatter (between `---` delimiters).
   **And** identifies the `currentNodeId` property.

3. **Given** a missing or empty `currentNodeId`
   **When** interpreting state
   **Then** it defaults to the `entryNodeId` defined in the Graph (loaded in Story 4.2).

4. **When** parsing is complete
   **Then** the `RuntimeStore` (or `Run` object) updates its state with:
   - `currentNodeId`
   - `variables` map (if present)
   - `stepsCompleted` list (if present)

## Technical Context

- **File**: The `workflow.md` file is the "living" document of the execution.
- **Library**: Use `gray-matter` or similar frontmatter parser (or regex if simple).
- **Fallback**: If `currentNodeId` is missing in `workflow.md` (e.g. fresh start), use Graph's entry node.

- [ ] Verify unit tests for extracting `currentNodeId`.

## References

- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-3-parse-frontmatter-to-determine-current-node.md

## Design

### Summary
- Use `gray-matter` to parse `workflow.md`.
- Implement `getWorkflowState` in `RuntimeStore`.
- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-3-parse-frontmatter-to-determine-current-node.md

### UX / UI
- N/A (Backend logic).

### API / Contracts
- IPC Channel: `workflow:getState` (Future/Implicit)
- Returns: `WorkflowState`

### Data / Storage
- Read-only access to `workflow.md` (for this story).

### Errors / Edge Cases
- Invalid YAML -> Parse Error.
- Missing Field -> Default/Partial Object.

### Test Plan
- **Automated Test Fixture**: MUST use the actual `spec/bmad-package-spec/v1.1/examples/create-story-micro` package files (zipped or direct) as the test case.
- **Unit Tests** (`electron/stores/runtimeStore.test.ts`):
  - `RuntimeStore.loadWorkflow`: Load `create-story-micro` and verify Graph structure.
  - `RuntimeStore.getWorkflowState`: Parse `create-story-micro/workflow.md` and verify `currentNodeId` equals `step-01-select-story`.
- **Frontmatter Verification**:
  - Check `currentNodeId` == `step-01-select-story`.
  - Check `variables` is object.
  - Check `stepsCompleted` is array.


