# Design: Python MCP Local Package Install and Run

**Story:** `5-18-python-mcp-local-install-and-run.md`  
**Design principle:** Minimal change, reuse existing MCP modal

---

## Objectives

1. Let users install a Python MCP server from a local zip/wheel file.
2. Keep module-based execution unchanged (`python -m <module>`).
3. Allow online dependency downloads (no `--no-index`).

---

## Scope

### In scope
- Add an optional "Local package file (.zip/.whl)" field for Python MCP type.
- Use local file for installation when provided.
- Keep existing Save, Save and Install, Test Connection flows.

### Out of scope
- File picker UI (text input only).
- Package metadata parsing from zip.

---

## UI Changes

### Location
Settings -> MCP Servers -> Add/Edit MCP Server modal (Python type).

### Python Form Fields (updated)
- Name (required)
- Module (required)
- Local package file (.zip/.whl, optional)  <-- new
- Arguments (optional)
- Environment Variables (optional)

### Form Layout (Python type)

```
Name:  [............................]
Module: [mcp_server_git]
Local package file (.zip/.whl, optional):
        [/path/to/mcp-server-docx-0.1.0.zip]
Arguments (optional): [--port 8080]
Environment Variables: [key/value rows]
```

### Helper Copy
- Field hint: "If set, Runtime installs from this file. Dependencies can download online."
- Error (file missing): "Local package file not found."

---

## Behavior

1. **Validation**
   - Module is always required.
   - If Local package file is set:
     - Path must exist at install time.
2. **Save**
   - Save stores `packagePath` on the server config.
3. **Save and Install**
   - If `packagePath` is provided, install from the file.
   - Otherwise fall back to existing online install from module mapping.
4. **Test Connection**
   - Uses `python -m <module>` (unchanged).

---

## Data / Contracts

Add optional field on Python MCP config:
```
packagePath?: string
```

---

## Error States

- Missing local file: show install error panel with "Local package file not found."
- Install failure: show pip output (existing behavior).

---

## Affected Files (Design-Level)

- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` (form field + validation)
- `crewagent-runtime/src/types/mcp.ts` (config type)
- `crewagent-runtime/electron/stores/runtimeStore.ts` (normalize/serialize)
- `crewagent-runtime/electron/services/mcpInstallService.ts` (install selection)
- `crewagent-runtime/electron/services/pythonService.ts` (pipInstall options)
- `crewagent-runtime/electron/main.ts` (IPC install flow)

---

## Test Checklist (Manual)

1. Add Python MCP with module only -> Save and Install succeeds (online).
2. Add Python MCP with module + local zip -> Save and Install succeeds.
3. Remove the zip file -> Test Connection still succeeds.
4. Invalid local path -> Install shows clear error.
