# Tech Spec: Story 5-15 Python Script Execution

## 1. Overview

This spec describes how to integrate Python script execution capability into the CrewAgent Runtime. The key design decision is to **bundle a portable Python distribution** with the Electron app to ensure consistent behavior regardless of the user's host environment.

## 2. Architecture

### 2.1 Bundled Python Strategy

| Platform | Approach |
|----------|----------|
| **macOS** | Download `python-3.11.x-macos11.pkg` standalone build from [python.org/ftp](https://www.python.org/ftp/python/) or use a relocatable build. Extract to `resources/python/`. |
| **Windows** | Use `python-embed` zip from python.org. Extract to `resources/python/`. |
| **Linux** | Use AppImage-bundled Python or extract from official tarball. |

The bundled Python will be placed in `resources/python/` and accessed via Electron's `process.resourcesPath` API.

### 2.2 Tool Schema

```typescript
// In getVisibleTools()
{
  type: 'function',
  function: {
    name: 'python.run',
    description: 'Execute a Python script using the bundled Python environment. Returns stdout, stderr, and exit code.',
    parameters: {
      type: 'object',
      properties: {
        code: { 
          type: 'string', 
          description: 'Python code to execute. If provided, this takes precedence over file.' 
        },
        file: { 
          type: 'string', 
          description: 'Path to a .py file (alias path like @project/scripts/foo.py). Used if code is not provided.' 
        },
        args: {
          type: 'array',
          items: { type: 'string' },
          description: 'Optional command-line arguments to pass to the script.'
        }
      },
      additionalProperties: false,
    },
  },
}
```

### 2.3 Result Type

```typescript
type PythonRunResult =
  | { ok: true; stdout: string; stderr: string; exitCode: number; durationMs: number }
  | { ok: false; error: ToolError }
```

## 2.4 Feasibility Analysis

### Technical Feasibility: ✅ Highly Feasible

Based on analysis of the existing Runtime codebase:

| Component | Current State | Integration Effort |
|:----------|:-------------|:-------------------|
| `ToolHost` interface | Extensible via `getVisibleTools()` + `executeToolCall()` | Low |
| `FileSystemToolHost` | Handles `fs.*` tools; can add `python.run` in same pattern | Low |
| `ExecutionEngine` | Already routes all tools via `ToolHost`; no changes needed | None |
| `RuntimeSettings` | Already has `engine.maxTurns`; can add `pythonTimeout` | Low |
| `SettingsPage` | Already renders engine settings; add one input field | Low |

### Integration Points (Runtime Only)

| File | Change |
|:-----|:-------|
| `electron/services/fileSystemToolHost.ts` | Add `python.run` to `getVisibleTools()`. Add `pythonRun()` handler in `dispatch()`. |
| `electron/services/pythonService.ts` | **NEW**: Utility to resolve bundled Python path. |
| `electron/stores/runtimeStore.ts` | Add `pythonTimeout` to `RuntimeSettings`. |
| `src/pages/SettingsPage/SettingsPage.tsx` | Add "Python Timeout" input field. |
| `electron/services/prompt-templates/system-base-rules.md` | Add Python usage instructions. |
| `electron-builder.json5` | Configure `extraResources` to bundle Python. |

**No changes needed in:**
- `executionEngine.ts` (already routes tools via `ToolHost`)
- Builder frontend/backend (out of scope)
- `bmad-package-spec` (not strictly necessary for v1)

### Risk Analysis

| Risk | Level | Mitigation |
|:-----|:------|:-----------|
| Large bundle size (~50MB Python) | Medium | Accept for v1; consider optional download later |
| Missing Python on host | None | Bundled Python eliminates this risk |
| Missing packages (pandas, numpy) | Medium | LLM sees `ModuleNotFoundError` and can inform user |
| Infinite loop in script | Low | Timeout watchdog + SIGTERM/SIGKILL (POSIX kills process group) |
| Security (arbitrary code execution) | High | Same risk as `fs.write`; Accept for v1 (auto-execution) |
| State persistence (REPL mode) | Low | **Not supported in v1** (Script Mode only) |

### Design Decisions

1. **Bundled Python**: Use portable Python 3.11+ in `resources/python/` to ensure consistent behavior regardless of host environment.
2. **Script Mode**: Each `python.run` call is a fresh process. Variables do not persist between calls. (REPL mode deferred to future.)
3. **Auto-Execution**: No user confirmation required (consistent with `fs.write`).
4. **Timeout**: Configurable in Settings, default 60 seconds.
5. **Python Binary Resolution**: Auto-detect platform and resolve correct binary path (`python3` on macOS/Linux, `python.exe` on Windows).

## 3. Implementation

### 3.1 Bundled Python Path Resolution

```typescript
// electron/services/pythonService.ts (new file)
import path from 'node:path'
import { app } from 'electron'
import fs from 'node:fs'

export function getBundledPythonPath(): string | null {
  const resourcesPath = process.resourcesPath ?? app.getAppPath()
  const candidates = [
    path.join(resourcesPath, 'python', 'bin', 'python3'),    // macOS
    path.join(resourcesPath, 'python', 'python.exe'),        // Windows
    path.join(resourcesPath, 'python', 'bin', 'python'),     // Linux
  ]
  for (const candidate of candidates) {
    if (fs.existsSync(candidate)) return candidate
  }
  return null
}
```

### 3.2 pythonRun() Implementation

```typescript
// In fileSystemToolHost.ts
import { spawn } from 'node:child_process'
import os from 'node:os'
import { getBundledPythonPath } from './pythonService'

private async pythonRun(
  args: Record<string, unknown>,
  context: ExecuteContext,
  timeoutSeconds: number
): Promise<ToolResult> {
  const pythonPath = getBundledPythonPath()
  if (!pythonPath) {
    return this.err('PYTHON_NOT_FOUND', 'Bundled Python not found in resources')
  }

  const code = typeof args.code === 'string' ? args.code : undefined
  const filePath = typeof args.file === 'string' ? args.file : undefined
  const scriptArgs = Array.isArray(args.args) ? args.args.filter(a => typeof a === 'string') : []

  if (!code && !filePath) {
    return this.argErr('python.run requires either code or file argument')
  }

  let scriptPath: string
  const tempDir = os.tmpdir()

  if (code) {
    // Write code to temp file
    const tempFile = path.join(tempDir, `crewagent-py-${Date.now()}.py`)
    fs.writeFileSync(tempFile, code, 'utf8')
    scriptPath = tempFile
  } else {
    // Resolve file path
    const resolved = this.resolveExistingPath(filePath!, context)
    if (!resolved.ok) return resolved
    scriptPath = resolved.absPath
  }

  return new Promise((resolve) => {
    const startedAt = Date.now()
    const child = spawn(pythonPath, ['-E', '-s', scriptPath, ...scriptArgs], {
      cwd: context.projectRoot,
      detached: process.platform !== 'win32',
      windowsHide: true,
    })

    let stdout = ''
    let stderr = ''
    let timedOut = false

    // Timeout handled via watchdog (SIGTERM → SIGKILL). POSIX implementations should prefer killing the process group.
    const timeout = setTimeout(() => {
      timedOut = true
      try { child.kill('SIGTERM') } catch {}
      setTimeout(() => { try { child.kill('SIGKILL') } catch {} }, 2000)
    }, timeoutSeconds * 1000)

    child.stdout.on('data', (data) => { stdout += data.toString() })
    child.stderr.on('data', (data) => { stderr += data.toString() })

    child.on('close', (exitCode) => {
      clearTimeout(timeout)
      const durationMs = Date.now() - startedAt
      if (code) fs.unlinkSync(scriptPath) // Clean up temp file
      if (timedOut) {
        return resolve(this.err('PYTHON_TIMEOUT', `Script exceeded timeout (${timeoutSeconds}s)`))
      }
      resolve({
        ok: true,
        stdout: stdout.slice(0, 50000),  // Limit output size
        stderr: stderr.slice(0, 10000),
        exitCode: exitCode ?? 1,
        durationMs,
        stdoutTruncated: stdout.length > 50000,
        stderrTruncated: stderr.length > 10000,
      })
    })

    child.on('error', (error) => {
      clearTimeout(timeout)
      const durationMs = Date.now() - startedAt
      if (code && fs.existsSync(scriptPath)) fs.unlinkSync(scriptPath)
      resolve(this.err('PYTHON_EXEC_FAILED', error.message))
    })
  })
}
```

### 3.3 Settings Schema Update

```typescript
// In runtimeStore.ts - RuntimeSettings type
export interface RuntimeSettings {
  // ... existing fields
  engine?: {
    maxTurns?: number
    pythonTimeout?: number  // NEW: Default 60
  }
}
```

### 3.4 Electron Builder Configuration

```json5
// electron-builder.json5
{
  // ... existing config
  "extraResources": [
    {
      "from": "resources/python",
      "to": "python",
      "filter": ["**/*"]
    }
  ],
  "mac": {
    // ... existing
    "asarUnpack": ["resources/python/**"]
  }
}
```

## 4. System Prompt Update

Add to `system-base-rules.md`:

```markdown
## Python Execution

You have access to a Python execution environment via the `python.run` tool.

**When to use Python:**
- Complex calculations or data transformations
- Operations requiring deterministic logic (vs. LLM inference)
- File parsing (CSV, JSON, XML) when built-in tools are insufficient

**Usage:**
- `python.run(code="print(2+2)")` — Execute inline code
- `python.run(file="@project/scripts/analyze.py")` — Execute a project file
- `python.run(file="@project/scripts/analyze.py", args=["input.csv"])` — With arguments

**Important:**
- The Python environment is isolated; do not assume host packages are available.
- Standard library modules (json, csv, os, math, etc.) are always available.
- If you need external packages, inform the user and ask them to install.
- Prefer reading/writing files via `fs.read`/`fs.write` tools for transparency.
```

## 5. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Large bundle size (~50MB for Python) | Accept for v1; consider optional download later |
| Python version mismatch expectations | Document bundled version (3.11+) in Settings |
| Security (arbitrary code execution) | Same risk as `fs.write`; accept for v1 |
| Missing packages | LLM sees error and can inform user |

## 6. Testing Strategy

### Unit Tests
- Mock `spawn` to verify correct Python path and arguments
- Verify timeout handling
- Verify temp file cleanup

### Integration Tests
- Spawn actual bundled Python on dev machine
- Verify stdout/stderr capture

### Manual Tests
- Test on clean macOS without Python installed
- Test timeout behavior
- Test file execution with arguments
