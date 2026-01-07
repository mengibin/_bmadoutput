# Story 4.7: Validate Graph Transition & Update Frontmatter

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime**,
I want to validate state transitions against the workflow graph and persist state in `@state/workflow.md`,
so that execution is correct, auditable, and recoverable.

## Acceptance Criteria

1. **Frontmatter Parse + Schema Validation**
   - **Given** the LLM attempts to update `@state/workflow.md` (typically via `fs.apply_patch`)
   - **When** the Runtime prepares to persist the change
   - **Then** the Runtime validates:
     - Frontmatter YAML is parseable (no YAML/frontmatter parse errors)
     - Frontmatter matches `schemas/workflow-frontmatter.schema.json` (v1.1) after normalization

2. **`currentNodeId` Existence & Transition Guard**
   - **Given** the selected workflow graph (from the workflow record / `RUN_DIRECTIVE.graph`)
   - **And** a previous state `currentNodeId_oldRaw` (raw string; empty allowed in file)
   - **When** the new frontmatter sets `currentNodeId_newRaw`
   - **Then** the Runtime derives effective node ids:
     - `currentNodeId_oldEffective = trim(currentNodeId_oldRaw) || graph.entryNodeId`
     - `currentNodeId_newEffective = trim(currentNodeId_newRaw) || graph.entryNodeId`
   - **And** the Runtime validates:
     - `currentNodeId_newRaw` is a string (empty string allowed)
     - `currentNodeId_newEffective` exists in the graph
     - If `currentNodeId_newEffective !== currentNodeId_oldEffective`, the transition MUST be allowed by an outgoing edge:
       - allowed if `graph.edges` contains `{ from: currentNodeId_oldEffective, to: currentNodeId_newEffective }`

3. **`stepsCompleted` Must Not Regress**
   - **Given** `stepsCompleted_old` and `stepsCompleted_new`
   - **Then** `stepsCompleted_new` MUST be a superset of `stepsCompleted_old` (no removals).

4. **Rejection UX (LLM Feedback)**
   - **Given** an invalid update
   - **When** the Runtime rejects it
   - **Then** the tool result returns a structured error:
     - `code = E_INVALID_TRANSITION` (for transition violations) or `E_SCHEMA_VALIDATION` (for schema violations)
     - `message` includes the attempted `from → to`
     - `details.allowedNext` lists allowed next nodes from `currentNodeId_oldEffective` (include `to/label/isDefault/conditionText` when available)

5. **Atomic Persist + Recoverability**
   - **Given** a valid update
   - **Then** the Runtime saves `@state/workflow.md` atomically (tmp → rename) and updates `sha256After`.
   - **And** restoring a run requires only `@state/workflow.md` + the selected graph under `@pkg/...` (no chat history dependency).

## Technical Context

- **Runtime state file**: `@state/workflow.md` (Document-as-State).
- **Graph source of truth**: workflow graph referenced by the selected workflow record (mounted under `@pkg/...`).
- **Schema**:
  - `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
  - `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
- **Primary implementation points**:
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts` (`fs.apply_patch`, `fs.write`)
  - `crewagent-runtime/electron/services/executionEngine.ts` (passes context for validation; reacts to state updates)
  - `crewagent-runtime/electron/stores/runtimeStore.ts` (already has AJV validators + `getWorkflowState` normalization rules)
- **Related stories**:
  - Story 4.3 (parse/validate frontmatter on read)
  - Story 4.6 (filesystem tool host + atomic writes)

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-7-validate-graph-transition-and-update-frontmatter.md`
- Treat `@state/workflow.md` as a guarded state file: validate YAML parseability, frontmatter schema, graph transition legality, and `stepsCompleted` monotonicity **before** persisting.
- Add workflow-graph resolution to ToolHost execution context (`workflowId` or `graphRelPath`) so validation uses the selected `RUN_DIRECTIVE.graph` (no hardcoded `@pkg/workflow.graph.json`).
- On invalid updates, reject with actionable ToolErrors (`E_SCHEMA_VALIDATION` / `E_INVALID_TRANSITION`) and include `details.allowedNext` so the LLM can self-correct.

### UX / UI

- N/A (backend-only). UI should surface the returned ToolError message and (for transitions) `details.allowedNext`.

### API / Contracts

- ToolHost execution context must include workflow identity for graph lookup:
  - Preferred: `context: { projectRoot, packageId, workflowId, runId, agentId }`
  - Alternative: pass `graphRelPath` directly if `workflowId` is not available at tool time.
- Validation rules (effective IDs):
  - `currentNodeId_oldEffective = trim(currentNodeId_oldRaw) || graph.entryNodeId`
  - `currentNodeId_newEffective = trim(currentNodeId_newRaw) || graph.entryNodeId`
  - If `currentNodeId_newEffective !== currentNodeId_oldEffective`, require an outgoing edge `from=currentNodeId_oldEffective` and `to=currentNodeId_newEffective`.
- `stepsCompleted` must be monotonic (new is a superset of old). `fs.write` to `@state/workflow.md` MUST NOT bypass this guard.
- Tool errors (extend Story 4.6 codes):
  - `E_SCHEMA_VALIDATION`: include AJV errors in `details.errors`.
  - `E_INVALID_TRANSITION`: include `{ from, to, allowedNext: Array<{ to, label, isDefault?, conditionText? }> }` in `details`.
  - Preserve: `E_INVALID_FRONTMATTER`, `E_PRECONDITION_FAILED`.
- Graph conditions:
  - MVP validates **edge existence only** (does not evaluate `conditionExpr`); include `conditionText` in `allowedNext` for human/LLM guidance.

### Data / Storage

- Reads:
  - `@state/workflow.md` (previous content for sha + previous frontmatter)
  - selected workflow graph JSON at `@pkg/<graphRelPath>` (from workflow record / `RUN_DIRECTIVE.graph`; optionally cached by `{packageId, workflowId}`)
- Writes:
  - `@state/workflow.md` via atomic rename (tmp → rename) only on validation success
  - Audit entry appended to `@state/logs/execution.jsonl` (already in ToolHost)

### Errors / Edge Cases

- Empty/whitespace `currentNodeId`:
  - Allowed in file; normalize to `graph.entryNodeId` for transition checks and execution.
- Transition to unknown node or disallowed successor:
  - Reject with `E_INVALID_TRANSITION` and include `details.allowedNext` derived from outgoing edges of `currentNodeId_oldEffective`.
- Schema invalid frontmatter:
  - Reject with `E_SCHEMA_VALIDATION` (include AJV errors, no absolute paths).
- Optimistic concurrency:
  - Respect `ifMatchSha256` on both `fs.apply_patch` and `fs.write` (reject with `E_PRECONDITION_FAILED`).

### Test Plan

- Unit (vitest):
  - Valid transition `A → B` (edge exists) allows `fs.apply_patch` to persist an update.
  - Invalid transition `A → Z` rejects with `E_INVALID_TRANSITION` and includes `details.allowedNext`; file remains unchanged.
  - Unknown node id rejects with `E_INVALID_TRANSITION` (and still includes `allowedNext` from old effective node).
  - Schema invalid frontmatter rejects with `E_SCHEMA_VALIDATION` (include AJV errors); file remains unchanged.
  - `stepsCompleted` regression attempt (via `fs.write` to `@state/workflow.md`) is rejected; file remains unchanged.
  - Empty `currentNodeId` normalization: oldRaw `""` treated as entry node for allowedNext/transition checks.

## Tasks / Subtasks

- [x] Add a shared validator: `validateWorkflowStateUpdate(prevFrontmatter, nextFrontmatter, graph)` returning `{ ok } | { error }`.
- [x] Extend ToolHost context to include `workflowId` (or `graphRelPath`) so `FileSystemToolHost` can load the correct graph.
- [x] Enforce validation for `fs.apply_patch` when `path === @state/workflow.md`:
  - [x] Parse frontmatter, apply patch, validate, then write atomically.
  - [x] On invalid transition, return `E_INVALID_TRANSITION` + `allowedNext`.
- [x] Enforce validation for `fs.write` when `path === @state/workflow.md` (either validate full file content or reject and require `fs.apply_patch`).
- [x] Add unit tests in `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` for transition/schema/regression cases.

## Verification

- `npm -C crewagent-runtime run test`

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-7-validate-graph-transition-and-update-frontmatter.md`
- Epics: `_bmad-output/epics.md` (Story 4.7)
- Architecture: `_bmad-output/architecture/runtime-architecture.md` (Section 8.2: 状态写入一致性)
- Entrypoints: `_bmad-output/architecture/entrypoints-agent-vs-workflow.md` (tool loop + state reload)
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` (Section 6.6 `fs.apply_patch`)
- Schema: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
- Schema: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
- Story 4.6: `_bmad-output/implementation-artifacts/4-6-execute-tool-calls-sandboxed-filesystem.md`

## Change Log

- 2026-01-03: Added graph-aware validation for `@state/workflow.md` writes (schema + transition + monotonicity) and expanded ToolHost context; added unit tests.

## Dev Agent Record

### Completion Notes

- Implemented graph-aware validation for `fs.apply_patch` / `fs.write` to `@state/workflow.md` (schema validation, transition guard, and `stepsCompleted` non-regression).
- Extended ToolHost execution context to pass `workflowId` so ToolHost can load the selected workflow graph.
- Tests: `npm -C crewagent-runtime test`.

### File List

- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
