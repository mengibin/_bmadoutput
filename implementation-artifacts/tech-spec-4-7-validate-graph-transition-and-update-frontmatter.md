# Tech Spec: Story 4.7 - Validate Graph Transition & Update Frontmatter

**Created:** 2026-01-03  
**Status:** Ready for Development  
**Source Story:** `_bmad-output/implementation-artifacts/4-7-validate-graph-transition-and-update-frontmatter.md`

## Overview

### Problem Statement

In the v1.1 micro-file runtime, the LLM advances workflow execution by writing run state into `@state/workflow.md` frontmatter (Document-as-State).

Story 4.6 implemented a sandboxed filesystem ToolHost with `fs.apply_patch(updateFrontmatter)` and atomic writes, but currently **does not** guard:

- YAML/frontmatter correctness (parseability).
- Frontmatter schema compliance (`workflow-frontmatter.schema.json` v1.1).
- Graph transition legality (prevent invalid node jumps).
- `stepsCompleted` monotonicity (prevent regression).

Without these guards, a single bad update can corrupt run state and break resume/recovery.

### Solution

Introduce a **state-write validation layer** for `@state/workflow.md`:

1. Parse existing and next `workflow.md` using `gray-matter`.
2. Normalize the frontmatter object and validate it with AJV2020 + `ajv-formats` against `workflow-frontmatter.schema.json` (v1.1).
3. Load the **selected workflow graph** (the graph referenced by the workflow record / `RUN_DIRECTIVE.graph`), and validate:
   - `currentNodeId` exists in graph (after normalization).
   - `currentNodeId` transitions only follow an outgoing edge.
   - `stepsCompleted` does not regress.
4. Reject invalid updates with a structured ToolError that includes an actionable `allowedNext` list for the LLM to self-correct.

### Scope (In / Out)

**In scope**
- Validate writes to `@state/workflow.md` coming from:
  - `fs.apply_patch` (primary path).
  - `fs.write` (must not bypass validation).
- Add `E_SCHEMA_VALIDATION` and `E_INVALID_TRANSITION` failures with `details.allowedNext`.
- Extend tool execution context so ToolHost can resolve the correct graph (`workflowId` or `graphRelPath`).
- Add unit tests for schema violations, invalid transitions, and regression.

**Out of scope / deferred**
- Evaluating edge conditions (`conditionExpr`) at runtime. (MVP: guard by edge existence only; surface `conditionText` for guidance.)
- UI-level rendering improvements; UI can surface the ToolError as-is.

## Context for Development

### Codebase Patterns

- Runtime “main process” services: `crewagent-runtime/electron/services/*`
- Package + schema validation lives in `crewagent-runtime/electron/stores/runtimeStore.ts` (AJV2020 + `ajv-formats`).
- Tool execution path:
  - `ExecutionEngineImpl` calls `ToolHost.executeToolCall(...)` and backfills `role="tool"` messages (`crewagent-runtime/electron/services/executionEngine.ts`).
  - File tools are implemented in `crewagent-runtime/electron/services/fileSystemToolHost.ts`.

### Files to Reference

- Story: `_bmad-output/implementation-artifacts/4-7-validate-graph-transition-and-update-frontmatter.md`
- File tools: `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- ToolHost types: `crewagent-runtime/electron/services/toolHost.ts`
- Engine integration: `crewagent-runtime/electron/services/executionEngine.ts`
- Schema validation patterns: `crewagent-runtime/electron/stores/runtimeStore.ts` (`Ajv2020`, `addFormats`, `validateWorkflowFrontmatterV11`, `validateWorkflowGraphV11`)
- Graph schema: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
- Frontmatter schema: `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-frontmatter.schema.json`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md` (§6.6 `fs.apply_patch`)
- Architecture: `_bmad-output/architecture/runtime-architecture.md` (§8.2 状态写入一致性)

### Technical Decisions

1. **Graph source of truth is the selected workflow graph**
   - Do not hardcode `@pkg/workflow.graph.json`.
   - Resolve graph via `workflowId -> workflow.graph` (or pass `graphRelPath` directly).

2. **Normalize `currentNodeId` before validation**
   - Use the same semantics as `RuntimeStore.getWorkflowState`:
     - `effectiveId = trim(raw) || graph.entryNodeId`
   - Transition checks use effective ids to avoid treating `""` as a separate node.

3. **Reject invalid updates without touching disk**
   - Validation runs before writing `@state/workflow.md`.
   - On failure, return `ToolResult = { ok: false, error: ... }`.

4. **`fs.write` must not bypass validation**
   - If `fs.write.path === '@state/workflow.md'`, validate the full content (same rules as apply_patch) before writing.

5. **Allowed-next guidance is mandatory on invalid transitions**
   - Return `details.allowedNext` derived from outgoing edges of `currentNodeId_oldEffective`.
   - Include `label/isDefault/conditionText` when present to help the LLM choose the correct branch.

## Contracts

### ToolHost Execution Context

`FileSystemToolHost` needs the selected workflow graph to validate transitions. Add `workflowId` to ToolHost execution context.

Proposed minimal change:

```ts
// crewagent-runtime/electron/services/toolHost.ts
executeToolCall(params: {
  toolCallId: string
  name: string
  argumentsJson: string
  context: { projectRoot: string; packageId: string; workflowId: string; runId: string; agentId: string }
  signal?: AbortSignal
}): Promise<ToolResult>
```

`ExecutionEngineImpl` already has `workflowId` in `runLoop(...)` params and should pass it through.

### Error Codes & Shapes

ToolHost already uses structured errors. Extend with:

- `E_SCHEMA_VALIDATION`: frontmatter fails AJV schema validation.
- `E_INVALID_TRANSITION`: `currentNodeId` invalid or transition not allowed.

Recommended invalid-transition error shape:

```ts
type AllowedNext = Array<{ to: string; label: string; isDefault?: boolean; conditionText?: string }>

{
  ok: false,
  error: {
    code: 'E_INVALID_TRANSITION',
    message: "Invalid transition: <from> → <to>",
    details: {
      from: string,
      to: string,
      allowedNext: AllowedNext
    }
  }
}
```

## Implementation Plan

### Tasks

- [ ] Extend `ToolHost.executeToolCall(...).context` to include `workflowId`.
- [ ] Extend `FileSystemToolHostRuntimeStore` to resolve the selected workflow graph:
  - [ ] Either: `getPackage(packageId)` returns `{ path, workflows: Array<{id, graph}> }`
  - [ ] Or: add a dedicated method `getWorkflowGraphRelPath(packageId, workflowId): string`.
- [ ] Add `WorkflowGraph` loader (read + JSON parse + optional AJV validate) with caching by `{packageId, workflowId}`.
- [ ] Add a shared validator:
  - `validateWorkflowStateUpdate(prevFrontmatter, nextFrontmatter, graph)` that enforces:
    - YAML/frontmatter parse ok
    - AJV schema valid (v1.1 frontmatter)
    - `currentNodeId_newEffective` exists in graph
    - transition edge exists when changing nodes
    - `stepsCompleted_new` is a superset of `stepsCompleted_old`
- [ ] Enforce validation for `fs.apply_patch` on `@state/workflow.md`:
  - [ ] Apply patch in-memory.
  - [ ] Validate against previous state + graph.
  - [ ] Write atomically only on success.
- [ ] Enforce validation for `fs.write` when `path === '@state/workflow.md'`:
  - [ ] Parse next content, validate, then write atomically.
- [ ] Tests (`vitest`):
  - [ ] Invalid transition returns `E_INVALID_TRANSITION` and includes `details.allowedNext`.
  - [ ] Unknown node id returns `E_INVALID_TRANSITION`.
  - [ ] Schema invalid returns `E_SCHEMA_VALIDATION` with AJV errors in `details`.
  - [ ] `stepsCompleted` regression rejected (only possible via `fs.write` path).

### Acceptance Criteria

- [ ] AC1: YAML/frontmatter parseable + schema-valid (`workflow-frontmatter.schema.json` v1.1).
- [ ] AC2: `currentNodeId` exists in graph (after normalization) and transition is guarded by outgoing edges.
- [ ] AC3: `stepsCompleted` does not regress.
- [ ] AC4: Invalid updates return structured error + `allowedNext`.
- [ ] AC5: Valid updates are atomic and recovery depends only on `@state/workflow.md` + selected `@pkg/...graph`.

## Additional Context

### Dependencies

- `gray-matter` (already in use)
- `ajv/dist/2020` + `ajv-formats` (already in use in `RuntimeStore`)

### Testing Strategy

- Add focused unit tests next to existing tool host tests:
  - `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- Use a tiny synthetic workflow graph fixture for deterministic transition checks.
- Ensure tests assert “no write occurred” on validation failure (file content remains unchanged).

### Notes

- This story only validates **edge existence**; it does not evaluate conditions. The LLM already receives `allowedNext` (with `conditionText`) in `NODE_BRIEF`, and validation errors should echo the same list for self-correction.

## Traceability

- Story: `_bmad-output/implementation-artifacts/4-7-validate-graph-transition-and-update-frontmatter.md`
- Architecture: `_bmad-output/architecture/runtime-architecture.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`

