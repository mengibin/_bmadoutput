# Tech Spec: Story 4-10 Execute MCP Tool Calls

## 1. Architecture

### 1.1 Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CrewAgent Runtime                         │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                 ExecutionEngine                          ││
│  │  ┌───────────────┐    ┌────────────────────────────────┐││
│  │  │ FileSystem    │    │     McpClientService           │││
│  │  │ ToolHost      │    │  ┌──────────────────────────┐  │││
│  │  │ (fs.*)        │    │  │ Client Pool              │  │││
│  │  └───────────────┘    │  │ serverId -> McpClient    │  │││
│  │                       │  └──────────────────────────┘  │││
│  │  ┌───────────────┐    │  - connect()                   │││
│  │  │ Python        │    │  - listTools()                 │││
│  │  │ ToolHost      │    │  - callTool()                  │││
│  │  │ (python.*)    │    │  - disconnect()                │││
│  │  └───────────────┘    └────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
│                              │                               │
│                    JSON-RPC over stdio                       │
│                              ▼                               │
│  ┌─────────────────────────────────────────────────────────┐│
│  │         Bundled Node.js (resources/node/)               ││
│  │         └── npx @package/mcp-server-xxx                 ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Bundled Node.js Structure

Similar to the existing Python bundling approach:

```
resources/
├── python/                 # Existing (~50MB)
│   ├── bin/python3
│   └── lib/...
└── node/                   # NEW (~70MB)
    ├── bin/
    │   ├── node
    │   ├── npm
    │   └── npx
    ├── lib/
    │   └── node_modules/
    │       └── npm/
    └── cache/              # npm cache directory
```

### 1.3 Installed MCP Servers Structure

MCP servers are installed locally for offline use. Installation directory is the **application data directory**:

```
~/Library/Application Support/CrewAgent/   (macOS)
%APPDATA%/CrewAgent/                       (Windows)
├── node_modules/           # Installed npm MCP packages
│   └── @anthropic/
│       └── mcp-server-brave-search/
├── mcp-servers/            # Copied custom executables (entire directory)
│   └── my-custom-server/
│       ├── server.py
│       ├── config.yaml
│       └── lib/               # Dependencies copied with directory
└── mcp-config.json         # Installation metadata
```

**Design Clarifications:**
- **Python Dependencies**: Auto-install to bundled Python's `site-packages/` (allowed)
- **Custom Type**: Copy entire directory (including all dependencies)
- **npm Tarballs**: Support installing local `.tgz` tarballs for offline distribution
- **Version Updates**: No auto-update; users must uninstall and reinstall
- **Session Isolation**: Each Agent session gets its own MCP client instances (future multi-agent support)

### 1.4 New Services

**`McpClientService`** (new file: `src/main/services/mcpClientService.ts`):
- Manages a map of `serverId -> McpClient`.
- Handles process spawning using installed packages.
- Handles JSON-RPC over stdio.
- Exposes `listTools()` and `callTool()` methods.

**`McpInstallService`** (new file: `src/main/services/mcpInstallService.ts`):
- Handles package installation:
  - `installNpmPackage(packageName)` → installs to `RuntimeStore/node_modules/`
  - `installPythonPackage(moduleName)` → installs to bundled Python's `site-packages/`
  - `copyCustomServer(sourcePath, serverId)` → copies to `RuntimeStore/mcp-servers/`
- Tracks installation status and versions in `mcp-config.json`.
- Supports uninstallation and updates.

## 2. Data Structures

### 2.1 Configuration

Stored in `RuntimeSettings`. Supports three types of MCP servers:

```typescript
// Type 1: npm package (uses bundled Node.js/npx)
interface NpmMcpServerConfig {
  type: 'npm';
  id: string;           // e.g. "brave-search"
  name: string;         // Display name: "Brave Search"
  package: string;      // npm package: "@anthropic/mcp-server-brave-search"
  tarballPath?: string; // Optional local .tgz for offline install
  args?: string[];      // Additional args passed to the package
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;   // True if package is installed locally
  installedVersion?: string;  // e.g. "1.2.3"
}

// Type 2: Python module (uses bundled Python)
interface PythonMcpServerConfig {
  type: 'python';
  id: string;           // e.g. "git-server"
  name: string;         // Display name: "Git Server"
  module: string;       // Python module: "mcp_server_git"
  args?: string[];      // Additional args
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;   // True if module is installed locally
  installedVersion?: string;
}

// Type 3: Custom executable (direct command)
interface CustomMcpServerConfig {
  type: 'custom';
  id: string;           // e.g. "my-server"
  name: string;         // Display name: "My Custom Server"
  sourcePath?: string;  // Original path (for import)
  command: string;      // Installed path or original path
  args?: string[];      // Command arguments
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;   // True if copied to RuntimeStore
}

// Type 4: Remote server (SSE transport)
interface SseMcpServerConfig {
  type: 'sse';
  id: string;           // e.g. "company-search"
  name: string;         // Display name: "Company Search Service"
  url: string;          // e.g. "https://mcp.example.com/sse"
  headers?: Record<string, string>;  // Auth headers (e.g., Authorization)
  enabled: boolean;
}

type McpServerConfig = NpmMcpServerConfig | PythonMcpServerConfig | CustomMcpServerConfig | SseMcpServerConfig;

interface RuntimeSettings {
  // ... existing fields ...
  mcp: {
    servers: McpServerConfig[];
    globalEnabled: boolean;  // Master switch for all MCP
  };
}
```

### 2.2 Runtime State

```typescript
interface McpConnectionStatus {
  serverId: string;
  status: 'disconnected' | 'connecting' | 'connected' | 'error';
  error?: string;
  tools?: McpTool[];      // Discovered tools
  lastConnected?: Date;
}
```

### 2.3 Tool Naming Convention

To avoid collisions with static tools, MCP tools are prefixed:

```
mcp__<serverId>__<originalToolName>

Examples:
- mcp__brave_search__web_search
- mcp__git_server__git_status
- mcp__filesystem__read_file
```

## 3. Implementation Steps

### Phase 1: Bundled Node.js Setup

1. **Download portable Node.js** for each platform:
   - macOS x64: `node-v20.x.x-darwin-x64.tar.gz`
   - macOS arm64: `node-v20.x.x-darwin-arm64.tar.gz`
   - Windows x64: `node-v20.x.x-win-x64.zip`

2. **Configure electron-builder**:
   ```json5
   // electron-builder.json5
   {
     "extraResources": [
       {
         "from": "resources/node/${arch}/",
         "to": "node"
       }
     ]
   }
   ```

3. **Create `nodeService.ts`** (similar to `pythonService.ts`):
   ```typescript
   export function getNodePath(): string {
     const platform = process.platform;
     const resourcesPath = app.isPackaged 
       ? path.join(process.resourcesPath, 'node')
       : path.join(__dirname, '../../resources/node');
     
     if (platform === 'win32') {
       return path.join(resourcesPath, 'node.exe');
     }
     return path.join(resourcesPath, 'bin', 'node');
   }
   
   export function getNpxPath(): string {
     // Similar logic for npx
   }
   ```

### Phase 2: MCP Client Service

1. **Install MCP SDK** (if available) or implement lightweight JSON-RPC client:
   ```bash
   npm install @modelcontextprotocol/sdk
   ```

2. **Implement `McpClientService`**:
   ```typescript
   class McpClientService {
     private clients: Map<string, McpClient> = new Map();
     
     private resolveCommand(config: McpServerConfig): { command: string; args: string[] } {
       switch (config.type) {
         case 'npm':
           return {
             command: getNpxPath(),
             args: ['-y', config.package, ...(config.args ?? [])]
           };
         case 'python':
           return {
             command: getPythonPath(),
             args: ['-m', config.module, ...(config.args ?? [])]
           };
         case 'custom':
           return {
             command: config.command,
             args: config.args ?? []
           };
         case 'sse':
           // SSE uses HTTP transport, not stdio
           throw new Error('SSE transport does not use command spawning');
       }
     }
     
     // For SSE transport (remote servers)
     async connectSse(config: SseMcpServerConfig): Promise<void> {
       const client = new McpSseClient(config.url, config.headers);
       await client.connect();
       this.clients.set(config.id, client);
     }
     
     async connect(config: McpServerConfig): Promise<void> {
       if (config.type === 'sse') {
         return this.connectSse(config as SseMcpServerConfig);
       }
       
       const { command, args } = this.resolveCommand(config);
       const child = spawn(command, args, {
         stdio: ['pipe', 'pipe', 'pipe'],
         env: { ...process.env, ...config.env }
       });
       
       const client = new McpStdioClient(child.stdin, child.stdout);
       await client.initialize();
       this.clients.set(config.id, client);
     }
     
     async listTools(serverId: string): Promise<McpTool[]> {
       const client = this.clients.get(serverId);
       return client.listTools();
     }
     
     async callTool(serverId: string, name: string, args: any): Promise<any> {
       const client = this.clients.get(serverId);
       return client.callTool(name, args);
     }
   }
   ```

### Phase 3: Tool Registry Integration

1. **Update `PromptComposer`** to merge tools:
   ```typescript
   async getAvailableTools(): Promise<ToolDefinition[]> {
     const staticTools = this.getStaticTools(); // fs.*, python.*
     const mcpTools = await this.mcpService.getAllTools();
     
     return [
       ...staticTools,
       ...mcpTools.map(t => ({
         type: 'function',
         function: {
           name: `mcp__${t.serverId}__${t.name}`,
           description: t.description,
           parameters: t.inputSchema
         }
       }))
     ];
   }
   ```

2. **Update `ExecutionEngine`** to route MCP calls:
   ```typescript
   async executeToolCall(toolCall: ToolCall): Promise<ToolResult> {
     if (toolCall.name.startsWith('mcp__')) {
       const [, serverId, toolName] = toolCall.name.split('__');
       return this.mcpService.callTool(serverId, toolName, toolCall.args);
     }
     // ... existing static tool handling
   }
   ```

### Phase 4: Settings Integration

1. **Extend RuntimeSettings UI** (Story 5-7):
   - Add "MCP Servers" section
   - Add/Edit/Delete server configurations
   - Test connection button
   - Show connection status

## 4. File Changes Summary

| File | Change |
|:-----|:-------|
| `resources/node/` | **NEW**: Bundled Node.js |
| `src/main/services/nodeService.ts` | **NEW**: Node.js path resolution |
| `src/main/services/mcpClientService.ts` | **NEW**: MCP client manager |
| `src/main/engine/promptComposer.ts` | **MOD**: Merge MCP tools |
| `src/main/engine/executionEngine.ts` | **MOD**: Route MCP tool calls |
| `src/shared/types/runtimeStore.ts` | **MOD**: Add McpServerConfig |
| `electron-builder.json5` | **MOD**: Bundle Node.js |

## 5. Risks & Mitigations

| Risk | Level | Mitigation |
|:-----|:------|:-----------|
| Large bundle size (+70MB) | Medium | Accept for v1; consider lazy download later |
| Stdio parsing issues (stderr breaks JSON-RPC) | Medium | Separate stderr handling; use SDK |
| MCP server installation latency | Medium | Show progress indicator; cache installed packages |
| Environment PATH issues | Low | Use absolute paths from bundled Node.js |
| Platform-specific issues | Medium | Test on macOS x64, arm64, Windows |

## 6. Security Considerations

- MCP servers run with user privileges (same as Python scripts)
- No additional sandboxing beyond process isolation
- User must explicitly enable MCP servers in Settings
- API keys stored in Settings (same security model as LLM API keys)

## 7. Testing Strategy

### Unit Tests
- `McpClientService.connect()` with mock process
- Tool name parsing/routing logic
- Configuration validation

### Integration Tests
- Spawn actual Node.js process (bundled)
- Connect to a simple echo MCP server
- Execute tool call round-trip

### E2E Tests
- Configure MCP server in Settings
- Verify tools appear in LLM context
- Execute MCP tool through Agent conversation
