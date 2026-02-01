# Story 9.3: Directory Structure Refresh (Fast/Watcher)

Status: done

<!-- Note: Code review complete. See code-review-9-3-directory-structure-refresh.md -->

## Overview

Implement automatic directory structure refreshing in the Runtime file explorer to reflect external changes (e.g., files created by the OS or other tools) immediately.

---

## User Story

As a **Consumer**,
I want the file explorer to refresh directory structures quickly and accurately,
So that I always see the current state of my project files.

---

## Acceptance Criteria

### AC-1: Auto-Refresh (Watcher)
**Given** the Runtime is open
**When** a file/folder is created, deleted, or renamed externally in the project root
**Then** the File Explorer UI updates automatically to reflect the change
**And** the update latency is low (<2s)

### AC-3: Performance
**Given** a project with many files (e.g., node_modules)
**When** the watcher runs
**Then** it should not cause high CPU usage (ignore typical ignored directories like `.git`, `node_modules`)

---

## Validation Report

**Validated**: 2026-01-27
**Verdict**: ✅ **Specs & Design Approved**

### 1. Integrity Check
- **Capabilities**: Watcher covers needs.
- **Tech Stack**: `chokidar` is the standard choice.

### 2. Implementation Risks
- **Resource Leak**: Must close watcher on project switch/quit.
- **Event Flooding**: Debounce is critical.

---

## Design Specification

### 1. Design Principles
- **Main Process Watcher**: Offload file watching to Main process (chokidar) to keep Renderer light.
- **Push Updates**: Main process pushes `files:changed` event; Renderer reacts.
- **Stability**: Robust error handling and resource cleanup.

### 2. Change Scope
| Component | Change | Description |
|-----------|--------|-------------|
| `package.json` | MODIFY | Add `chokidar` (to `devDependencies` or main `dependencies`? Runtime needs it in prod, so `dependencies`). |
| `FileWatcherService` | NEW | Singleton service in Main process. |
| `main.ts` | MODIFY | Initialize service, bind IPC. |
| `preload.ts` | MODIFY | Expose watcher control IPC (`startFileWatcher` / `stopFileWatcher`). |
| `appStore` | MODIFY | Start/stop watcher on project open/close. |
| `useFileExplorer` | MODIFY | Listen to `files:changed` and reload file tree. |

### 3. Data Structures
```typescript
class FileWatcherService {
    private watcher: chokidar.FSWatcher | null = null;
    start(projectRoot: string): void
    stop(): void
}
```

### 4. Data Flow
1. **Start Watching**: Renderer on project open -> `window.ipcRenderer.startFileWatcher(projectRoot)` -> Main `files:watch-start` -> `FileWatcherService.start(projectRoot)`.
2. **Detection**: `chokidar` detects `add/change/unlink/...` and aggregates changes.
3. **Optimizing**: Debounce events (100-300ms) and emit batched alias paths.
4. **Notify**: Main broadcasts `files:changed` to all windows with `{ projectRoot, paths }` (paths are `@project/...` alias paths).
5. **Update**: Renderer `useFileExplorer` listens to `files:changed`, reloads tree (`loadProjectTree()`), and refreshes selected file content when needed.

---

## Implementation Steps

1. **Install**: `npm install chokidar`.
2. **Service**: Implement `FileWatcherService.ts` in `electron/services/`.
3. **IPC**: Define `files:changed` channel.
4. **Integration**:
   - Start watcher when project opens.
   - Stop watcher when project closes/switches.
   - Renderer listens to IPC and triggers re-fetch (or precise update).

---

## Impact Analysis

| Component | Change | Risk |
|:----------|:-------|:-----|
| `Main Process` | Background Service | Low (if managed well) |
| `Performance` | CPU Usage | Medium (Ignored paths essential) |

---

## Tasks / Subtasks

- [x] Install `chokidar` <!-- id: 0 -->
- [x] Implement `FileWatcherService` (Main) <!-- id: 1 -->
- [x] Add `files:changed` IPC event <!-- id: 2 -->
- [x] Implement Renderer listener & Auto-refresh logic <!-- id: 3 -->

### Review Follow-ups (AI)

- [x] [AI-Review][MEDIUM] Restrict `files:watch-start` to the active project root (validate path exists + prevent renderer from watching arbitrary directories). [crewagent-runtime/electron/main.ts]
- [x] [AI-Review][MEDIUM] Stop file watcher when all windows are closed on macOS (avoid background watcher when app remains active without windows). [crewagent-runtime/electron/main.ts]
- [x] [AI-Review][MEDIUM] Reconcile Story 9-3 File List with actual git changes (or split unrelated changes into separate story/commit) so review evidence is auditable. [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]
- [x] [AI-Review][LOW] Update Design Spec to reflect actual `files:changed` payload shape (`{ projectRoot, paths }`) and where the renderer listener lives (`useFileExplorer`). [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]
- [x] [AI-Review][LOW] Remove remaining “Manual refresh” wording from Validation Report to avoid spec drift. [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]

---

## File List

- `crewagent-runtime/package.json`
- `crewagent-runtime/package-lock.json`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/FileWatcherService.ts`
- `crewagent-runtime/electron/services/FileWatcherService.test.ts`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/hooks/useFileExplorer.ts`
- `crewagent-runtime/src/components/Spreadsheet/SpreadsheetEditor.css`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.css`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`

## Dev Agent Record

### Agent Model Used

GPT-5（Codex CLI）

### Debug Log References

- `crewagent-runtime`: `npm test`
- `crewagent-runtime`: `npm run lint`（当前仓库存在既有 `any`/`no-extra-semi` 等问题，lint 仍失败；本 Story 相关新增文件已避免引入新的显式 any）

### Completion Notes List

- 新增 Main 进程 `FileWatcherService`（`chokidar`）监听 ProjectRoot 变更，并对事件做 150ms debounce 聚合后广播 `files:changed`（payload: `{ projectRoot, paths }`），与现有 Renderer 的 `useFileExplorer` 监听逻辑复用。
- Watcher 默认忽略 `.git`、`node_modules` 路径，避免大项目高 CPU；关闭项目/退出应用/窗口关闭会 stop watcher，避免资源泄漏。
- `files:watch-start` 在 Main 侧做 active project + 路径存在性校验，避免被滥用 watch 任意目录。
- Renderer：项目打开时启动 watcher，关闭项目时停止；UI 不再提供手动 `Refresh`（按需求移除）。
- 新增 `FileWatcherService` 单测覆盖 debounce、忽略目录、stop 行为与重复 start。

## Change Log

- 2026-01-28: 实现 Runtime 文件树自动刷新（Watcher + Debounce）；新增单测；移除手动 Refresh；状态更新为 review。
- 2026-01-28: Senior Developer Review（AI）完成：记录 3 MEDIUM / 2 LOW 问题并添加 Review Follow-ups；状态回退为 in-progress。
- 2026-01-28: 完成 Review Follow-ups（watch-start 校验、macOS window-all-closed stop、文档/清单修正）；复跑 `npm -C crewagent-runtime test`/`npm -C crewagent-runtime run build:ci` 通过；状态更新为 done。

## Senior Developer Review (AI)

**Date:** 2026-01-28  
**Outcome:** Approved  

### Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 0 |
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `npm -C crewagent-runtime test` passed（Post-review） |
| Lint | ❌ `npm -C crewagent-runtime run lint` fails（仓库既有 `no-explicit-any` 等错误，非本 Story 引入） |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed（Post-review） |

### Action Items

- [x] [AI-Review][MEDIUM] Restrict `files:watch-start` to the active project root (validate path exists + prevent renderer from watching arbitrary directories). [crewagent-runtime/electron/main.ts]
- [x] [AI-Review][MEDIUM] Stop file watcher when all windows are closed on macOS (avoid background watcher when app remains active without windows). [crewagent-runtime/electron/main.ts]
- [x] [AI-Review][MEDIUM] Reconcile Story 9-3 File List with actual git changes (or split unrelated changes into separate story/commit) so review evidence is auditable. [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]
- [x] [AI-Review][LOW] Update Design Spec to reflect actual `files:changed` payload shape (`{ projectRoot, paths }`) and where the renderer listener lives (`useFileExplorer`). [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]
- [x] [AI-Review][LOW] Remove remaining “Manual refresh” wording from Validation Report to avoid spec drift. [_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md]
