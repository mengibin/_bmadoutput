# Code Review: Story 9-3 – Directory Structure Refresh (Fast/Watcher)

**Date:** 2026-01-28  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `9-3-directory-structure-refresh.md`

---

## 范围说明

- 对照 Story 9-3 的 AC 审计 Runtime 文件树 **外部变更自动刷新**（Main watcher + IPC + Renderer 刷新）。  
- 本次工作区存在多处**混入的非本 Story 改动**（例如删除能力/Spreadsheet dark 模式样式），会作为 “Git vs Story Discrepancies” 与审计风险点记录（建议拆分提交或补齐 Story 记录）。

---

## Summary

| Metric | Value |
|--------|-------|
| Git vs Story Discrepancies | 5（见下方问题 M3） |
| HIGH Issues | 0 |
| MEDIUM Issues | 3 |
| LOW Issues | 2 |
| Unit Tests | ✅ `npm -C crewagent-runtime test` passed |
| Lint | ❌ `npm -C crewagent-runtime run lint` fails（仓库既有 `no-explicit-any` 等错误，非本 Story 引入） |
| Build | ✅ `npm -C crewagent-runtime run build:ci` passed |

---

## Acceptance Criteria 验证

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 Auto-Refresh (Watcher) | ✅ | Main watcher：`crewagent-runtime/electron/services/FileWatcherService.ts:28`；IPC 启动：`crewagent-runtime/electron/main.ts:771`、`crewagent-runtime/electron/preload.ts:67`；项目打开后启动 watcher：`crewagent-runtime/src/stores/appStore.ts:427`；Renderer 收到 `files:changed` 触发 `loadProjectTree()`：`crewagent-runtime/src/hooks/useFileExplorer.ts:93` |
| AC-3 Performance | ✅/⚠️ | 已忽略 `.git`/`node_modules`：`crewagent-runtime/electron/services/FileWatcherService.ts:83`；事件 flood 通过 debounce 聚合：`crewagent-runtime/electron/services/FileWatcherService.ts:100`。⚠️ 仍对整个 ProjectRoot 做递归 watch，超大项目除 `.git/node_modules` 外仍可能较重（属于可接受风险，但建议后续可配置 ignored 列表/深度）。 |

---

## Issues & Recommendations

### M1 — `files:watch-start` IPC 缺少路径/权限校验（Renderer 可触发任意目录 watch）

**Problem**  
`files:watch-start` 直接对传入 `projectRoot` 执行 `fileWatcherService.start(...)`，没有校验该路径是否为当前激活项目，或是否允许被 watch。若 Renderer 被注入/误用，会导致隐私与性能风险（watch 任意目录、触发大量事件）。  

**Evidence**  
- `crewagent-runtime/electron/main.ts:771`  
- `crewagent-runtime/electron/preload.ts:67`

**Recommendation**  
- 在 Main 侧校验：`projectRoot` 必须等于当前打开项目（或由 Main 持有 activeProjectRoot），且必须存在；不符合则拒绝。  
- 或者改成 `files:watch-start` 不接受路径参数，仅 “start watching current project”。

---

### M2 — macOS 关闭所有窗口但应用不退出时，watcher 仍可能后台运行

**Problem**  
macOS 下 `window-all-closed` 不会 `app.quit()`；目前只在 `before-quit` 调用 `fileWatcherService.stop()`，意味着用户关闭所有窗口但不退出应用时 watcher 仍可能持续运行。  

**Evidence**  
- `crewagent-runtime/electron/main.ts:613`  
- `crewagent-runtime/electron/main.ts:620`

**Recommendation**  
- 在 `window-all-closed`（或最后一个窗口 close）时也 stop watcher；重新打开项目时再启动即可。  

---

### M3 — Git 变更与 Story File List 不一致（审计不可追溯 + 改动混入）

**Problem**  
`crewagent-runtime` 当前 git 工作区变更包含多个文件未出现在 Story 9-3 的 File List 中（以及存在明显非本 Story 范围的改动），导致代码评审证据不完整、后续回溯困难。  

**Evidence**  
- Story File List：`_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md:112`  
- 实际变更（示例）：`crewagent-runtime/electron/stores/runtimeStore.ts`、`crewagent-runtime/src/hooks/useFileExplorer.ts`、`crewagent-runtime/src/pages/FilesPage/FilesPage.css`、`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`、`crewagent-runtime/src/components/Spreadsheet/SpreadsheetEditor.css`

**Recommendation**  
- 要么补齐 Story File List（并说明为何这些改动属于 9-3），要么把无关改动拆分到对应 Story/单独提交。  

---

### L1 — Story 文档中 `files:changed` payload/数据流描述与实现不一致

**Problem**  
Story 文档仍写 `{ type, path }`，但当前实现复用了既有事件形态 `{ projectRoot, paths }`。文档与实现偏离会误导后续维护。  

**Evidence**  
- Story：`_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md:77`  
- 实现：`crewagent-runtime/src/hooks/useFileExplorer.ts:93`

**Recommendation**  
- 更新 Story 的 Design Specification（并补充：listener 实际位于 `useFileExplorer` hook）。  

---

### L2 — Story 验证说明仍提及 Manual refresh（与现状不一致）

**Problem**  
Story 的 Validation Report 仍写 “Watcher + Manual refresh covers needs”，但 UI 已按需求移除 Refresh。  

**Evidence**  
- `_bmad-output/implementation-artifacts/9-3-directory-structure-refresh.md:40`

**Recommendation**  
- 更新对应说明，避免规格漂移。  

---

## Review Outcome

- Outcome: **Changes Requested**（已在 Story 中添加 `Review Follow-ups (AI)` 并将状态回退为 `in-progress`）  
- Sprint tracking: `_bmad-output/implementation-artifacts/sprint-status.yaml` 当前未包含 `9-3-directory-structure-refresh` 条目，无法同步状态（需后续补齐/或由 SM 更新）。  

---

## Post-Review Fixes (2026-01-28)

- ✅ M1 已修复：Main 侧校验 `files:watch-start` 的 `projectRoot`（存在性/目录校验 + 必须为 active project）。  
- ✅ M2 已修复：`window-all-closed`/`before-quit` 会 stop watcher，避免 macOS 无窗口时后台 watch。  
- ✅ M3 已修复：Story 9-3 File List 已补齐（与 `crewagent-runtime` 实际 diff 对齐）。  
- ✅ L1 已修复：Story Design Spec 更新为实际 `files:changed` payload `{ projectRoot, paths }`，并注明 listener 在 `useFileExplorer`。  
- ✅ L2 已修复：Story Validation Report 去除 “Manual refresh” 描述，与现状一致。  
- ✅ Updated Outcome: **Approved**（Post-review fixes applied; tests/build re-run）。  
