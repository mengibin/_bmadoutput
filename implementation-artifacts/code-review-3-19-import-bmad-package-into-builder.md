# Code Review: Story 3-19 – Import .bmad Package into Builder

**Date:** 2026-01-07  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `3-19-import-bmad-package-into-builder.md`

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | N/A (repo has no `.git/`) |
| HIGH Issues | 0 (resolved) |
| MEDIUM Issues | 0 (resolved) |
| LOW Issues | 0 (resolved) |
| Frontend Lint | ✅ `npm -C crewagent-builder-frontend run lint` (warnings only) |
| Frontend Build | ✅ `npm -C crewagent-builder-frontend run build` |
| Backend Tests | ✅ `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests` |

---

## 🔴 HIGH ISSUES

### H1. Import does not validate `bmad.json` / `agents.json` / `workflow.graph.json` against v1.1 JSON Schemas (AC-2 gap)
- **Status:** ✅ Fixed
- **What’s wrong:** The story requires schema validation, but the backend only does ad-hoc checks (e.g. `schemaVersion.startswith("1.1")`) and some graph/step validations.
- **Evidence:**
  - Backend import only checks schemaVersion (no schema validation): `crewagent-builder-backend/app/services/package_import_service.py:517`
  - Graph validation is described as “beyond JSON Schema”, but JSON Schema is never applied: `crewagent-builder-backend/app/services/package_import_service.py:402`
  - Existing tests use **out-of-spec** `bmad.json` (missing required `version` + `entry` per `bmad.schema.json`), meaning current behavior is already spec-divergent: `crewagent-builder-backend/tests/test_step_integration.py:19`
- **Impact:** Builder can import invalid packages that will fail later (runtime validation, export, execution), and malformed JSON can trigger 500s instead of actionable errors.
- **Fix:** Add JSON Schema validation using the official schemas in `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/` (or Builder-copied equivalents), then update tests to include required `bmad.json` fields.

### H2. AgentId reference validation is skipped for single-workflow packages (`steps/*.md` at root)
- **Status:** ✅ Fixed
- **What’s wrong:** Single-workflow steps are detected via `path.startswith("steps/")`, but agent reference validation later only scans paths containing `"/steps/"`, which excludes `steps/*.md`.
- **Evidence:** `crewagent-builder-backend/app/services/package_import_service.py:634`
- **Impact:** Invalid `agentId` references in single-workflow packages can pass import and then fail at runtime/editor binding.
- **Fix:** Include `path.startswith("steps/")` in the agent-reference scan filter (same as the step parsing logic).

### H3. Single-workflow import stores step keys as bare filenames, breaking Builder export constraints
- **Status:** ✅ Fixed
- **What’s wrong:** In single-workflow mode, the imported `step_files_json` uses keys like `step1.md` instead of `steps/step1.md`.
- **Evidence:**
  - Import drops the directory: `crewagent-builder-backend/app/services/package_import_service.py:614`
  - Export expects step keys to start with `steps/` and fails otherwise: `crewagent-builder-frontend/src/lib/bmad-zip-v11.ts:214`
- **Impact:** Imported projects may fail re-export until users manually “touch” steps in the editor or the system rewrites keys.
- **Fix:** Preserve normalized paths (prefer `steps/<filename>.md` for single-workflow) when populating `step_files_json`.

### H4. Name-conflict UX is not implemented (AC-4 missing)
- **Status:** ✅ Fixed
- **What’s wrong:** Backend supports name conflict detection and even exposes `/packages/import/check-name`, but the Dashboard import modal only shows an error string; it doesn’t prompt for rename/retry.
- **Evidence:**
  - UI just sets `importError` and returns: `crewagent-builder-frontend/src/app/dashboard/page.tsx:123`
  - Conflict check endpoint exists: `crewagent-builder-backend/app/routers/packages.py:123`
  - Backend raises conflict error: `crewagent-builder-backend/app/services/package_import_service.py:662`
- **Impact:** Users can’t recover from a common import failure without leaving the flow; this is a direct AC miss.
- **Fix:** Add a conflict dialog that offers a suggested name and retries import using `custom_name` query param (or call `/import/check-name` and then retry).

---

## 🟡 MEDIUM ISSUES

### M1. Multi-workflow “default workflow” ignores `bmad.json.entry` and always picks the first workflow
- **Status:** ✅ Fixed
- **What’s wrong:** Imported workflows are marked default via `is_default=(i==0)` rather than matching the workflow referenced by `bmad.json.entry.workflow/graph`.
- **Evidence:** `crewagent-builder-backend/app/services/package_import_service.py:689`
- **Impact:** UI may open the wrong workflow by default; imported package semantics aren’t preserved.
- **Fix:** Determine default workflow by matching `entry.workflow` / `entry.graph` paths to a workflow item and mark that one as default.

### M2. Graph validation can crash on malformed graphs (KeyError / attribute errors) instead of returning validation errors
- **Status:** ✅ Fixed
- **What’s wrong:** `nodes = {n["id"]: n for n in graph.get("nodes", [])}` assumes every node is a dict with `id`. Same for edges assuming dicts.
- **Evidence:** `crewagent-builder-backend/app/services/package_import_service.py:413`
- **Impact:** Malformed `workflow.graph.json` can produce 500s instead of actionable `PKG_VALIDATION_FAILED` errors.
- **Fix:** Add defensive parsing (type checks) and emit `PKG_SCHEMA_ERROR` / `PKG_GRAPH_ERROR` entries rather than throwing.

### M3. Error “file” paths are misleading for multi-workflow graphs
- **Status:** ✅ Fixed
- **What’s wrong:** `validate_graph_constraints()` emits errors with `file="workflow.graph.json"` even when validating `workflows/<id>/workflow.graph.json`.
- **Evidence:** `crewagent-builder-backend/app/services/package_import_service.py:420`
- **Impact:** Users get error reports that point to the wrong file, slowing fixes.
- **Fix:** Pass the graph path into `validate_graph_constraints(graph_path=...)` and use it in `ValidationError.file`.

### M4. Story file is not review-ready (missing Dev Agent Record / File List)
- **Status:** ✅ Fixed
- **What’s wrong:** The story is marked `ready-for-dev` and lacks a Dev Agent Record, but the feature is already implemented in code (Dashboard + backend import service). This makes the story impossible to audit via “story claims vs implementation”.
- **Evidence:** Story lacks Dev Agent Record section: `_bmad-output/implementation-artifacts/3-19-import-bmad-package-into-builder.md:1`
- **Impact:** Process drift: review artifacts can’t reliably trace changes or verify tasks marked done.
- **Fix:** Either (a) move import story tracking to the actual story where it was implemented (you indicated Story 6-1), or (b) complete this story file with proper Dev record and status updates.

---

## 🟢 LOW ISSUES

### L1. Assets import accepts unsafe path segments (drops silently later)
- **Status:** ✅ Fixed
- **What’s wrong:** Backend imports any `assets/...` file path without checking for `..` segments; frontend parsing will later drop invalid paths.
- **Evidence:** `crewagent-builder-backend/app/services/package_import_service.py:649`
- **Impact:** Confusing “missing assets after import” behavior; mild security/consistency concern.
- **Fix:** Validate asset paths are safe (`no ..`, no empty segments) and report a validation error when invalid.

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1: Upload UI (.bmad picker) | ✅ | `accept=".bmad"` + extension check in `DashboardPage`: `crewagent-builder-frontend/src/app/dashboard/page.tsx:458` |
| AC-2: Extract + schema validate | ✅ | v1.1 JSON Schema validation added for `bmad.json`, `agents.json`, `workflow.graph.json`, plus frontmatter schemas: `crewagent-builder-backend/app/services/package_import_service.py:60` |
| AC-3: Create project from package | ✅ | `import_package()` creates `WorkflowPackage` + `PackageWorkflow` records: `crewagent-builder-backend/app/services/package_import_service.py:670` |
| AC-4: Conflict handling UI | ✅ | 409 + suggested name meta + retry flow in Dashboard: `crewagent-builder-frontend/src/app/dashboard/page.tsx:138` |
| AC-5: Multi-workflow import preserved | ✅ | Default workflow is selected based on `bmad.json.entry`: `crewagent-builder-backend/app/services/package_import_service.py:635` |

---

## Notes (Implementation Strengths)

- Good zip bomb guard by total uncompressed size limit: `crewagent-builder-backend/app/services/package_import_service.py:55`
- Step-template validation is detailed and includes line numbers for YAML issues (useful UX): `crewagent-builder-backend/app/services/package_import_service.py:274`

---

## Next Actions

- [x] [AI-Review][HIGH] Add v1.1 JSON Schema validation for `bmad.json/agents.json/workflow.graph.json` and update backend tests accordingly
- [x] [AI-Review][HIGH] Fix agentId reference scan to include `steps/*.md` in single-workflow mode
- [x] [AI-Review][HIGH] Preserve `steps/` prefix in imported `step_files_json` keys for single-workflow packages
- [x] [AI-Review][HIGH] Implement Dashboard name-conflict resolution (retry with `custom_name`)
- [x] [AI-Review][MEDIUM] Determine default workflow based on `bmad.json.entry` instead of first workflow
- [x] [AI-Review][MEDIUM] Harden `validate_graph_constraints()` against malformed graph payloads and improve error file paths
- [x] [AI-Review][LOW] Validate asset paths to avoid silently dropping unsafe paths after import
