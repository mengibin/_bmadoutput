# Code Review: Story 4-3 – Parse Frontmatter to Determine Current Node

**Date:** 2026-01-01  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `4-3-parse-frontmatter-to-determine-current-node.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Story vs Implementation Discrepancies | 4 |
| HIGH Issues | 1 |
| MEDIUM Issues | 3 |
| LOW Issues | 2 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ✅ `npm -C crewagent-runtime run test` passed |
| Lint | ❌ repo-wide lint currently fails (unrelated issues) |

---

## 🔴 HIGH ISSUES

### H1. Potential path traversal via `runId` when computing run state paths
- **Current:** `ensureRunStateDir(projectRoot, runId)` builds the path with `path.join(projectRuntimeRoot, 'runs', runId, 'state')` and immediately `mkdirSync` it.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1660` (`ensureRunStateDir`)
- **Impact:** A malicious or compromised renderer could pass `runId` containing `..` or path separators to write/read outside the intended `runs/<runId>/state` directory (escape `projectRuntimeRoot`).
- **Fix:** Validate `runId` with a strict allowlist (e.g. reuse `assertSimpleName(runId)`), or resolve + verify the final path remains inside `<projectRuntimeRoot>/runs/`.

---

## 🟡 MEDIUM ISSUES

### M1. Story IPC contract for `workflow:getState` doesn’t match implementation
- **Story Contract:** Describes request as `{ runId: string }` and response as raw `WorkflowState`.
- **Implementation:** `workflow:getState` requires `{ packageId, workflowId, runId, projectRoot }` and returns `{ success, state?, error? }`.
- **Evidence:**
  - IPC handler: `crewagent-runtime/electron/main.ts:491`
  - Preload bridge: `crewagent-runtime/electron/preload.ts:51`
  - Typed renderer API: `crewagent-runtime/electron/electron-env.d.ts:161`
- **Impact:** Story doc can mislead UI/engine integration.
- **Fix:** Update story doc “API / Contracts” to reflect the actual payload and response shape.

### M2. Error contract mismatch: story calls for structured errors, implementation returns `string`
- **Story Design:** Mentions structured errors `{ code, message, details? }` for blocking + repair hints.
- **Implementation:** `getWorkflowState()` returns `{ success:false, error: string }` and IPC passes it through.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1198`
- **Impact:** UI can’t branch on error types without brittle string matching; inconsistent with Story 4.2’s structured error shape.
- **Fix:** Introduce a `WorkflowStateError` type similar to `WorkflowLoadError` (Story 4.2) and return `{ error: { code, message, details? } }`.

### M3. Read path creates directories as a side effect
- **Current:** `getWorkflowState()` calls `ensureRunStateDir()` which creates directories even when the state file is missing.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1234`
- **Impact:** “Read” operations have write side effects; can create empty `runs/<runId>/state/` folders for invalid run IDs and make debugging harder.
- **Fix:** Replace `ensureRunStateDir()` with a “compute-only” helper (no `mkdir`), and let run creation stories (e.g. 4.8/4.7) own directory creation.

---

## 🟢 LOW ISSUES

### L1. Tech spec checklist/status not updated after implementation
- **Current:** Tech spec still shows “Ready for Development” and unchecked tasks.
- **Evidence:** `_bmad-output/implementation-artifacts/tech-spec-4-3-parse-frontmatter-to-determine-current-node.md`
- **Impact:** Reduces traceability; may confuse “what’s done” vs “what’s planned”.
- **Fix:** Mark completed items and update status to reflect implementation + tests.

### L2. Graph loading logic duplicated across `loadWorkflow()` and `getWorkflowState()`
- **Current:** `getWorkflowState()` reimplements graph read + schema validation + integrity validation instead of reusing a shared helper.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1206`
- **Impact:** Higher maintenance cost (future schema/path changes need multiple edits).
- **Fix:** Factor out a `loadWorkflowGraph(packageId, workflowId)` helper that returns `{ success, graph, error }` without loading step files.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Read active workflow.md | ✅ | Reads run state `runs/<runId>/state/workflow.md` when present |
| AC2: Parse YAML frontmatter + currentNodeId | ✅ | `gray-matter` parsing and frontmatter extraction |
| AC3: Fallback currentNodeId to graph.entryNodeId | ✅ | Trims and falls back when empty |
| AC4: Return state fields | ✅ | Returns `WorkflowState` with `variables/stepsCompleted/decisionLog/artifacts` |

---

## Notes (Implementation Strengths)

- Uses schema validation (`workflow-frontmatter.schema.json` and `workflow-graph.schema.json`) via AJV before trusting parsed content.
- Provides a helpful hint message when `currentNodeId` doesn’t exist in the graph (includes sample node IDs).
- Includes unit tests for: normal parse, empty currentNodeId fallback, invalid currentNodeId blocking.

---

## Next Actions

- [ ] [AI-Review][HIGH] Validate `runId` and prevent path traversal in `ensureRunStateDir()`
- [ ] [AI-Review][MEDIUM] Align story IPC contract (`workflow:getState`) with implementation
- [ ] [AI-Review][MEDIUM] Adopt structured error codes for workflow state parsing
- [ ] [AI-Review][MEDIUM] Remove directory creation side effect from read path
- [ ] [AI-Review][LOW] Update tech spec checklists/status
- [ ] [AI-Review][LOW] Deduplicate graph loading logic

