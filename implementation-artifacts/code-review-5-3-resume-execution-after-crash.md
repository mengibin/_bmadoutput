# Code Review: Story 5-3 – Workflow Progress with Real Data

**Date:** 2026-01-01  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `5-3-resume-execution-after-crash.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 5 |
| HIGH Issues | 2 |
| MEDIUM Issues | 5 |
| LOW Issues | 3 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ⚠️ Exists (`vitest`) but no `test` script |
| Lint | ❌ Fails due to unrelated issues in repo |

---

## 🔴 CRITICAL ISSUES (HIGH)

### H1. “Real-time updates” not implemented (no `runs:progressUpdated` emitter)
- **Story Requirement:** AC3 requires progress UI to update automatically when `@state/workflow.md` changes.
- **Current:** Renderer subscribes to `runs:progressUpdated`, but the main process never emits this event.
- **Evidence:**
  - Subscribed: `crewagent-runtime/src/hooks/useWorkflowProgress.ts:62`
  - No emitter found: `rg "runs:progressUpdated"` only matches the hook.
- **Impact:** Progress panel only updates on initial load; users will not see step transitions during execution.
- **Fix:** Emit `runs:progressUpdated` whenever the run state changes (likely after `fs.apply_patch` / state writes in the ExecutionEngine or ToolHost). Until ExecutionEngine exists, add a dev-only simulation path (similar to Story 5.2).

### H2. Run-scoped state path is assumed but not wired to `@state` mount yet
- **Story Requirement:** Data source is `@state/workflow.md` for a specific run.
- **Current:** `RuntimeStore.getWorkflowProgress` reads `RuntimeStoreRoot/projects/<projectId>/runs/<runId>/state/workflow.md`, but `@state` alias currently maps to `RuntimeStoreRoot/projects/<projectId>/state` and is read-only.
- **Evidence:**
  - Reads run-scoped: `crewagent-runtime/electron/stores/runtimeStore.ts:1445`
  - `@state` mapping: `crewagent-runtime/electron/stores/runtimeStore.ts:724`
- **Impact:** Even if tools write to `@state/workflow.md`, it won’t land in the run-scoped location this story reads; progress will stay at defaults in most real runs.
- **Fix:** Align `@state` to a run context (Story 4-8 / run-scoped state) or change this story to explicitly read the current (project-scoped) state path until run-scoping is implemented.

---

## 🟡 MEDIUM ISSUES

### M1. Step display name likely wrong (`node.name` vs `node.title`)
- **Current:** `getWorkflowProgress` uses `node.name`, but exported graphs use `title` (per spec examples).
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1488` and graph examples under `crewagent-runtime/spec/.../workflow.graph.json`.
- **Impact:** UI shows node IDs instead of human titles (“step-01-intake” instead of “Intake”).
- **Fix:** Prefer `title` (and fall back to `id`).

### M2. Graph path resolution is not sandboxed (potential path escape)
- **Current:** `const graphPath = path.join(pkg.path, workflow.graph)` accepts arbitrary `workflow.graph` values.
- **Impact:** A crafted package could attempt to reference files outside package root.
- **Fix:** Use existing `resolveInsideDir(pkg.path, workflow.graph)` (already used in `loadWorkflow()`).

### M3. Frontmatter parsing is brittle (regex; only supports inline `stepsCompleted: [...]`)
- **Current:** Simple regex parsing for `currentNodeId` and `stepsCompleted: [..]`.
- **Impact:** Real YAML lists (`stepsCompleted:\n - a\n - b`) will not parse; UI silently shows wrong progress.
- **Fix:** Reuse `gray-matter` (`matter(content)`) like `getWorkflowState()` does, or parse YAML properly.

### M4. “Execution order” not guaranteed
- **Current:** Steps are derived from `graph.nodes.map(...)` without computing an execution order.
- **Impact:** If node array ordering changes (or complex graphs), the progress list order may not match actual execution traversal.
- **Fix:** Either document that Builder outputs nodes in execution order (and validate), or compute a traversal order from `entryNodeId` + `edges` (with branch handling).

### M5. Story artifact inconsistencies (status/name/DoD)
- **Current:** Story file says `Status: done`, but Definition of Done checklist is unchecked, and sprint status currently says `ready-for-dev`.
- **Evidence:** `_bmad-output/implementation-artifacts/5-3-resume-execution-after-crash.md:3` and `_bmad-output/implementation-artifacts/sprint-status.yaml:98`.
- **Fix:** Set story status to `review` until H1/H2 are addressed (or adjust scope/AC), then move to `done` after validation.

---

## 🟢 LOW ISSUES

### L1. IPC payload validation missing for `runs:getProgress`
- **Current:** No runtime validation for `packageId/workflowId/runId/projectRoot`.
- **Fix:** Validate `payload` shape and reject invalid values early for clearer errors.

### L2. `runs:getProgress` handler lacks try/catch
- **Current:** Delegates to store method; store catches internally, but main should still guard and standardize `{ success:false, error }`.

### L3. Missing tests for `getWorkflowProgress`
- **Current:** There are vitest tests for importing packages and parsing workflow state, but none for run progress derivation.
- **Fix:** Add a unit test that writes a sample run state `workflow.md` into the runtime store run path and asserts step statuses.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Steps from graph | ✅ | `runs:getProgress` reads `workflow.graph.json` and maps nodes |
| AC2: Status from `stepsCompleted/currentNodeId` | ⚠️ Partial | Works only for inline `stepsCompleted: [..]`; YAML list format will fail |
| AC3: Real-time updates | ❌ | No `runs:progressUpdated` emitter exists |
| AC4: Resume + “Continue” button | ❌ | Progress can load last state if file exists; no “Continue” UI/action implemented |

---

## Next Actions

- [ ] [AI-Review][HIGH] Implement `runs:progressUpdated` emission on state changes (or add dev simulation until ExecutionEngine exists)
- [ ] [AI-Review][HIGH] Align `@state` mapping with run-scoped state (Story 4-8) or change this story’s read path to match current mount behavior
- [ ] [AI-Review][MEDIUM] Use `node.title` for display name, fall back to `id`
- [ ] [AI-Review][MEDIUM] Use `resolveInsideDir()` for graph path; prevent path escape
- [ ] [AI-Review][MEDIUM] Parse frontmatter via `gray-matter` (or YAML parser) instead of regex
- [ ] [AI-Review][MEDIUM] Decide on ordering guarantee vs traversal computation
- [ ] [AI-Review][LOW] Add vitest coverage for `getWorkflowProgress`
