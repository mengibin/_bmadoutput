# Tech Spec: Story 5-18 Python MCP Local Package Install and Run

## 1. Overview

This story adds support for installing a Python MCP server from a local zip or wheel file while keeping the existing module-based execution flow. Dependencies may still download online because we do not add `--no-index`.

## 2. Data Model Changes

### 2.1 Runtime (Shared Type)
Update Python MCP config to include an optional local package path.

```ts
export interface PythonMcpServerConfig extends BaseMcpServerConfig {
  type: 'python'
  module: string
  packagePath?: string
  args?: string[]
  env?: Record<string, string>
}
```

**Files**
- `crewagent-runtime/src/types/mcp.ts`
- `crewagent-runtime/electron/services/mcpClientService.ts`

### 2.2 Settings Normalization
Persist and normalize `packagePath` for Python MCP servers.

**File**
- `crewagent-runtime/electron/stores/runtimeStore.ts`

## 3. UI Changes

### 3.1 MCP Server Modal (Python Type)
Add a new field under Module:

Label: `Local package file (.zip/.whl, optional)`  
Placeholder: `/path/to/mcp-server-docx-0.1.0.zip`  
Hint: `If set, Runtime installs from this file. Dependencies can download online.`

**File**
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

### 3.2 Form Validation
- Module remains required.
- `packagePath` is optional and only validated at install time (renderer has no direct filesystem access).

## 4. Install Flow

### 4.1 Install Selection (Python)
In `McpInstallService.installPythonModule`:

1. If `config.packagePath` is provided:
   - Ensure the file exists.
   - Run `pip install <packagePath>`.
2. Else:
   - Use current behavior: `pip install <module>` (module name converted to package name with underscore -> hyphen).

**Files**
- `crewagent-runtime/electron/services/mcpInstallService.ts`
- `crewagent-runtime/electron/services/pythonService.ts` (no new API required; still `pipInstall(string)`)

### 4.2 Error Handling
- If local file is missing, return `{ ok: false, error: 'Local package file not found: <path>' }`.
- UI surfaces this via existing install error panel.

## 5. Execution Flow (Unchanged)

No change to runtime execution:
`python -m <module>` remains the launch mechanism.

**File**
- `crewagent-runtime/electron/services/mcpClientService.ts`

## 6. IPC / Persistence

No new IPC channels. Existing `mcp:saveServer` and `mcp:install` flows continue to work.

Ensure `packagePath` is preserved in settings serialization.

**Files**
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`

## 7. Affected Files Summary

- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/types/mcp.ts`
- `crewagent-runtime/electron/services/mcpClientService.ts`
- `crewagent-runtime/electron/services/mcpInstallService.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`

## 8. Test Plan

### Manual
1. Add Python MCP with module only -> Save and Install succeeds (online).
2. Add Python MCP with module + local zip -> Save and Install succeeds.
3. Remove the zip file -> Test Connection still succeeds.
4. Invalid local path -> Install shows clear error.

### Unit / Integration
- `normalizeMcpServer` round-trip includes `packagePath`.
- `installPythonModule` prefers `packagePath` when present.
