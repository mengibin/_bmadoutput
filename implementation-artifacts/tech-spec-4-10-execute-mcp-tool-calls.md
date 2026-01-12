# Tech Spec: Story 4-10 Execute MCP Tool Calls

## 1. Architecture

We will implement a new valid `ToolHost` implementation called `McpToolHost` (or extend `FileSystemToolHost` to support MCP namespaces), but more cleanly, we should introduce a `McpClientService` that manages persistent connections to MCP servers.

### 1.1 New Services
- **`McpClientService`**:
    - Manages a map of `serverId -> McpClient`.
    - Handles process spawning (via `child_process.spawn`).
    - Handles JSON-RPC over stdio.
    - Exposes `listTools()` and `callTool()` methods.

### 1.2 Integration with ExecutionEngine
The `ToolHost` concept in the Runtime currently maps `toolName` -> `function`.
We need to augment `ExecutionEngine` or `ToolHost` to:
1.  Initialize configured MCP clients at startup (or on demand).
2.  Aggregate tools from all MCP clients into the tool definition list sent to the LLM.
3.  Route calls like `mcp__<serverId>__<toolName>` to the correct client. (Namespace strategy to avoid collisions).

## 2. Data Structures

### 2.1 Configuration
Stored in `RuntimeStore` (Settings) or `bmad.json` (Project-specific). For now, we might hardcode or use a simple config file `mcp-servers.json` in the user appData.

```typescript
type McpServerConfig = {
  id: string;       // e.g. "fs-server"
  command: string;  // e.g. "npx"
  args: string[];   // e.g. ["-y", "@modelcontextprotocol/server-filesystem", "/Users/me"]
  env?: Record<string, string>;
};
```

### 2.2 Runtime Store Updates
- `mcpServers`: `Record<string, McpServerConfig>`
- `mcpConnectionStatus`: `Record<string, 'connecting' | 'connected' | 'error'>`

## 3. Implementation Steps

1.  **Dependencies**: Install `@modelcontextprotocol/sdk` (if available for Node) or implement a lightweight JSON-RPC 2.0 client over Stdio. *Note: The official SDK is likely the best path.*
2.  **`McpClientService.ts`**:
    - `connect(config: McpServerConfig): Promise<void>`
    - `getTools(): Promise<Tool[]>`
    - `callTool(name: string, args: any): Promise<any>`
3.  **`ExecutionExtensions`**:
    - Modify `executionEngine.ts` to merge static tools (like `read_file` native) with dynamic MCP tools.
    - Add logic to prefix tool names (e.g., `filesystem_list_files` if the server is `filesystem`) to prevent collisions, or just trust the server's tool names if unique.
4.  **UI Updates** (Scope of Story 5-7, but minimal logs needed here):
    - Log MCP connection events to the Execution Log.

## 4. Risks & Unknowns
- **Stdio Parsing**: Ensuring `stderr` from the server doesn't break the JSON-RPC parsing on `stdout`.
- **Latency**: Tool discovery on every run vs caching.
- **Environment**: Ensuring `npx` or python executables are found in the Electron environment's PATH.

## 5. Security
- **Sandbox**: MCP servers run with the user's full privileges if spawned locally. We must warn the user (future refined in Security Policy).
