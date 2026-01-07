# Code Review: Story 4-2 – Load Workflow Graph and Step Files

**Date:** 2026-01-01  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `4-2-load-workflow-graph-and-step-files.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 2 |
| HIGH Issues | 0 |
| MEDIUM Issues | 3 |
| LOW Issues | 2 |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |
| Unit Tests | ✅ `npx vitest run` (in `crewagent-runtime/`) passed |
| Lint | ❌ repo-wide lint currently fails (unrelated issues) |

---

## 🟡 MEDIUM ISSUES

### M1. IPC contract in story doc doesn’t match implementation shape
- **Story Contract:** `workflow:load` response described as `{ graph, steps }`.
- **Current Implementation:** Returns `{ success, definition?: { graph, steps }, error? }`.
- **Evidence:**
  - IPC handler: `crewagent-runtime/electron/main.ts:475`
  - Preload bridge: `crewagent-runtime/electron/preload.ts:49`
  - Typed renderer API: `crewagent-runtime/electron/electron-env.d.ts:151`
- **Impact:** Story doc can mislead frontend/engine integration (payload shape mismatch).
- **Fix:** Update story doc “API / Contracts” to reflect `{ success, definition, error }` (or change code to match the doc).

### M2. Errors are not structured (missing `code/details`)
- **Story Design:** Mentions structured errors with `message` (+ optional `code/details`) for UI messaging.
- **Current:** Returns `error: string` only.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:1024` returns `{ success:false, error: string }`.
- **Impact:** UI can’t reliably branch on error types (missing schema vs missing file vs invalid edge) without string matching.
- **Fix:** Introduce an error shape `{ code: 'GRAPH_NOT_FOUND' | 'INVALID_GRAPH' | 'FILE_NOT_FOUND' | ...; message: string; details?: unknown }`.

### M3. No test script in `crewagent-runtime/package.json`
- **Current:** `vitest` exists and tests pass, but there’s no `npm run test`.
- **Impact:** CI/dev workflow is less discoverable; story/tech-spec “npm run test” guidance won’t work.
- **Fix:** Add `"test": "vitest run"` to `crewagent-runtime/package.json`.

---

## 🟢 LOW ISSUES

### L1. `schemaVersion` is validated twice
- **Current:** JSON Schema already constrains `schemaVersion` (`^1\\.1(\\.\\d+)?$`), and `validateGraphIntegrity()` also checks `startsWith('1.1')`.
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.ts:953`.
- **Impact:** Minor redundancy; can drift if schema changes.
- **Fix:** Keep one source of truth (prefer schema), or make the integrity check explicitly accept only schema-passing graphs.

### L2. No negative tests for schema invalidation paths
- **Current Tests:** Cover “edge references missing node” and “missing step file” but not invalid schema (e.g., missing `label`, wrong `node.type`).
- **Evidence:** `crewagent-runtime/electron/stores/runtimeStore.test.ts:94` onwards.
- **Impact:** Lower confidence that schema errors produce expected messages.
- **Fix:** Add one test case for schema invalid graph and assert `Invalid workflow graph:` error.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC1: Locate + parse graph | ✅ | `RuntimeStore.loadWorkflow()` resolves `workflow.graph.json` via package metadata |
| AC2: Validate schema + integrity | ✅ | JSON Schema validation + `validateGraphIntegrity()` checks entry/nodes/edges |
| AC3: Load all node step files; fail on missing | ✅ | `loadStepFiles()` reads all node `.md` files and fails with clear error |
| AC4: Return graph + steps map | ✅ | Returns `WorkflowDefinition { graph, steps }` |

---

## Notes (Implementation Strengths)

- Uses `resolveInsideDir()` for graph + step file paths (prevents package path traversal).
- Has both schema validation and logical validation (duplicate IDs, edge validity, entry node existence).
- Includes negative tests for missing node edges and missing step files.

---

## Next Actions

- [ ] [AI-Review][MEDIUM] Align story IPC contract with actual `{ success, definition, error }` response
- [ ] [AI-Review][MEDIUM] Add structured error codes for `loadWorkflow()` failures
- [ ] [AI-Review][MEDIUM] Add `crewagent-runtime` `"test"` script for vitest
- [ ] [AI-Review][LOW] Add one schema-invalid negative test
