# Tech Spec: Import `.bmad` Package into Builder (v1.1)

**Source Story:** `_bmad-output/implementation-artifacts/3-19-import-bmad-package-into-builder.md`

## Overview

Builder supports importing a `.bmad` ZIP bundle (Package Spec v1.1) to create a new project containing:
- Project-level agents (`agents.json`)
- One or more workflows (`workflow.graph.json`, `workflow.md`, `steps/*.md`)
- Assets (`assets/**`)

Import runs **server-side** to avoid leaking filesystem details and to keep validation consistent.

## API

### `POST /packages/import`

- Auth: required
- Content-Type: `multipart/form-data`
- Body:
  - `file`: `.bmad` zip
- Query (optional):
  - `custom_name`: override project name (used for conflict resolution)

**Success (201)**
```json
{
  "data": {
    "project_id": 123,
    "project_name": "My Package",
    "workflows_count": 2,
    "agents_count": 5
  },
  "error": null
}
```

**Errors**
- `400 PKG_INVALID_FORMAT` (not `.bmad`)
- `413 PKG_TOO_LARGE` (zip > 50MB)
- `400 PKG_VALIDATION_FAILED` with `details[]` (schema/refs/frontmatter/step-template errors)
- `409 PKG_NAME_CONFLICT` (project name exists; `details` includes `conflicting_name` + `suggested_name`)
- `400 PKG_INVALID_ZIP` (bad zip)

### `GET /packages/import/check-name`

- Auth: required
- Query:
  - `name`: proposed project name

**Success (200)**
```json
{
  "data": {
    "exists": true,
    "suggested_name": "My Package-imported"
  },
  "error": null
}
```

## Import Pipeline (Backend)

Implementation: `crewagent-builder-backend/app/services/package_import_service.py`

1. **Extract ZIP to memory**
   - Zip bomb guard: total uncompressed size ≤ 100MB
   - Normalize nested root directory (strip common top-level folder)
2. **Validate required files**
   - `bmad.json`, `agents.json`
   - `workflow.graph.json` + `workflow.md` (single) OR `workflows/*/workflow.graph.json` (multi)
3. **Parse & validate JSON**
   - Strict JSON decoding (UTF-8)
   - `bmad.json.schemaVersion` must start with `1.1`
4. **Load workflows**
   - Multi-workflow: iterate `bmad.json.workflows[]` and load each `graph/workflow`
   - Single-workflow: resolve from `bmad.json.entry.*` with defaults
5. **Validate graph constraints**
   - `entryNodeId` exists
   - All edges reference existing nodes
   - Node step path exists and parses as markdown
6. **Validate step files**
   - YAML frontmatter required fields: `schemaVersion`, `nodeId`, `type`
   - Required sections: `## Goal`, `## Instructions` (Completion optional)
   - Errors include file path and approximate line pointer when possible
7. **Validate agent references**
   - For each step’s `agentId`, verify it exists in `agents.json`
8. **Collect assets**
   - `assets/**` stored in DB
   - Text stored as UTF-8 strings
   - Binary stored as `base64:` prefixed strings
9. **Create project + workflow records**
   - Create `workflow_packages` record (project)
   - Create `package_workflows` records for each imported workflow

## Frontend (Dashboard UI)

Implementation: `crewagent-builder-frontend/src/app/dashboard/page.tsx`

- Modal: select `.bmad` file and POST to `/packages/import`
- Shows:
  - progress strings (Uploading/Validating)
  - error summary and a structured list of validation issues when available
- Success: `router.push(/builder/{project_id})`

## Data Mapping

Stored in `workflow_packages` (project-level) and `package_workflows` (workflow-level):

- Project:
  - `name` ← `bmad.json.name` (or `custom_name`)
  - `agents_json` ← `agents.json.agents[]` (project-level agents)
  - `assets_json` ← `assets/**`
  - Backward compat (first workflow copied into legacy fields):
    - `workflow_md`, `graph_json`, `step_files_json`
- Workflows:
  - One record per imported workflow:
    - `name` ← `workflows[].id` (or derived)
    - `workflow_md`, `graph_json`, `step_files_json`

## Test Plan

- Backend (pytest):
  - invalid zip / too large / missing required files
  - invalid JSON / invalid schemaVersion
  - missing Goal/Instructions sections
  - invalid agentId references
  - name conflict and check-name endpoint
- Frontend (manual):
  - import valid package → redirect and workflows/agents present
  - import invalid package → validation list renders and remains actionable
