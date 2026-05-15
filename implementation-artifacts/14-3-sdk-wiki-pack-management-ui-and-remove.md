# Story 14.3: SDK Wiki Pack Management UI and Remove

Status: done

<!-- Created after Story 14.2 search/read/symbol/relation implementation and code review were approved; completed with design-story. -->

## Story

作为 **Runtime User**，
我希望在 Runtime 的 Settings 中导入、查看和删除 SDK Wiki Pack，
以便不依赖开发者 IPC 或手工文件操作就能管理 SDK Wiki 知识源。

## Acceptance Criteria

### AC-1: Settings SDK Wiki Packs section

**Given** 用户打开 Settings 页面
**When** Runtime 渲染设置内容
**Then** Settings 必须提供 `SDK Wiki Packs` 管理区块
**And** 该区块提供 `Import Directory`、`Import Archive` 和 refresh/list 能力
**And** 当前没有已安装 SDK Wiki 时显示空状态，而不是错误状态。

### AC-2: Installed SDK Wiki list

**Given** Runtime 已安装一个或多个 SDK Wiki Pack
**When** Settings 加载或用户刷新列表
**Then** UI 必须展示每个 pack 的 compact metadata
**And** 至少包含 `sdkId`、`version`、name、language、status、pageCount、rootAlias、contentHash/indexHash（如果存在）、source documents（如果存在）
**And** 不得展示 RuntimeStore 绝对路径。

### AC-3: Import directory and archive from UI

**Given** 用户在 Settings 中选择导入目录或导入压缩包
**When** 对应 dialog import 成功
**Then** UI 必须刷新 SDK Wiki Pack 列表并显示新安装项
**And** 必须展示导入成功的简要信息，包括 `sdkId@version` 和 page count。

**Given** 用户取消导入 dialog
**When** Runtime 返回 canceled
**Then** UI 不应显示错误，也不应改变当前列表。

### AC-4: Import validation failure display

**Given** 用户导入无效 SDK Wiki Pack
**When** Runtime 返回结构化 import error 和 validation report
**Then** UI 必须展示错误 code/message
**And** 应展示 validation report 中失败 check 的摘要
**And** 已安装列表保持不变。

### AC-5: Remove installed SDK Wiki Pack

**Given** Settings 中显示已安装 SDK Wiki Pack
**When** 用户点击 remove/delete 并确认
**Then** Runtime 必须按 `sdkId` + `version` 删除该 pack
**And** 从 `sdk-wikis/registry.json` 中移除对应条目
**And** 删除 RuntimeStore 下对应已安装 pack 目录
**And** UI 删除后刷新列表。

### AC-6: Remove safety and consistency

**Given** remove 过程中发生校验、registry 写入或文件系统失败
**When** Runtime 返回 remove error
**Then** registry 和已安装文件必须保持一致，不得出现 registry 指向不存在目录的状态
**And** UI 必须展示结构化错误
**And** 不得删除 project files、package files、Project KB、Personal KB、skills 或其他 SDK Wiki 版本。

### AC-7: Renderer state and IPC typing

**Given** Renderer 调用 SDK Wiki list/import/remove 能力
**When** TypeScript 编译
**Then** `window.ipcRenderer`、app store state/actions、Settings UI props 必须有明确类型
**And** 不得使用 `any` 扩散到 Settings SDK Wiki 管理状态之外。

### AC-8: Tests

**Given** 14.3 完成开发
**When** 运行目标测试
**Then** 必须覆盖 list、empty state、directory import、archive import、import failure report、remove confirmation、remove success、remove failure safety、no absolute path leak。

## Design

### Summary

- 14.3 在 Settings 中新增 SDK Wiki Packs 管理区块，作为用户导入/删除 SDK Wiki Pack 的首个可见入口。
- 复用 14.1 已实现的 `SdkWikiService.importFromPath()`、`sdk-wiki:list`、`sdk-wiki:importDirectoryDialog`、`sdk-wiki:importArchiveDialog`。
- 新增 remove service + IPC + preload + renderer store action。
- 删除操作必须以 `sdkId@version` 为唯一目标，不接受任意路径输入。
- UI 不展示 RuntimeStore 绝对路径，只展示 registry compact metadata 和 root alias。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-3-sdk-wiki-pack-management-ui-and-remove.md`

### Design Decisions

- SDK Wiki Packs 管理入口放在 Settings 的 Packages 之后，保持与 Runtime package/skill 管理入口相邻。
- Settings UI 使用 dialog-based import；`sdkWikiImportPath` 保留给开发者/测试路径，不作为主 UI 入口。
- Renderer-facing list 使用 compact `SdkWikiInstalledSummary`，必要时由 app store 从 `InstalledSdkWiki` 归一化。
- Remove IPC 只接受 `{ sdkId, version }`，不接受路径、alias 或完整 registry item。
- Remove 采用 install root move-to-temp、registry atomic write、registry 写失败 rollback 的流程。
- Duplicate import 不在 14.3 变成 replace；用户先 remove 再 import。
- SDK Wiki 管理状态放入 `appStore`，Settings 保持视图和交互编排层。

### UI Placement

First implementation should be a Settings subsection:

```text
Settings
  Packages
  SDK Wiki Packs
  Global Skills
  MCP Servers
  Node & npm
  Personal Knowledge
```

The section should include:

- header: `SDK Wiki Packs`;
- buttons: `Import Directory`, `Import Archive`, optional `Refresh`;
- installed list/table/cards with compact metadata;
- per-pack remove action with confirmation;
- status message area for import/remove success and errors;
- validation failure summary list for import failures.

### Renderer State Contract

Recommended app store additions:

```ts
type SdkWikiPackSummary = {
  sdkId: string
  version: string
  name?: string
  language?: string
  status: 'ready'
  rootAlias: string
  pageTypes: string[]
  sourceDocuments: string[]
  pageCount: number
  contentHash?: string
  indexHash?: string
}

type SdkWikiManagementMessage = {
  type: 'success' | 'error'
  text: string
  code?: string
  failedChecks?: Array<{ name: string; code?: string; message?: string }>
}
```

Store actions:

- `refreshSdkWikis(): Promise<void>`
- `importSdkWikiDirectory(): Promise<void>`
- `importSdkWikiArchive(): Promise<void>`
- `removeSdkWiki(sdkId: string, version: string): Promise<void>`

Renderer state:

- `sdkWikis: SdkWikiPackSummary[]`
- `sdkWikiBusyAction: 'none' | 'list' | 'import-directory' | 'import-archive' | 'remove'`
- `sdkWikiPendingKey?: string`
- `sdkWikiMessage?: SdkWikiManagementMessage`

### Main / Preload Contract

Existing IPC:

- `sdk-wiki:list`
- `sdk-wiki:importPath`
- `sdk-wiki:importDirectoryDialog`
- `sdk-wiki:importArchiveDialog`

New IPC:

- `sdk-wiki:remove`

Payload:

```ts
type SdkWikiRemovePayload = {
  sdkId: string
  version: string
}
```

Response should follow current `ipcOk` / `ipcErr` shape.

On success:

```ts
{
  removed: { sdkId: string; version: string }
  sdks: InstalledSdkWiki[]
}
```

On failure:

```ts
{
  code: 'SDK_WIKI_INVALID_ARGS' | 'SDK_WIKI_NOT_INSTALLED' | 'SDK_WIKI_REMOVE_FAILED'
  message: string
  details?: unknown
}
```

### Service Remove Design

Add `SdkWikiService.removeInstalled(params: { sdkId: string; version: string })`.

Recommended algorithm:

1. Validate `sdkId` and `version` with the same safe segment rules used by import/install paths.
2. Read registry and find exact `sdkId@version`.
3. Resolve install root only from RuntimeStore helper, never from user input.
4. Move install root to a RuntimeStore temp/remove workspace on the same volume.
5. Atomically write registry without the removed item.
6. Delete temp/remove workspace.
7. If registry write fails after move, move the directory back before returning failure.
8. Append remove success/failure events to SDK Wiki log.

This gives the strongest practical consistency available with filesystem operations and atomic registry write.

### Error Codes

Add query/service errors as needed:

- `SDK_WIKI_INVALID_ARGS`
- `SDK_WIKI_NOT_INSTALLED`
- `SDK_WIKI_REMOVE_FAILED`

Do not reuse import-specific codes when remove fails after validation.

### Validation Commands

Expected minimum verification:

```bash
cd crewagent-runtime && npm test -- electron/services/sdkWikiService.test.ts
cd crewagent-runtime && npm test -- electron/stores/runtimeStore.test.ts
cd crewagent-runtime && npm test -- src/stores/appStore.test.ts
cd crewagent-runtime && npm test -- src/pages/SettingsPage/SettingsPage.test.tsx
cd crewagent-runtime && npm exec tsc -- --noEmit
```

If no Settings test file exists yet, 14.3 should create one or add coverage to the nearest existing renderer test harness.

## Developer Context

### Story Boundary

14.3 只实现 SDK Wiki Pack 的可见管理入口和删除生命周期。

In scope:

- Settings SDK Wiki Packs section;
- renderer state/actions for list/import/remove;
- remove IPC/preload/type declarations;
- `SdkWikiService.removeInstalled`;
- RuntimeStore helpers needed for safe remove;
- import failure report display;
- remove confirmation and error display;
- tests for UI/store/service remove behavior.

Out of scope:

- `sdk_wiki.plan_api_usage`，属于 Story 14.4；
- SDK tool risk confirmation gate，属于 Story 14.5；
- SAM golden path / generic adapter contract，属于 Story 14.6；
- SDK Wiki index rebuild/generation、embedding、BM25/SQLite；
- importing SDK Wiki into Project KB、Personal KB or skills；
- separate top-level SDK Wiki page unless design-story explicitly chooses it.

### Runtime Code Intelligence

- 14.1 added `SdkWikiService.importFromPath()` and RuntimeStore SDK Wiki registry/import-log/install-root helpers.
- 14.1/14.2 already expose preload typings for:
  - `sdkWikiList`
  - `sdkWikiImportPath`
  - `sdkWikiImportDirectoryDialog`
  - `sdkWikiImportArchiveDialog`
- `main.ts` has IPC handlers for list/import directory/import archive, but no remove handler yet.
- `RuntimeStore.listInstalledSdkWikis()` returns registry items without an explicit filesystem path.
- `RuntimeStore.getSdkWikiInstallRoot(sdkId, version)` can resolve the private installed pack directory; renderer must not receive that path.
- Existing package removal path (`packages:remove`, `RuntimeStore.removePackage`) is a simpler reference but does not provide the consistency guarantee required here.
- Settings already has a Packages section and patterns for import buttons, warning notices, MCP confirmation dialogs, Global Skills management, and Project/Personal KB busy states.
- `appStore.ts` already centralizes project/package/global skill/project knowledge state and async actions; 14.3 should add SDK Wiki management state there unless design-story extracts a feature store.

### UI/UX Requirements

- Use existing Settings visual language; do not create a marketing/landing page.
- Keep the section operational and dense: concise metadata, obvious import buttons, per-row remove action.
- Show disabled/busy states while import/remove is in progress.
- Remove action must be explicit and confirmed via existing `confirmDialog`.
- Validation report display should be readable but compact: failed check name, code, and message are enough for first release.
- Empty state should explain that no SDK Wiki Packs are installed and offer import actions.

### Security and Safety Requirements

- Remove accepts only `sdkId` and `version`; never accept a path from renderer.
- Validate `sdkId`/`version` with safe segment rules before resolving install root.
- Do not expose absolute RuntimeStore paths in list, error, or UI state.
- Remove must not modify ProjectRoot, packages, KB stores, skills, MCP config, or other SDK Wiki versions.
- Import and remove logs must not include raw private paths in renderer-visible errors.

### File Structure Requirements

Expected implementation files:

- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.test.tsx` or nearest existing Settings renderer test

Avoid modifying:

- `FileSystemToolHost` SDK Wiki query tools unless remove/list changes require shared types cleanup;
- Project KB / Personal KB internals;
- MCP execution/risk policy;
- `sdk_wiki.plan_api_usage`.

### Source Documents and Code Read

- `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
- `_bmad-output/implementation-artifacts/code-review-14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

## Tasks / Subtasks

- [x] Task 1: Remove service and persistence safety (AC: 5,6)
  - [x] 1.1 Add remove result/error types to `shared/sdkWikiTypes.ts`
  - [x] 1.2 Add RuntimeStore helper(s) needed for safe SDK Wiki remove workspace/root resolution
  - [x] 1.3 Implement `SdkWikiService.removeInstalled(sdkId, version)`
  - [x] 1.4 Keep remove operation path-free from renderer input
  - [x] 1.5 Append remove success/failure log events

- [x] Task 2: Main/preload IPC (AC: 5,6,7)
  - [x] 2.1 Add `sdk-wiki:remove` IPC handler
  - [x] 2.2 Add preload wrapper `sdkWikiRemove`
  - [x] 2.3 Add renderer TypeScript declaration
  - [x] 2.4 Ensure IPC errors are structured and path-sanitized

- [x] Task 3: Renderer state and actions (AC: 1-7)
  - [x] 3.1 Add SDK Wiki management state to app store
  - [x] 3.2 Add refresh/list action
  - [x] 3.3 Add import directory action
  - [x] 3.4 Add import archive action
  - [x] 3.5 Add remove action with busy/pending state
  - [x] 3.6 Normalize import validation report into compact UI message state

- [x] Task 4: Settings UI (AC: 1-5)
  - [x] 4.1 Add `SDK Wiki Packs` subsection to Settings
  - [x] 4.2 Render empty state
  - [x] 4.3 Render installed pack metadata without absolute paths
  - [x] 4.4 Add Import Directory / Import Archive buttons
  - [x] 4.5 Add remove action and confirmation dialog
  - [x] 4.6 Render success/error/validation report messages

- [x] Task 5: Tests (AC: 1-8)
  - [x] 5.1 Service tests for remove success and not installed
  - [x] 5.2 Service tests for remove failure consistency/rollback
  - [x] 5.3 IPC/preload typing or main handler coverage where practical
  - [x] 5.4 App store tests for list/import/remove state transitions
  - [x] 5.5 Settings UI tests for empty/list/import failure/remove confirmation
  - [x] 5.6 No absolute private paths in renderer-facing payloads

## Dev Agent Record

### Agent Model Used

Codex GPT-5

### Debug Log References

- Story prepared from Epic 14 plan, updated PRD/architecture docs, and runtime code inspection.
- Design prepared from current Runtime SDK Wiki service/store/IPC/Settings code inspection.
- No runtime implementation code changed during this create-story/design-story step.
- 2026-05-14 - Implemented remove service, Settings SDK Wiki management UI, IPC/preload/app store wiring, and tests.
- 2026-05-14 - Build failure root cause: remove IPC payload cast and stale extracted Settings constants; fixed by explicit payload construction and moving derived values into the pure SDK Wiki section component.
- 2026-05-14 - Code-review follow-up: added remove confirmation interaction coverage and directory import success coverage; 14.3 approved.

### Completion Notes List

- 14.3 story created after 14.2 reached done.
- Story captures the required visible Settings import/list/remove entrypoint that was missing from 14.1/14.2.
- Delete lifecycle is explicitly scoped before 14.4 API planning.
- Tech spec defines remove consistency, IPC, renderer store, Settings UI, and testing contracts.
- Implemented `sdk-wiki:remove`, path-free renderer payloads, Settings import/list/remove controls, validation failure summaries, and rollback-aware service remove behavior.
- Story passed code review and is done.

### File List

- `_bmad-output/implementation-artifacts/14-3-sdk-wiki-pack-management-ui-and-remove.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-3-sdk-wiki-pack-management-ui-and-remove.md`
- `_bmad-output/implementation-artifacts/code-review-14-3-sdk-wiki-pack-management-ui-and-remove.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/src/vite-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/stores/appStore.test.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.test.tsx`
- `crewagent-runtime/src/pages/SettingsPage/sdkWikiSettingsHelpers.ts`

### Change Log

- 2026-05-14 - Created story and marked ready-for-design.
- 2026-05-14 - Completed design-story, added tech spec, and marked ready-for-dev.
- 2026-05-14 - Completed dev-story implementation and marked ready for code review.
- 2026-05-14 - Completed code-review; changes requested for missing remove confirmation and directory import success interaction test coverage.
- 2026-05-14 - Fixed code-review finding M1, reran review, and marked Story 14.3 done.
