# Story 5-7: View and Manage MCP Drivers

## Overview
**Epic**: 5 – Observability, Settings & Recovery
**Priority**: Medium
**Status**: `done` (tested)

## Goal
Provide a Settings UI for users to configure and manage MCP servers, enabling non-technical users to add MCP capabilities through simple configuration.

## Business Value
- **User-Friendly**: Non-technical users ("IT白痴") can add powerful capabilities without command-line knowledge.
- **Extensibility**: Users can add any MCP server (web search, git, databases, etc.) through configuration.
- **No Pre-installation Required**: Runtime bundles Node.js; users only need to provide package names.

## Design Decisions (Updated 2026-01-12)

1. **Four Server Types Supported**:
   - **npm**: npm packages (e.g., `@anthropic/mcp-server-brave-search`) - uses bundled Node.js
   - **python**: Python modules (e.g., `mcp_server_git`) - uses bundled Python
   - **custom**: Custom executables (e.g., `/path/to/my-server`)
   - **sse**: Remote servers via HTTP/SSE (e.g., `https://mcp.example.com/sse`)

2. **Automatic Installation**: When user adds an npm server, Runtime uses bundled npx to install and run the package.
   - Support installing from a local `.tgz` tarball for offline distribution.

3. **Connection Status**: Show real-time connection status for each server.

4. **Test Before Save**: "Test Connection" button to validate configuration before saving.

## Acceptance Criteria

### 1. View MCP Servers Section
- **Given** I am in Runtime Settings
- **When** I navigate to the "MCP Servers" section
- **Then** I see:
  - Master toggle: "Enable MCP" (on/off)
  - List of configured servers with status indicators
  - "Add Server" button

### 2. Add MCP Server
- **Given** I click "Add Server"
- **When** I fill in the configuration form
- **Then** the form includes:
  - Name (display name, required)
  - Package (npm package name, required)
  - Local tarball (optional, `.tgz` file path for offline install)
  - Environment Variables (key-value pairs, optional)
  - Enabled toggle (default: true)

### 3. Test Connection
- **Given** I have filled in server configuration
- **When** I click "Test Connection"
- **Then** Runtime attempts to spawn the MCP server using bundled Node.js
- **And** shows success/failure message with discovered tools count
- **And** shows error details if connection fails

### 4. Edit/Delete Server
- **Given** a configured MCP server exists
- **When** I click Edit or Delete
- **Then** I can modify configuration or remove the server
- **And** changes take effect immediately (reconnect if needed)

### 5. Connection Status Display
- **Given** MCP servers are configured
- **When** I view the server list
- **Then** each server shows current status:
  - 🔴 Disconnected
  - 🟡 Connecting
  - 🟢 Connected (with tools count)
  - ⚠️ Error (with error message)

### 6. Global MCP Toggle
- **Given** MCP is disabled globally
- **When** the LLM requests any `mcp://...` tool call
- **Then** the request is rejected with "MCP disabled" error
- **And** no process is spawned

### 7. Add Remote (SSE) Server
- **Given** I select "Remote" type when adding a server
- **When** I enter the server URL and optional auth headers
- **Then** Runtime connects via HTTP/SSE transport (no local process spawned)
- **And** I can test the connection and see discovered tools
- **And** the server shows [remote] tag in the list

### 8. Install MCP Server Locally
- **Given** I have added an npm, python, or custom MCP server
- **When** I click "Install"
- **Then** Runtime:
  - npm → runs `npm install <package>` or `npm install <path-to.tgz>` into `RuntimeStore/node_modules/`
  - python → runs `pip install <module>` into bundled Python
  - custom → copies executable files to `RuntimeStore/mcp-servers/`
- **And** shows installation progress and success/failure message
- **And** the server shows "Installed (v1.2.3)" status after success
- **And** the server works offline after installation

## Design

### Summary
- Settings UI for MCP server configuration with 4 server types
- Install button for local installation (offline capability)
- Real-time connection status display
- Test connection before save
- Tech Spec: [tech-spec-5-7-view-and-manage-mcp-drivers.md](file:///Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-7-view-and-manage-mcp-drivers.md)

### UX / UI

**Settings Page Layout:**
1. **MCP Servers Section** - collapsible section with global toggle
2. **Server List** - cards showing each server with status, type badge, edit/delete buttons
3. **Add/Edit Modal** - type selector (npm/python/custom/remote) with conditional fields
4. **Status Indicators** - 🟢 connected, 🟡 connecting, 🔴 disconnected, ⚠️ error

**User Flow:**
1. Open Settings → MCP Servers
2. Click "Add Server" → Select type → Fill fields
3. Click "Install" (for local types) → Show progress
4. Click "Test Connection" → Show result
5. Click "Save" → Server appears in list

### API / Contracts

**Preload API:**
```typescript
window.api.mcp = {
  getServers(): Promise<McpServerConfig[]>;
  saveServer(config: McpServerConfig): Promise<{ success: boolean }>;
  deleteServer(id: string): Promise<{ success: boolean }>;
  testConnection(config: Partial<McpServerConfig>): Promise<TestResult>;
  install(serverId: string): Promise<{ success: boolean; version?: string }>;
  uninstall(serverId: string): Promise<{ success: boolean }>;
  getStatus(): Promise<Record<string, McpConnectionStatus>>;
  onStatusChange(callback): void;
  onInstallProgress(callback): void;
};
```

**IPC Channels:**
- `mcp:getServers`, `mcp:saveServer`, `mcp:deleteServer`
- `mcp:testConnection`, `mcp:install`, `mcp:uninstall`
- `mcp:getStatus`, `mcp:statusChange` (event), `mcp:installProgress` (event)

### Data / Storage

**RuntimeSettings (electron-store):**
```typescript
mcp: {
  globalEnabled: boolean;
  servers: McpServerConfig[];
}
```

**React State:**
```typescript
const { servers, status, loading, saveServer, deleteServer, testConnection, install } = useMcpSettings();
```

### Errors / Edge Cases

| Scenario | User Feedback |
|:---------|:--------------|
| Invalid package name | "Could not find package. Check the package name." |
| Network error during install | "Network error. Check your connection." |
| Server crash | Show ⚠️ status with error message |
| Missing env var | "Server requires BRAVE_API_KEY" |
| Test connection timeout | "Connection timed out" |
| Duplicate server name | "Server with this name already exists" |

### Test Plan

**Unit Tests:**
- Form validation (required fields, URL format)
- Status indicator rendering for each state
- Environment variable editor CRUD

**Integration Tests:**
- Save/load server configuration via IPC
- Test connection flow (success/failure)
- Install/uninstall flow

**E2E Tests:**
- Add npm server → Install → Test → Save → Verify in list
- Add SSE server → Test → Save → Verify [remote] tag
- Edit server → Reconnect → Verify new config
- Delete server → Verify removed
- Global toggle → Verify tools availability

## UI Mockup

```
┌────────────────────────────────────────────────────────────┐
│  Settings                                                   │
├────────────────────────────────────────────────────────────┤
│  MCP Servers                                         [ON/OFF│
│  ┌──────────────────────────────────────────────────────── │
│  │ + Add Server                                             │
│  ├──────────────────────────────────────────────────────── │
│  │ 🟢 Brave Search              [npm]           [Edit][Del] │
│  │    @anthropic/mcp-server-brave-search                    │
│  │    ✅ Installed (v1.2.3) · 5 tools                         │
│  ├──────────────────────────────────────────────────────── │
│  │ 🟢 Git Server                [python]        [Edit][Del] │
│  │    mcp_server_git                                        │
│  │    ✅ Installed (v0.8.1) · 8 tools                         │
│  ├──────────────────────────────────────────────────────── │
│  │ ⚠️ Custom Server             [custom]        [Edit][Del] │
│  │    /path/to/my-server                                    │
│  │    ❌ Not installed · [Install]                            │
│  └──────────────────────────────────────────────────────── │
└────────────────────────────────────────────────────────────┘

┌─ Add MCP Server ───────────────────────────────────────────┐
│                                                             │
│  Type:  [● npm] [○ Python] [○ Custom] [○ Remote]            │
│                                                             │
│  ─── npm Mode ───                                           │
│  Name:     [Brave Search                     ]              │
│  Package:  [@anthropic/mcp-server-brave-search]             │
│                                                             │
│  ─── Python Mode ───                                        │
│  Name:     [Git Server                       ]              │
│  Module:   [mcp_server_git                   ]              │
│                                                             │
│  ─── Custom Mode ───                                        │
│  Name:     [My Server                        ]              │
│  Command:  [/path/to/my-server               ]              │
│  Args:     [--port 8080                      ]              │
│                                                             │
│  ─── Remote Mode (SSE) ───                                  │
│  Name:     [Company Search                   ]              │
│  URL:      [https://mcp.example.com/sse      ]              │
│  Headers:  (optional auth headers)                          │
│                                                             │
│  Environment Variables:                                     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ BRAVE_API_KEY  │ sk-xxxxxxxxxxxx           │ [×]    │   │
│  └─────────────────────────────────────────────────────┘   │
│  [+ Add Variable]                                           │
│                                                             │
│  [Test Connection]                    [Cancel]  [Save]      │
└─────────────────────────────────────────────────────────────┘
```

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `src/renderer/pages/SettingsPage.tsx` | **MOD**: Add MCP Servers section |
| `src/renderer/components/McpServerForm.tsx` | **NEW**: Server configuration form |
| `src/renderer/components/McpServerList.tsx` | **NEW**: Server list with status |
| `src/shared/types/runtimeStore.ts` | **MOD**: Add McpServerConfig type |
| `src/main/services/mcpClientService.ts` | **MOD**: Add testConnection() method |
| `src/main/ipc/handlers/settings.ts` | **MOD**: Add MCP settings handlers |

## Dependencies
- Story 4-10: Execute MCP Tool Calls - *MCP Client core implementation*
- Story 5-5: Settings Page - *Base settings infrastructure* ✅

## Verification Plan

### Manual Verification
1. Navigate to Settings → MCP Servers
2. Click "Add Server" and enter a valid package (e.g., `@modelcontextprotocol/server-filesystem`)
3. Click "Test Connection" - should show success with tools count
4. Save and verify server appears in list with 🟢 status
5. Start a conversation and verify MCP tools appear in LLM context
6. Edit server configuration and verify changes take effect
7. Delete server and verify it's removed
8. Disable global MCP toggle and verify tools are not available

### Edge Cases
- Invalid package name → Show clear error
- Network failure during npx install → Show error, allow retry
- Server crashes during operation → Show error status, allow reconnect
- Missing API key (if required) → Server fails with helpful error
