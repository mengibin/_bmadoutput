# Story 5.15: Python Script Execution Capability

Status: done

<!-- Note: Validation complete. See validation-report-story-5-15.md -->

## Story

As a **Runtime user**,
I want the Agent to execute Python scripts using a bundled Python environment,
so that I can leverage Python for complex calculations, data processing, and deterministic logic without relying on host Python installation.

## Acceptance Criteria

1. **`python.run` Tool Available**
   - **Given** the Agent needs to perform a computation or data operation
   - **When** it calls `python.run` with `code` OR `file` argument
   - **Then** the Runtime executes the script using the **bundled Python** (not host Python)
   - **And** returns `stdout`, `stderr`, and `exitCode` as the tool result

2. **Bundled Python Environment**
   - **Given** the Runtime installation package
   - **Then** it must include a portable Python 3.11+ distribution in `resources/python/`
   - **And** all `python.run` executions must use this bundled Python

3. **Configurable Timeout**
   - **Given** the Settings page
   - **When** I view the Engine section
   - **Then** I can configure "Python Execution Timeout" (default: 60 seconds)
   - **And** scripts exceeding this timeout are killed and return an error

4. **Execution Sources**
   - **Given** the `python.run` tool
   - **Then** it must support:
     - **Ad-hoc code**: `python.run(code="print(2+2)")` — inline Python
     - **File-based**: `python.run(file="@project/scripts/foo.py")` — project file
     - **With arguments**: `python.run(file="...", args=["arg1"])`

5. **Auto-Execution**
   - **Given** the Agent calls `python.run`
   - **Then** the code executes immediately without user confirmation
   - (Consistent with other tools like `fs.write`)

6. **LLM Awareness**
   - **Given** the system prompt (`system-base-rules.md`)
   - **Then** it must document `python.run` availability and usage

## Design

Design complete. Status set to `ready-for-dev`.

### UX / UI

- **Settings → Engine**
  - New input field: "Python Execution Timeout (seconds)"
  - Default: `60`
  - Min: `5`, Max: `600`
  - Label: "Timeout for Python script execution"

### API / Contracts

**Tool Schema (`python.run`)**

```typescript
{
  type: 'function',
  function: {
    name: 'python.run',
    description: 'Execute Python code using the bundled Python. Returns stdout, stderr, exitCode.',
    parameters: {
      type: 'object',
      properties: {
        code: { type: 'string', description: 'Python code to execute' },
        file: { type: 'string', description: 'Path to .py file (e.g., @project/scripts/foo.py)' },
        args: { type: 'array', items: { type: 'string' }, description: 'Arguments to pass to script' },
      },
      additionalProperties: false,
    },
  },
}
```

**Result Type**

```typescript
type PythonRunResult =
  | { ok: true; stdout: string; stderr: string; exitCode: number; durationMs: number }
  | { ok: false; error: ToolError }
```

**Bundled Python Path Resolution**

```typescript
// electron/services/pythonService.ts
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

### Data / Storage

- **EngineSettings** 扩展：`pythonTimeout?: number`（默认 60 秒）
- **Settings 持久化**：`RuntimeStoreRoot/settings.json`（字段：`settings.engine.pythonTimeout`）

### Errors / Edge Cases

- `PYTHON_NOT_FOUND`：Bundled Python 未找到（extraResources 配置错误或包损坏）
- `PYTHON_TIMEOUT`：脚本超时被 kill
- `PYTHON_EXEC_FAILED`：spawn 失败（权限问题等）
- `E_INVALID_ARGS`：未提供 `code` 或 `file` 参数
- `E_FILE_NOT_FOUND`：`file` 参数指定的 `.py` 文件不存在

### Test Plan

- **Unit Tests** (`fileSystemToolHost.test.ts`)
  - `python.run` executes inline code and returns stdout
  - `python.run` executes file from @project path
  - `python.run` respects timeout and returns error
  - `python.run` cleans up temp files after execution
  - `python.run` returns error when Python not found

- **Integration Tests**
  - Spawn actual bundled Python on dev machine
  - Verify stdout/stderr capture

- **Manual Tests**
  - Build packaged app and test on clean macOS
  - Verify Settings timeout field
  - Chat test: "Calculate 2+2 using Python"

## Tasks / Subtasks

- [x] 1) Bundle Python into Runtime package（AC: 2）
  - [x] 1.1 Download portable Python 3.11+ for macOS
  - [x] 1.2 Place in `resources/python/` directory
  - [x] 1.3 Configure `electron-builder.json5` with `extraResources`
  - [x] 1.4 Test packaging and verify Python exists in built app

- [x] 2) Implement `python.run` tool（AC: 1,4,5）
  - [x] 2.1 Create `electron/services/pythonService.ts`（`getBundledPythonPath()`）
  - [x] 2.2 Add `python.run` schema to `FileSystemToolHost.getVisibleTools()`
  - [x] 2.3 Implement `pythonRun()` handler in `FileSystemToolHost`
    - [x] Handle `code` argument（write to temp file）
    - [x] Handle `file` argument（resolve via mount alias）
    - [x] Support `args` array
    - [x] Capture stdout/stderr
    - [x] Clean up temp files
  - [x] 2.4 Implement timeout handling via watchdog + kill (SIGTERM → SIGKILL)

- [x] 3) Add Settings UI（AC: 3）
  - [x] 3.1 Add `pythonTimeout` to `EngineSettings` interface in `runtimeStore.ts`
  - [x] 3.2 Add default value（60 秒）to `defaultSettings.engine`
  - [x] 3.3 Add input field in `SettingsPage.tsx` Engine section

- [x] 4) Update System Prompt（AC: 6）
  - [x] 4.1 Add Python usage instructions to `system-base-rules.md`

- [x] 5) Testing & Verification
  - [x] 5.1 Add unit tests for `pythonRun()` in `fileSystemToolHost.test.ts`
  - [x] 5.2 Run `npm test` and `npm run lint`
  - [x] 5.3 Manual verification on clean macOS

## Dev Notes

### Context / Current Gap

- Runtime 目前只有 `fs.*` 和 `ui.ask_user` 工具，缺少代码执行能力。
- LLM 在数学计算、数据处理等场景下容易产生幻觉，需要 deterministic 的代码执行。
- 用户机器可能没有安装 Python，需要 bundled Python 保证可移植性。

### Feasibility Analysis

| Component | Current State | Integration Effort |
|:----------|:-------------|:-------------------|
| `ToolHost` interface | Extensible via `getVisibleTools()` + `executeToolCall()` | Low |
| `FileSystemToolHost` | Handles `fs.*` tools; can add `python.run` in same pattern | Low |
| `ExecutionEngine` | Already routes all tools via `ToolHost`; no changes needed | None |
| `RuntimeSettings` | Already has `engine.maxTurns`; can add `pythonTimeout` | Low |
| `SettingsPage` | Already renders engine settings; add one input field | Low |

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

1. **Bundled Python**: Use portable Python 3.11+ in `resources/python/` 确保跨平台一致性。
2. **Script Mode**: Each `python.run` call is a fresh process. Variables do not persist between calls. (REPL mode deferred to future.)
3. **Auto-Execution**: No user confirmation required (consistent with `fs.write`).
4. **Timeout**: Configurable in Settings, default 60 seconds.
5. **Python Binary Resolution**: Auto-detect platform and resolve correct binary path.

### Implementation Guardrails

- `pythonRun()` 必须在 `code` 模式下清理临时文件（无论成功或失败）。
- Timeout 必须使用 `spawn` 的 `timeout` 选项，而非自定义 timer。
- 输出大小限制：`stdout` 最大 50KB，`stderr` 最大 10KB（防止 token 爆炸）。
- 路径解析必须复用现有的 `resolveExistingPath()` 方法（确保 sandbox 安全）。

### References

- Spec / docs
  - `crewagent-runtime/docs/runtime-architecture.md`
  - `crewagent-runtime/docs/runtime-spec.md`
  - `_bmad-output/prd.md` (FR-INT-05)
  - `_bmad-output/architecture.md` (Runtime → Python integration)
  - `_bmad-output/implementation-artifacts/tech-spec-5-15-python-script-execution.md`

- Related runtime code
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `crewagent-runtime/electron/services/toolHost.ts`
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
  - `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`
  - `crewagent-runtime/electron-builder.json5`

## File List

- `_bmad-output/implementation-artifacts/5-15-python-script-execution.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/resources/python/` (NEW directory)
- `crewagent-runtime/electron-builder.json5`
- `crewagent-runtime/electron/services/pythonService.ts` (NEW)
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/electron/services/prompt-templates/system-base-rules.md`

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `crewagent-runtime`: `npm test`
- `crewagent-runtime`: `npm run lint`
- `crewagent-runtime`: `npm run build:ci`
- `crewagent-runtime`: `npx electron-builder --dir --config electron-builder.json5`

### Completion Notes List

- 已将 Python 3.11.14（macOS x86_64 便携版）放入 `resources/python`，并通过 `extraResources` 打包；验证构建产物包含 `Resources/python/bin/python3`。
- 新增 `python.run` 工具：支持 `code`/`file`/`args`，使用 bundled Python 运行；采集 stdout/stderr（50KB/10KB byte 限制，超限标记 `stdoutTruncated`/`stderrTruncated`）；执行时使用 `-E -s` 降低宿主环境影响；超时/abort 由 watchdog 触发 SIGTERM→SIGKILL（POSIX 尝试杀进程组）并清理临时文件；同步扩展 ToolResult 与日志脱敏。
- 引入 `engine.pythonTimeout`（5–600 秒，默认 60），并在 Settings/Execution 中提供配置与持久化。
- 修复开发态 Python 路径解析，优先读取 `process.resourcesPath`，并回退到 `APP_ROOT/resources/python` / `app.getAppPath()`。
- 新增单测覆盖 inline/file/timeout/缺失 Python 场景；已执行 `npm test`（修复后再次运行）、`npm run lint`、`npm run build:ci`、`npx electron-builder --dir --config electron-builder.json5`（lint 有 TS 版本提示）。
- 已完成手工验证：设置 5 秒超时，执行 `sleep 10` 脚本触发 `PYTHON_TIMEOUT`，并通过心跳文件确认 kill 行为。

## Change Log

- 2026-01-10: 完成 python.run 工具、打包资源与测试；补充 code-review 改进（更稳的超时 kill、隔离宿主 env、输出截断标记）；状态更新为 done。
