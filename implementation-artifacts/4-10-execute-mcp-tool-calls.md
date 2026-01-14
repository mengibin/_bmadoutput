# Story 4-10: Execute MCP Tool Calls (Stdio Driver)

## Overview
**Epic**: 4 – Execution Core
**Priority**: High
**Status**: `done` (tested)

## Goal
Enable the Runtime to execute tools provided by Model Context Protocol (MCP) servers using the `stdio` transport. This allows the Agent to interact with external systems (filesystem, git, web search, etc.) via standard MCP servers.

## Business Value
- **Extensibility**: Instantly unlocks a vast ecosystem of existing MCP servers (e.g., brave-search, filesystem, git).
- **Standardization**: Replaces ad-hoc tool implementations with a standard protocol.
- **User-Friendly**: Non-technical users can add MCP capabilities through simple configuration.

## Design Decisions (Updated 2026-01-12)

Based on user requirements:

1. **Four Server Types Supported**:
   - **npm**: npm packages (e.g., `@anthropic/mcp-server-brave-search`) - uses bundled Node.js
   - **python**: Python modules (e.g., `mcp_server_git`) - uses bundled Python
   - **custom**: Custom executables (e.g., `/path/to/my-server`)
   - **sse**: Remote servers via HTTP/SSE (e.g., `https://mcp.example.com/sse`)

2. **Install-to-Embed (Local Installation)**:
   - npm packages are installed into `~/Library/Application Support/CrewAgent/node_modules/`
   - npm installs may use a local `.tgz` tarball (offline distribution)
   - Python modules are installed into bundled Python's `site-packages/` (auto-install dependencies allowed)
   - Custom executables are copied **entirely** (whole directory) to `mcp-servers/`
   - After installation, MCP servers work **offline** (except SSE)
   - No auto-update; users must uninstall and reinstall for new versions

3. **Configuration-Driven**: Users configure MCP servers via Settings UI (Story 5-7). No pre-installed MCP servers.

4. **Dynamic Tool Discovery**: MCP tools are discovered at runtime and dynamically added to the LLM's tool definitions.

5. **Target Users**: Non-technical users ("IT白痴") who cannot configure complex environments.

## Acceptance Criteria

### 1. MCP Client Integration
- **Given** a configured MCP server in Settings
- **When** the Runtime starts a conversation/workflow
- **Then** it establishes a connection to the MCP server via stdio (using bundled Node.js).

### 2. Tool Discovery
- **Given** a connected MCP server
- **When** the Agent session starts
- **Then** the Runtime calls `listTools()` and merges MCP tools with static tools.
- **And** the system prompt includes MCP tool descriptions.

### 3. Tool Execution
- **Given** the LLM requests an MCP tool call
- **When** the Runtime receives the tool call
- **Then** it:
    1. Routes the call to the correct MCP client (by server id prefix).
    2. Translates to MCP `CallToolRequest`.
    3. Receives the `CallToolResult`.
    4. Returns the result to the LLM.

### 4. Error Handling
- **Given** an MCP tool call fails (stderr output, protocol error, or server crash)
- **When** the error occurs
- **Then** the Runtime catches the error and returns it as tool output to the LLM (not crash Runtime).

### 5. Bundled Runtimes
- **Given** the Runtime is installed on a fresh macOS/Windows machine
- **When** the user configures an MCP server (npm or python type)
- **Then** Runtime uses bundled Node.js/Python to run the server without requiring user installation.

### 6. SSE/Remote Server
- **Given** the user configures a remote MCP server (SSE type)
- **When** the Runtime connects
- **Then** it uses HTTP/SSE transport instead of stdio
- **And** no local process is spawned

### 7. Local Installation (Install-to-Embed)
- **Given** the user adds an npm or python MCP server
- **When** they click "Install"
- **Then** Runtime downloads and installs the package into local storage:
  - npm → `RuntimeStore/node_modules/<package>/` (from registry or local `.tgz`)
  - python → bundled Python's `site-packages/`
  - custom → `RuntimeStore/mcp-servers/<serverId>/` (copy files)
- **And** the server works offline after installation (no network required)

## Design

### Summary
- MCP Client integration via `McpClientService` managing stdio/SSE connections to MCP servers
- Four server types: npm, python, custom, SSE (remote)
- Install-to-embed approach for offline capability
- Dynamic tool discovery with namespaced tool naming (`mcp__<serverId>__<toolName>`)
- Tech Spec: [tech-spec-4-10-execute-mcp-tool-calls.md](file:///Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-4-10-execute-mcp-tool-calls.md)

### UX / UI
- N/A (backend story; UI handled in Story 5-7)
- Tool execution transparent to user - tools appear in LLM context automatically

### API / Contracts

**McpClientService API:**
```typescript
interface McpClientService {
  connect(config: McpServerConfig): Promise<void>;
  disconnect(serverId: string): Promise<void>;
  listTools(serverId: string): Promise<McpTool[]>;
  callTool(serverId: string, toolName: string, args: any): Promise<ToolResult>;
  getAllTools(): Promise<McpTool[]>;
  getStatus(serverId: string): McpConnectionStatus;
  testConnection(config: Partial<McpServerConfig>): Promise<TestResult>;
}
```

**Tool Call Routing:**
- Tool name format: `mcp__<serverId>__<originalToolName>`
- ExecutionEngine parses prefix, routes to correct McpClient

### Data / Storage

**Installation Directory** (Application Data):
```
~/Library/Application Support/CrewAgent/   (macOS)
%APPDATA%/CrewAgent/                       (Windows)
```

**mcp-config.json:**
```json
{
  "installedPackages": {
    "@anthropic/mcp-server-brave-search": { "version": "1.2.3", "installedAt": "2026-01-12" }
  }
}
```

**Installation directories:**
- npm: `<appData>/node_modules/<package>/`
- python: bundled Python `site-packages/` (with dependencies)
- custom: `<appData>/mcp-servers/<serverId>/` (entire directory copied)

**Session Isolation:**
- Each Agent session spawns its own MCP client instances
- Prepared for future multi-agent support

### Errors / Edge Cases

| Error | Handling |
|:------|:---------|
| Server not installed | Return error to LLM: "MCP server not installed" |
| Connection timeout | Return error to LLM, set status to 'error' |
| Server crash mid-call | Catch error, return to LLM, attempt reconnect |
| Invalid tool name | Return error to LLM: "Unknown tool" |
| MCP globally disabled | Reject tool call with "MCP disabled" |
| stdio parse error | Log stderr, return error to LLM |
| SSE connection lost | Auto-reconnect with exponential backoff |

### Test Plan

**Unit Tests:**
- `McpClientService.connect()` with mock process
- Tool name parsing and routing logic
- Configuration validation for all 4 types
- Error handling for each error scenario

**Integration Tests:**
- Spawn bundled Node.js, connect to echo MCP server
- Execute tool call round-trip
- SSE client connection and message parsing

**E2E Tests:**
- Install npm MCP package, verify in node_modules
- Configure server in Settings, verify tools in LLM context
- Execute tool via Agent conversation, verify results

## Technical Components / Changes

1. **`resources/node/`**: Bundled portable Node.js (similar to `resources/python/`).

2. **`McpClientService`**: 
   - Manages a map of `serverId -> McpClient`.
   - Handles process spawning via bundled Node.js.
   - Handles JSON-RPC over stdio.
   - Exposes `listTools()` and `callTool()` methods.

3. **`ToolRegistry` Extension**: 
   - Support "dynamic" tools from MCP clients.
   - Tool naming: `mcp__<serverId>__<toolName>` (namespace strategy).

4. **`PromptComposer` Update**:
   - Aggregate static tools + MCP tools.
   - Generate tool descriptions for system prompt.

5. **`RuntimeSettings` Extension**:
   - `mcpServers`: Array of configured MCP servers.
   - `mcpConnectionStatus`: Connection state per server.

## Dependencies
- Story 4-6: Execute Tool Calls (Sandboxed Filesystem) - *Foundation for tool execution* ✅
- Story 5-5: Settings Page - *Base settings infrastructure* ✅
- Story 5-7: View and Manage MCP Drivers - *Settings UI for MCP configuration*

## Verification Plan

### Manual Verification
1. **Setup**: Configure a simple MCP server in Settings (e.g., `@anthropic/mcp-server-brave-search`).
2. **Discovery**: Start a chat. Check that MCP tools appear in the tool definitions.
3. **Execution**: Ask the agent to use the tool (e.g., "Search for latest AI news").
4. **Result**: Verify the agent sees the real search results.

### Automated Tests
- Unit tests for `McpClientService` (mocking stdio transport).
- Integration test: Spawn a dummy node process acting as an MCP server and perform Request/Response cycle.
- E2E test: Verify bundled Node.js can successfully run `npx` commands.
