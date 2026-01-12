# Story 5.16: Python Auto-Install Missing Libraries

Status: in-progress

<!-- Note: Depends on Story 5.15 (Python Script Execution). Design complete. -->

## Story

As a **Runtime user**,
I want the system to automatically install missing Python libraries when the Agent's code fails with `ModuleNotFoundError`,
so that I don't need to manually manage package installations.

## Acceptance Criteria

1. **Auto-Detect Missing Module**
   - **Given** the Agent calls `python.run(code="import pandas; ...")`
   - **When** the script fails with `ModuleNotFoundError: No module named 'pandas'`
   - **Then** Runtime detects this error and extracts the missing module name

2. **Auto-Install Flow**
   - **Given** a detected missing module
   - **Then** Runtime automatically runs `pip install <module_name>`
   - **And** retries the original `python.run` if install succeeds
   - **And** returns the retry result to the LLM

3. **Install Progress Feedback**
   - **Given** an auto-install is in progress
   - **Then** Runtime returns a tool result indicating "Installing pandas..."
   - **And** the final result includes install outcome + script result

4. **Configurable Behavior (Settings)**
   - **Given** the Settings page → Python section
   - **Then** I can configure:
     - Auto-install: On/Off (default: On)
     - PyPI mirror URL (for restricted networks)
     - Max auto-install retries per run (default: 3)
     - Offline mode: On/Off

5. **Pre-Bundled Light Libraries**
   - **Given** the Runtime installation
   - **Then** it includes: `requests`, `openpyxl`, `python-docx`, `PyYAML`, `beautifulsoup4`, `lxml` (~15MB)
   - **And** large libraries (numpy, pandas) are installed on-demand

6. **Cross-Conversation Sync**
   - **Given** a library was auto-installed in Conversation A
   - **When** the user starts new Conversation B
   - **Then** Conversation B's tool description includes the newly installed library

## Design

Design complete. Status set to `ready-for-dev`.

### Execution Flow

```
python.run(code="import pandas; df = pd.read_csv('x.csv')")
                              │
                              ▼
                 Execute script with bundled Python
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
         [Success]                   [ModuleNotFoundError]
              │                               │
              ▼                               ▼
      Return stdout/stderr          Extract module: "pandas"
                                              │
                                              ▼
                                    pip install pandas
                                              │
                              ┌───────────────┴───────────────┐
                              ▼                               ▼
                       [Install OK]                    [Install Failed]
                              │                               │
                              ▼                               ▼
                     Retry python.run              Return error with details
                              │
                              ▼
                    Return result + autoInstalled: ["pandas"]
```

### API / Contracts

#### [MODIFY] pythonService.ts

```typescript
// Module → Package mapping
export const MODULE_TO_PACKAGE: Record<string, string> = {
  'cv2': 'opencv-python',
  'PIL': 'Pillow',
  'sklearn': 'scikit-learn',
  'yaml': 'PyYAML',
  'bs4': 'beautifulsoup4',
}

// Parse ModuleNotFoundError from stderr
export function parseModuleNotFoundError(stderr: string): string | null {
  const match = stderr.match(/ModuleNotFoundError: No module named '([^']+)'/)
  return match?.[1] ?? null
}

// pip install
export async function pipInstall(pkg: string, opts?: { mirrorUrl?: string }): Promise<{ ok: boolean; output: string }>

// In-memory cache
let installedCache: Set<string> | null = null
export async function getInstalledPackages(): Promise<string[]>
export function addToInstalledCache(pkg: string): void
```

#### [MODIFY] fileSystemToolHost.ts

**pythonRun() with retry:**

```typescript
private async pythonRun(args: PythonRunArgs, context: ExecuteContext): Promise<ToolResult> {
  const { autoInstall, maxAutoInstallRetries, pypiMirrorUrl, offlineMode } = this.runtimeStore.getSettings().python
  let autoInstalled: string[] = []

  for (let attempt = 0; attempt <= maxAutoInstallRetries; attempt++) {
    const result = await this.executePythonScript(args, context)
    if (result.ok) return { ...result, autoInstalled: autoInstalled.length ? autoInstalled : undefined }

    if (!autoInstall || offlineMode || attempt === maxAutoInstallRetries) return result

    const missingModule = parseModuleNotFoundError(result.error?.details?.stderr ?? '')
    if (!missingModule) return result

    const packageName = MODULE_TO_PACKAGE[missingModule] ?? missingModule
    const installResult = await pipInstall(packageName, { mirrorUrl: pypiMirrorUrl })

    if (!installResult.ok) {
      return this.err('INSTALL_FAILED', `Failed to install ${packageName}`, {
        module: missingModule,
        pipOutput: installResult.output,
        suggestions: ['Check network', 'Configure PyPI mirror', 'Import local .whl']
      })
    }
    autoInstalled.push(packageName)
    addToInstalledCache(packageName)
  }
}
```

**getVisibleTools() with dynamic description:**

```typescript
public getVisibleTools(context: ToolContext): OpenAIFunctionTool[] {
  const pkgs = getInstalledPackages().slice(0, 20).join(', ')
  return [
    {
      type: 'function',
      function: {
        name: 'python.run',
        description: `Execute Python code. Installed: ${pkgs}. Others auto-installed on import.`,
        parameters: { /* ... */ }
      }
    }
  ]
}
```

### Data / Storage

#### [MODIFY] runtimeStore.ts

```typescript
export interface PythonSettings {
  autoInstall: boolean          // Default: true
  pypiMirrorUrl?: string
  maxAutoInstallRetries: number // Default: 3
  offlineMode?: boolean         // Default: false
}

const defaultSettings = {
  python: { autoInstall: true, maxAutoInstallRetries: 3, offlineMode: false }
}
```

#### [NEW] resources/python/bundled-packages.json

```json
{
  "schemaVersion": "1.0",
  "packages": [
    { "name": "requests", "version": "2.31.0" },
    { "name": "openpyxl", "version": "3.1.2" },
    { "name": "python-docx", "version": "1.1.0" },
    { "name": "PyYAML", "version": "6.0.1" },
    { "name": "beautifulsoup4", "version": "4.12.3" },
    { "name": "lxml", "version": "5.1.0" }
  ]
}
```

### UX / UI

**Settings → Python Section:**
- Toggle: "Auto-install missing packages"
- Input: "PyPI Mirror URL"
- Toggle: "Offline Mode"
- Button: "Import Local Package (.whl)"

### Errors / Edge Cases

| Code | Scenario | LLM Response |
|:-----|:---------|:-------------|
| `NETWORK_ERROR` | 无网络或镜像不可达 | 告知用户配置镜像或导入本地包 |
| `INSTALL_CONFLICT` | 依赖冲突 | 显示 pip 输出，建议手动处理 |
| `PACKAGE_NOT_FOUND` | 镜像源无此包 | 建议检查包名或使用其他镜像 |
| `MAX_RETRIES_EXCEEDED` | 单次运行安装过多包 | 停止安装，建议手动处理 |

### Test Plan

**Unit Tests:**
1. `parseModuleNotFoundError()` extracts module name
2. `pipInstall()` spawns pip correctly
3. `pythonRun()` retries after successful install
4. `pythonRun()` returns `INSTALL_FAILED` on pip failure
5. `getInstalledPackages()` returns cached list

**Manual Tests:**
1. Auto-install: `import cowsay` triggers install + success
2. Offline mode: Returns clear error without pip attempt
3. Cross-conversation: Start new conversation, verify package in tool description

## Tasks / Subtasks

- [x] 1) Auto-Install Detection & Retry（AC: 1,2,3）
  - [x] 1.1 Add `parseModuleNotFoundError()` to `pythonService.ts`
  - [x] 1.2 Add `MODULE_TO_PACKAGE` mapping
  - [x] 1.3 Implement `pipInstall()` function
  - [x] 1.4 Update `pythonRun()` with retry loop
  - [x] 1.5 Add `autoInstalled` field to result

- [x] 2) Pre-Bundled Libraries（AC: 5）
  - [x] 2.1 Install requests, openpyxl, python-docx, PyYAML, beautifulsoup4, lxml to bundled Python
  - [x] 2.2 Create `bundled-packages.json`
  - [ ] 2.3 Verify bundle size ~65MB

- [x] 3) Settings UI（AC: 4）
  - [x] 3.1 Add `PythonSettings` to `runtimeStore.ts`
  - [x] 3.2 Add Python section to `SettingsPage.tsx`

- [x] 4) Dynamic Tool Info（AC: 6）
  - [x] 4.1 Implement `getInstalledPackages()` with cache
  - [x] 4.2 Update `getVisibleTools()` to inject package list
  - [x] 4.3 Add install note to `ToolResult`

- [ ] 5) Testing
  - [x] 5.1 Unit tests for auto-install flow
  - [ ] 5.2 Manual verification

## Dev Notes

### Implementation Guardrails

- Auto-install 遵守 Settings 的 `autoInstall` 开关
- 每次 run 最多 `maxAutoInstallRetries` 次安装
- pip install 使用 bundled pip
- stdout/stderr 记录到 execution log

### Bundle Size

| Scenario | Size |
|:---------|:-----|
| Story 5-15 (Python only) | ~50MB |
| + Pre-bundled libs | ~65MB |
| + numpy + pandas (on-demand) | ~150MB |

### References

- Story 5-15: Python Script Execution
- Tech Spec: `tech-spec-5-15-python-script-execution.md`

## File List

- `crewagent-runtime/electron/services/pythonService.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/resources/python/` (add libs)
- `crewagent-runtime/resources/python/bundled-packages.json` (NEW)

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `crewagent-runtime`: `npm test -- fileSystemToolHost.test.ts`
- `crewagent-runtime`: `npx tsx .tmp/manual-auto-install.ts`
- `crewagent-runtime`: `npx tsx .tmp/manual-cross-conversation.ts`

### Completion Notes List

- 完成 python.run 自动安装流程：解析 `ModuleNotFoundError`、pip 安装、重试与安装日志；新增 `installLog` 与更清晰的 tool 描述（受离线/开关影响）。
- `pythonRunCore` 在非 0 退出码时返回错误（包含 stderr），以触发自动安装逻辑。
- 工具描述加入 “Recently installed” 列表，确保新会话可见刚安装的库。
- Settings 增加最大自动安装重试次数与本地 `.whl` 导入入口；新增 IPC 选择/安装 wheel，并补齐 Python 设置合并与校验逻辑。
- 预置 requests/openpyxl/python-docx/PyYAML/beautifulsoup4/lxml 到 bundled Python，并更新 `bundled-packages.json`。
- 新增单测覆盖缺包自动安装与失败分支。
- 手工验证 #1：`python.run` 执行 `import cowsay` 触发自动安装，返回 `autoInstalled`/`installLog`；随后卸载 cowsay。
- 手工验证 #4：安装 cowsay 后新建 host，`python.run` 描述包含 “Recently installed: cowsay”.
- 当前 `resources/python` 体积约 88MB（未达到 Dev Notes 中的 ~65MB 目标）。

## Change Log

- 2026-01-10: 修复 code-review 发现的问题，补全自动安装设置/进度反馈/缓存预热/预置依赖与测试；修复非 0 退出码处理并通过单测；补充跨会话可见的最近安装提示；因 bundle 体积偏大与手工验证未完成，状态调整为 in-progress。
