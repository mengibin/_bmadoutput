# Story 5-18: Python MCP Local Package Install & Run

## Overview
**Epic**: 5 - Observability, Settings & Recovery  
**Priority**: Medium  
**Status**: `done` (tested)

## Goal
Allow users to install a Python MCP server from a **local zip/wheel file** and run it via the configured module name, while still allowing pip to download dependencies online.

## Business Value
- **Offline distribution**: Users can distribute MCP servers as a single zip file and install locally.
- **Simple setup**: Non-technical users can install Python MCP without manual pip commands.
- **Consistent runtime**: Uses the bundled Python environment for installation and execution.

## Design Decisions
1. **Local Package Path for Python**:
   - Python server config supports an optional **Local package file** path (`.zip` / `.whl`).
   - Module name remains required and **does not need to match** the package name.
2. **Install Strategy**:
   - If a local package file is provided, runtime executes:
     - `pip install /path/to/package.zip` (or `.whl`)
   - No `--no-index` flag so dependencies can be downloaded online.
3. **Run Strategy**:
   - Runtime continues to spawn: `python -m <module>` using bundled Python.
   - The local package file is **not required** after installation completes.

## Acceptance Criteria

### 1. Python Server Form - Local Package Field
- **Given** I choose MCP server type = **Python**
- **When** I view the form
- **Then** I see a field: **Local package file (.zip/.whl, optional)**
- **And** I can still enter **Module** (required)

### 2. Install from Local Package
- **Given** I filled Module and Local package file
- **When** I click **Save & Install**
- **Then** runtime runs `pip install <local-file>` using bundled Python
- **And** install succeeds even if the module name differs from package name
- **And** dependencies may download online (no `--no-index`)

### 3. Run After Local Install
- **Given** installation succeeded
- **When** I click **Test Connection** or runtime auto-connects
- **Then** the MCP server starts via `python -m <module>`
- **And** tools/resources are discovered normally

### 4. Local Package File Cleanup
- **Given** installation succeeded
- **When** I delete or move the local zip/wheel
- **Then** the MCP server still runs (no dependency on the original file)
- **And** reinstall requires the file to be provided again

### 5. Validation & Errors
- **Given** the local package path is empty
- **Then** installation falls back to `pip install <packageName>` (from Module mapping)
- **Given** the local package path does not exist
- **Then** UI shows a clear error and install is blocked

## Design

### Summary
- Extend Python MCP server configuration to accept a local package file path.
- Update install pipeline to prefer local file when present.
- Keep execution flow unchanged (`python -m <module>`).

### UX / UI
- Python MCP form fields:
  - **Name** (required)
  - **Module** (required)
  - **Local package file (.zip/.whl, optional)**
  - **Arguments** (optional)
  - **Environment Variables** (optional)

### API / Contracts

**[MODIFY] McpServerConfig (Python)**
```ts
export interface PythonMcpServerConfig extends BaseMcpServerConfig {
  type: 'python'
  module: string
  packagePath?: string // local zip/wheel path
  args?: string[]
  env?: Record<string, string>
}
```

**[MODIFY] pythonService.pipInstall**
```ts
export async function pipInstall(
  packageNameOrPath: string,
  options?: { mirrorUrl?: string; timeout?: number; findLinks?: string }
): Promise<PipInstallResult>
```

### Data / Storage
- Persist `packagePath` in runtime settings for the MCP server config.

### Errors / Edge Cases
| Scenario | Feedback |
|:--|:--|
| Local file missing | "Local package file not found" |
| Install fails | Show pip output in error panel |
| Module mismatch | Still runs via `python -m <module>` |

### Test Plan
**Manual**
1. Add Python MCP with module + local zip -> Save & Install -> Test Connection succeeds.
2. Remove the local zip -> Test Connection still succeeds.
3. Local file path invalid -> Install shows error.
4. Module only (no local file) -> Falls back to online install.

**Unit/Integration**
- `pipInstall` adds `--find-links` or direct file path when provided.
- `mcpInstallService` prefers local package file for python type.
- Settings serialization round-trip for `packagePath`.

## Dependencies
- Story 5-7: View and Manage MCP Drivers (UI + install flow)
- Story 4-10: Execute MCP Tool Calls (runtime execution)
