# Tech Spec: Story 5-7 View and Manage MCP Drivers

## 1. Overview

This story implements the Settings UI for MCP server configuration. It depends on Story 4-10 (McpClientService) for the backend functionality.

## 2. UI Components

### 2.1 Settings Page Extension

Location: `src/renderer/pages/SettingsPage.tsx`

Add new section after "Python" settings:

```tsx
<SettingsSection title="MCP Servers" icon={<PlugIcon />}>
  <McpGlobalToggle />
  <McpServerList />
</SettingsSection>
```

### 2.2 McpGlobalToggle

```tsx
interface McpGlobalToggleProps {
  enabled: boolean;
  onChange: (enabled: boolean) => void;
}

function McpGlobalToggle({ enabled, onChange }: McpGlobalToggleProps) {
  return (
    <SettingsRow 
      label="Enable MCP" 
      description="Allow agent to use external MCP tools"
    >
      <Switch checked={enabled} onCheckedChange={onChange} />
    </SettingsRow>
  );
}
```

### 2.3 McpServerList

```tsx
function McpServerList() {
  const { mcpServers, mcpStatus } = useRuntimeSettings();
  
  return (
    <div className="mcp-server-list">
      <Button onClick={handleAddServer}>
        <PlusIcon /> Add Server
      </Button>
      
      {mcpServers.map(server => (
        <McpServerCard 
          key={server.id}
          server={server}
          status={mcpStatus[server.id]}
          onEdit={() => handleEdit(server)}
          onDelete={() => handleDelete(server.id)}
        />
      ))}
    </div>
  );
}
```

### 2.4 McpServerCard

```tsx
interface McpServerCardProps {
  server: McpServerConfig;
  status: McpConnectionStatus;
  onEdit: () => void;
  onDelete: () => void;
}

function McpServerCard({ server, status, onEdit, onDelete }: McpServerCardProps) {
  return (
    <Card>
      <div className="flex items-center gap-2">
        <StatusIndicator status={status.status} />
        <div className="flex-1">
          <h4>{server.name}</h4>
          <p className="text-sm text-muted">{server.package}</p>
          <StatusMessage status={status} />
        </div>
        <Button variant="ghost" size="sm" onClick={onEdit}>Edit</Button>
        <Button variant="ghost" size="sm" onClick={onDelete}>Delete</Button>
      </div>
    </Card>
  );
}

function StatusIndicator({ status }: { status: string }) {
  const colors = {
    connected: 'bg-green-500',
    connecting: 'bg-yellow-500',
    disconnected: 'bg-gray-500',
    error: 'bg-red-500'
  };
  return <div className={`w-3 h-3 rounded-full ${colors[status]}`} />;
}
```

### 2.5 McpServerForm (Modal)

```tsx
interface McpServerFormProps {
  server?: McpServerConfig;  // undefined for new, defined for edit
  onSave: (config: McpServerConfig) => Promise<void>;
  onCancel: () => void;
}

function McpServerForm({ server, onSave, onCancel }: McpServerFormProps) {
  const [type, setType] = useState<'npm' | 'python' | 'custom' | 'sse'>(server?.type ?? 'npm');
  const [name, setName] = useState(server?.name ?? '');
  
  // Type-specific fields
  const [pkg, setPackage] = useState((server as NpmMcpServerConfig)?.package ?? '');
  const [tarball, setTarball] = useState((server as NpmMcpServerConfig)?.tarballPath ?? '');
  const [module, setModule] = useState((server as PythonMcpServerConfig)?.module ?? '');
  const [command, setCommand] = useState((server as CustomMcpServerConfig)?.command ?? '');
  const [args, setArgs] = useState((server as CustomMcpServerConfig)?.args?.join(' ') ?? '');
  const [url, setUrl] = useState((server as SseMcpServerConfig)?.url ?? '');
  const [headers, setHeaders] = useState<Record<string, string>>((server as SseMcpServerConfig)?.headers ?? {});
  
  const [envVars, setEnvVars] = useState<Record<string, string>>(server?.env ?? {});
  const [testing, setTesting] = useState(false);
  const [testResult, setTestResult] = useState<TestResult | null>(null);
  
  const handleTestConnection = async () => {
    setTesting(true);
    try {
      const config = buildConfig();
      const result = await window.api.mcp.testConnection(config);
      setTestResult(result);
    } finally {
      setTesting(false);
    }
  };
  
  const buildConfig = (): McpServerConfig => {
    const base = { id: server?.id ?? uuid(), name, enabled: true };
    switch (type) {
      case 'npm':
        return { ...base, type: 'npm', package: pkg, tarballPath: tarball || undefined, env: envVars };
      case 'python':
        return { ...base, type: 'python', module, env: envVars };
      case 'custom':
        return { ...base, type: 'custom', command, args: args.split(' ').filter(Boolean), env: envVars };
      case 'sse':
        return { ...base, type: 'sse', url, headers };
    }
  };
  
  return (
    <Dialog>
      <DialogHeader>
        <DialogTitle>{server ? 'Edit' : 'Add'} MCP Server</DialogTitle>
      </DialogHeader>
      
      <DialogContent>
        <Label>Type</Label>
        <RadioGroup value={type} onValueChange={setType}>
          <RadioGroupItem value="npm">npm Package</RadioGroupItem>
          <RadioGroupItem value="python">Python Module</RadioGroupItem>
          <RadioGroupItem value="custom">Custom Executable</RadioGroupItem>
          <RadioGroupItem value="sse">Remote (SSE)</RadioGroupItem>
        </RadioGroup>
        
        <Label>Name</Label>
        <Input value={name} onChange={e => setName(e.target.value)} />
        
        {/* Type-specific fields */}
        {type === 'npm' && (
          <>
            <Label>Package</Label>
            <Input value={pkg} onChange={e => setPackage(e.target.value)}
              placeholder="@anthropic/mcp-server-brave-search" />
            <Label>Local tarball (optional)</Label>
            <Input value={tarball} onChange={e => setTarball(e.target.value)}
              placeholder="/path/to/mcp-server.tgz" />
          </>
        )}
        
        {type === 'python' && (
          <>
            <Label>Module</Label>
            <Input value={module} onChange={e => setModule(e.target.value)}
              placeholder="mcp_server_git" />
          </>
        )}
        
        {type === 'custom' && (
          <>
            <Label>Command</Label>
            <Input value={command} onChange={e => setCommand(e.target.value)}
              placeholder="/path/to/server" />
            <Label>Arguments (space-separated)</Label>
            <Input value={args} onChange={e => setArgs(e.target.value)}
              placeholder="--port 8080" />
          </>
        )}
        
        {type === 'sse' && (
          <>
            <Label>URL</Label>
            <Input value={url} onChange={e => setUrl(e.target.value)}
              placeholder="https://mcp.example.com/sse" />
            <Label>Auth Headers (optional)</Label>
            <KeyValueEditor value={headers} onChange={setHeaders} />
          </>
        )}
        
        {type !== 'sse' && (
          <>
            <Label>Environment Variables</Label>
            <EnvVarsEditor value={envVars} onChange={setEnvVars} />
          </>
        )}
        
        {testResult && <TestResultDisplay result={testResult} />}
      </DialogContent>
      
      <DialogFooter>
        <Button variant="outline" onClick={handleTestConnection} disabled={testing}>
          {testing ? <Spinner /> : 'Test Connection'}
        </Button>
        <Button variant="ghost" onClick={onCancel}>Cancel</Button>
        <Button onClick={() => onSave(buildConfig())}>Save</Button>
      </DialogFooter>
    </Dialog>
  );
}
```

## 3. IPC Handlers

### 3.1 Main Process Handlers

Location: `src/main/ipc/handlers/mcpHandlers.ts`

```typescript
// Register handlers
ipcMain.handle('mcp:getServers', async () => {
  return runtimeStore.get('mcp.servers', []);
});

ipcMain.handle('mcp:saveServer', async (_, config: McpServerConfig) => {
  const servers = runtimeStore.get('mcp.servers', []);
  const existing = servers.findIndex(s => s.id === config.id);
  
  if (existing >= 0) {
    servers[existing] = config;
  } else {
    servers.push(config);
  }
  
  runtimeStore.set('mcp.servers', servers);
  
  // Reconnect if enabled
  if (config.enabled) {
    await mcpClientService.reconnect(config.id);
  }
  
  return { success: true };
});

ipcMain.handle('mcp:deleteServer', async (_, serverId: string) => {
  await mcpClientService.disconnect(serverId);
  
  const servers = runtimeStore.get('mcp.servers', []);
  runtimeStore.set('mcp.servers', servers.filter(s => s.id !== serverId));
  
  return { success: true };
});

ipcMain.handle('mcp:testConnection', async (_, config: Partial<McpServerConfig>) => {
  try {
    const tools = await mcpClientService.testConnection(config);
    return { 
      success: true, 
      toolsCount: tools.length,
      tools: tools.map(t => ({ name: t.name, description: t.description }))
    };
  } catch (error) {
    return { 
      success: false, 
      error: error.message 
    };
  }
});

ipcMain.handle('mcp:getStatus', async () => {
  return mcpClientService.getAllStatus();
});

// Installation handlers
ipcMain.handle('mcp:install', async (_, serverId: string) => {
  const servers = runtimeStore.get('mcp.servers', []);
  const server = servers.find(s => s.id === serverId);
  
  if (!server) {
    return { success: false, error: 'Server not found' };
  }
  
  try {
    const result = await mcpInstallService.install(server);
    // Update installed status
    server.installed = true;
    server.installedVersion = result.version;
    runtimeStore.set('mcp.servers', servers);
    return { success: true, version: result.version };
  } catch (error) {
    return { success: false, error: error.message };
  }
});

ipcMain.handle('mcp:uninstall', async (_, serverId: string) => {
  const servers = runtimeStore.get('mcp.servers', []);
  const server = servers.find(s => s.id === serverId);
  
  if (!server) {
    return { success: false, error: 'Server not found' };
  }
  
  try {
    await mcpInstallService.uninstall(server);
    server.installed = false;
    server.installedVersion = undefined;
    runtimeStore.set('mcp.servers', servers);
    return { success: true };
  } catch (error) {
    return { success: false, error: error.message };
  }
});
```

### 3.2 Preload API

Location: `src/preload/index.ts`

```typescript
contextBridge.exposeInMainWorld('api', {
  // ... existing APIs ...
  mcp: {
    getServers: () => ipcRenderer.invoke('mcp:getServers'),
    saveServer: (config: McpServerConfig) => ipcRenderer.invoke('mcp:saveServer', config),
    deleteServer: (id: string) => ipcRenderer.invoke('mcp:deleteServer', id),
    testConnection: (config: Partial<McpServerConfig>) => ipcRenderer.invoke('mcp:testConnection', config),
    getStatus: () => ipcRenderer.invoke('mcp:getStatus'),
    install: (serverId: string) => ipcRenderer.invoke('mcp:install', serverId),
    uninstall: (serverId: string) => ipcRenderer.invoke('mcp:uninstall', serverId),
    onStatusChange: (callback: (status: Record<string, McpConnectionStatus>) => void) => {
      ipcRenderer.on('mcp:statusChange', (_, status) => callback(status));
    },
    onInstallProgress: (callback: (progress: { serverId: string; percent: number; message: string }) => void) => {
      ipcRenderer.on('mcp:installProgress', (_, progress) => callback(progress));
    }
  }
});
```

## 4. State Management

### 4.1 RuntimeStore Extension

Location: `src/shared/types/runtimeStore.ts`

```typescript
interface RuntimeSettings {
  // ... existing fields ...
  mcp: {
    globalEnabled: boolean;
    servers: McpServerConfig[];
  };
}

// Three types of MCP servers
interface NpmMcpServerConfig {
  type: 'npm';
  id: string;
  name: string;
  package: string;      // npm package name
  args?: string[];
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;   // True if installed locally
  installedVersion?: string;
}

interface PythonMcpServerConfig {
  type: 'python';
  id: string;
  name: string;
  module: string;       // Python module name
  args?: string[];
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;
  installedVersion?: string;
}

interface CustomMcpServerConfig {
  type: 'custom';
  id: string;
  name: string;
  sourcePath?: string;  // Original path for import
  command: string;      // Installed or original path
  args?: string[];
  env?: Record<string, string>;
  enabled: boolean;
  installed: boolean;   // True if copied to RuntimeStore
}

// Type 4: Remote server (SSE transport)
interface SseMcpServerConfig {
  type: 'sse';
  id: string;
  name: string;
  url: string;          // Remote server URL
  headers?: Record<string, string>;  // Auth headers
  enabled: boolean;
}

type McpServerConfig = NpmMcpServerConfig | PythonMcpServerConfig | CustomMcpServerConfig | SseMcpServerConfig;

interface McpConnectionStatus {
  serverId: string;
  status: 'disconnected' | 'connecting' | 'connected' | 'error';
  error?: string;
  tools?: { name: string; description: string }[];
  connectedAt?: string;
}
```

### 4.2 React Hook

Location: `src/renderer/hooks/useMcpSettings.ts`

```typescript
export function useMcpSettings() {
  const [servers, setServers] = useState<McpServerConfig[]>([]);
  const [status, setStatus] = useState<Record<string, McpConnectionStatus>>({});
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    // Initial load
    Promise.all([
      window.api.mcp.getServers(),
      window.api.mcp.getStatus()
    ]).then(([servers, status]) => {
      setServers(servers);
      setStatus(status);
      setLoading(false);
    });
    
    // Subscribe to status changes
    window.api.mcp.onStatusChange(setStatus);
  }, []);
  
  const saveServer = async (config: McpServerConfig) => {
    await window.api.mcp.saveServer(config);
    setServers(await window.api.mcp.getServers());
  };
  
  const deleteServer = async (id: string) => {
    await window.api.mcp.deleteServer(id);
    setServers(await window.api.mcp.getServers());
  };
  
  const testConnection = async (config: Partial<McpServerConfig>) => {
    return window.api.mcp.testConnection(config);
  };
  
  return { servers, status, loading, saveServer, deleteServer, testConnection };
}
```

## 5. Error Handling

### 5.1 User-Facing Error Messages

| Error Scenario | Message |
|:---------------|:--------|
| Invalid package name | "Could not find package '{name}'. Please check the package name." |
| Network error | "Network error. Please check your internet connection." |
| Server crash | "MCP server crashed unexpectedly. Check server logs for details." |
| Missing env var | "Server requires environment variable '{key}'." |
| Timeout | "Connection timed out. The server may be slow or unresponsive." |

## 6. File Changes Summary

| File | Change |
|:-----|:-------|
| `src/renderer/pages/SettingsPage.tsx` | **MOD**: Add MCP section |
| `src/renderer/components/mcp/McpServerList.tsx` | **NEW** |
| `src/renderer/components/mcp/McpServerCard.tsx` | **NEW** |
| `src/renderer/components/mcp/McpServerForm.tsx` | **NEW** |
| `src/renderer/hooks/useMcpSettings.ts` | **NEW** |
| `src/main/ipc/handlers/mcpHandlers.ts` | **NEW** |
| `src/preload/index.ts` | **MOD**: Add mcp API |
| `src/shared/types/runtimeStore.ts` | **MOD**: Add MCP types |

## 7. Testing

### Unit Tests
- Form validation (required fields, package name format)
- Status indicator rendering for each state
- Environment variable editor CRUD operations

### Integration Tests
- Save/load server configuration via IPC
- Test connection flow (success and failure paths)
- Status updates via IPC events

### E2E Tests
- Add server → Test → Save → Verify in list
- Edit server → Reconnect → Verify new config
- Delete server → Verify removed
- Global toggle → Verify MCP tools availability
