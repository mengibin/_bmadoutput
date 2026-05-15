# Tech-Spec: Story 14.3 SDK Wiki Pack Management UI and Remove

**Created:** 2026-05-14
**Status:** Ready for Development
**Source Story:** `_bmad-output/implementation-artifacts/14-3-sdk-wiki-pack-management-ui-and-remove.md`

## Overview

### Problem Statement

Story 14.1/14.2 已经完成 SDK Wiki Pack 的导入、注册、查询和只读工具集成，但用户目前只能通过开发者 IPC 或手工路径操作导入，Runtime Settings 里没有可见入口，也没有删除已安装 `sdkId@version` 的能力。后续 14.4/14.5 会依赖稳定的已安装 SDK Wiki 集合，因此 14.3 必须补上可见管理入口和安全 remove 生命周期。

### Solution

在 Settings 新增 `SDK Wiki Packs` 管理区块，复用现有 list/import directory/import archive IPC，并新增 `sdk-wiki:remove` IPC。主进程新增 `SdkWikiService.removeInstalled({ sdkId, version })`，按 registry 中的 `sdkId@version` 删除对应安装目录，并通过 move-to-temp + atomic registry write + rollback 保证 registry 和已安装文件不出现不一致状态。

Renderer 侧在 `appStore` 增加 SDK Wiki 管理状态与 actions，Settings 只消费 store 状态并展示 compact metadata、导入结果、校验失败摘要、删除确认和错误状态。Renderer 不接收、不展示 RuntimeStore 绝对路径。

### Scope

In scope:

- Settings `SDK Wiki Packs` section；
- list/import directory/import archive 的 renderer action 和 UI；
- import cancel、success、structured validation failure 展示；
- `SdkWikiService.removeInstalled`；
- RuntimeStore remove workspace/root helper if needed；
- `sdk-wiki:remove` main/preload/vite-env typing；
- app store SDK Wiki state/actions；
- service/store/renderer tests；
- no absolute RuntimeStore path leak verification。

Out of scope:

- `sdk_wiki.plan_api_usage`；
- SDK tool risk confirmation/audit gate；
- SDK Wiki index rebuild/generation；
- replace/reinstall flow；
- top-level SDK Wiki page；
- Project KB / Personal KB / skills import path reuse；
- SAM-specific UI or adapter behavior。

## Context for Development

### Current Runtime Surface

- `shared/sdkWikiTypes.ts` already defines `InstalledSdkWiki`, `SdkWikiInstalledSummary`, import result and query result types.
- `SdkWikiService` already provides:
  - `listInstalled()`
  - `listSdks(params?)`
  - `importFromPath({ path, kind })`
  - `searchPages/readPage/resolveSymbol/expandRelations`
  - `formatSdkWikiRegistrySystemMessage()`
- `RuntimeStore` already provides:
  - `ensureSdkWikiStoreInitialized()`
  - `getSdkWikiRootPath()`
  - `getSdkWikiRegistryPath()`
  - `getSdkWikiImportWorkspaceRoot(opId)`
  - `getSdkWikiInstallRoot(sdkId, version)`
  - `getSdkWikiRootAlias(sdkId, version)`
  - `readSdkWikiRegistry()`
  - `writeSdkWikiRegistry()`
  - `listInstalledSdkWikis()`
  - `appendSdkWikiImportLog(entry)`
- `main.ts` already has handlers:
  - `sdk-wiki:list`
  - `sdk-wiki:importPath`
  - `sdk-wiki:importDirectoryDialog`
  - `sdk-wiki:importArchiveDialog`
- `preload.ts` and `src/vite-env.d.ts` already expose list/import wrappers, but no remove wrapper.
- `SettingsPage.tsx` already has package/global skill/MCP patterns, including import buttons, `confirmDialog`, busy state, remove buttons, and compact cards/lists.
- There is no current `SettingsPage.test.tsx`; renderer tests use Vitest and often validate helper behavior or `renderToStaticMarkup`.

### Files to Reference

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

### Technical Decisions

- **TD-01**: The first visible entry is a Settings subsection, inserted near Packages because SDK Wiki Pack lifecycle is package-like runtime state.
- **TD-02**: `sdk-wiki:list` should return compact `SdkWikiInstalledSummary[]` to renderer. If the handler keeps returning `InstalledSdkWiki[]` internally, app store must normalize it before storing UI state.
- **TD-03**: The UI exposes only dialog-based import. `sdkWikiImportPath` can remain available for developer/test use but should not be the primary Settings action.
- **TD-04**: Remove accepts only `sdkId` and `version`, never path, alias, or registry item objects from renderer.
- **TD-05**: Remove is not replace. Duplicate import remains rejected; users remove an installed version first and then import a new pack.
- **TD-06**: The remove algorithm moves the install root to RuntimeStore temp before registry mutation and rolls back if registry write fails.
- **TD-07**: Temp cleanup failure after successful registry update is logged as cleanup warning and should not re-add the registry item; the installed pack is already absent from the install root.
- **TD-08**: UI state lives in `appStore` so Settings remains a view layer and later pages can reuse installed SDK Wiki state.

## Contracts

### Shared Types

Extend shared types with remove-specific result types. Reuse `SdkWikiInstalledSummary` as the renderer-facing installed-pack shape.

```ts
export type SdkWikiRemoveErrorCode =
  | 'SDK_WIKI_INVALID_ARGS'
  | 'SDK_WIKI_NOT_INSTALLED'
  | 'SDK_WIKI_REMOVE_FAILED'

export interface SdkWikiRemovePayload {
  sdkId: string
  version: string
}

export interface SdkWikiRemoveError {
  code: SdkWikiRemoveErrorCode
  message: string
  details?: unknown
}

export interface SdkWikiRemoveResult {
  success: boolean
  removed?: {
    sdkId: string
    version: string
  }
  installed: InstalledSdkWiki[]
  error?: SdkWikiRemoveError
}
```

Implementation can either add `SDK_WIKI_NOT_INSTALLED` and `SDK_WIKI_REMOVE_FAILED` to the existing service error union or keep the remove error union separate. The key rule is that remove failures must not be modeled as import validation reports.

### IPC API

Existing handlers stay available:

- `sdk-wiki:list`
- `sdk-wiki:importPath`
- `sdk-wiki:importDirectoryDialog`
- `sdk-wiki:importArchiveDialog`

New handler:

- `sdk-wiki:remove`

Payload:

```ts
type SdkWikiRemovePayload = {
  sdkId: string
  version: string
}
```

Recommended responses:

```ts
// list
ipcOk({
  sdks: SdkWikiInstalledSummary[]
})

// import dialog cancel
ipcOk({
  canceled: true
})

// import dialog success
ipcOk({
  canceled: false,
  result: SdkWikiImportResult,
  sdks: SdkWikiInstalledSummary[]
})

// import failure
ipcErr(error.code, error.message, {
  report: SdkWikiValidationReport,
  details: sanitizedDetails
})

// remove success
ipcOk({
  removed: { sdkId, version },
  sdks: SdkWikiInstalledSummary[]
})

// remove failure
ipcErr('SDK_WIKI_NOT_INSTALLED' | 'SDK_WIKI_REMOVE_FAILED' | 'SDK_WIKI_INVALID_ARGS', message, {
  details: sanitizedDetails
})
```

No IPC response may include:

- `getSdkWikiRootPath()` absolute path；
- install root absolute path；
- temp workspace path；
- source import path from a user-selected directory/archive。

### Preload / Renderer Typing

Add:

```ts
sdkWikiRemove: (payload: { sdkId: string; version: string }) => Promise<{
  success: boolean
  removed?: { sdkId: string; version: string }
  sdks?: SdkWikiInstalledSummary[]
  error?: { code?: string; message?: string; details?: unknown }
}>
```

Tighten SDK Wiki wrapper typings in `src/vite-env.d.ts` where practical:

- `sdkWikiList`
- `sdkWikiImportPath`
- `sdkWikiImportDirectoryDialog`
- `sdkWikiImportArchiveDialog`
- `sdkWikiRemove`

It is acceptable for older generic IPC APIs to remain `Promise<any>`, but the SDK Wiki management wrappers added or touched by 14.3 should be explicit.

### App Store State

Add state:

```ts
type SdkWikiBusyAction =
  | 'none'
  | 'list'
  | 'import-directory'
  | 'import-archive'
  | 'remove'

type SdkWikiManagementMessage = {
  type: 'success' | 'error'
  text: string
  code?: string
  failedChecks?: Array<{
    name: string
    code?: string
    message?: string
  }>
}

sdkWikis: SdkWikiInstalledSummary[]
sdkWikiBusyAction: SdkWikiBusyAction
sdkWikiPendingKey: string | null
sdkWikiMessage: SdkWikiManagementMessage | null
```

Add actions:

```ts
refreshSdkWikis(): Promise<{ success: boolean; error?: string }>

importSdkWikiDirectory(): Promise<{
  success: boolean
  canceled?: boolean
  imported?: SdkWikiInstalledSummary
  error?: string
  code?: string
  failedChecks?: SdkWikiManagementMessage['failedChecks']
}>

importSdkWikiArchive(): Promise<{
  success: boolean
  canceled?: boolean
  imported?: SdkWikiInstalledSummary
  error?: string
  code?: string
  failedChecks?: SdkWikiManagementMessage['failedChecks']
}>

removeSdkWiki(sdkId: string, version: string): Promise<{
  success: boolean
  error?: string
  code?: string
}>
```

Store behavior:

- `refreshSdkWikis` sets `sdkWikiBusyAction = 'list'` while loading and stores normalized summaries.
- Import cancel returns `{ success: false, canceled: true }` and does not overwrite message or list.
- Import success refreshes `sdkWikis` from response and sets a success message with `sdkId@version` and page count.
- Import failure extracts failed checks from `error.details.report.checks` and does not replace current list unless the response includes a known-good installed list.
- Remove success refreshes `sdkWikis` from response and sets a success message.
- Remove failure leaves `sdkWikis` unchanged and sets structured error message.
- `sdkWikiPendingKey` is `${sdkId}@${version}` only during remove.

### Settings UI

Add a section after Packages:

```text
Settings
  Packages
  SDK Wiki Packs
  LLM Provider
  ...
```

Recommended section controls:

- icon: `BookOpen` or `Library`;
- header actions:
  - `Import Directory`
  - `Import Archive`
  - refresh icon button with `aria-label="Refresh SDK Wiki packs"`；
- summary chips:
  - installed count；
  - ready count；
  - total pages；
- empty state when `sdkWikis.length === 0`；
- list rows/cards for each pack；
- remove button per pack, disabled while unrelated SDK Wiki action is busy；
- `confirmDialog` before remove。

Displayed metadata per pack:

- name or `sdkId`；
- `sdkId@version`；
- language；
- status；
- page count；
- page types；
- root alias；
- content/index hash short forms when present；
- source documents when present。

Do not render:

- install root absolute path；
- registry path；
- temp workspace path；
- original source import path。

### Remove Confirmation Copy

Use explicit destructive copy:

```ts
{
  title: 'Remove SDK Wiki Pack',
  message: `Remove ${displayName} (${sdkId}@${version})?`,
  detail: 'This removes the installed SDK Wiki Pack from Runtime private storage. It does not delete your original source documents.',
  okLabel: 'Remove',
  cancelLabel: 'Cancel',
}
```

### Validation Failure Message

Render a compact message:

```text
Import failed: SDK_WIKI_INDEX_INVALID
index-files: index manifest is missing required file: index/pages.json
```

Display at most 5 failed checks and add `+N more` if needed.

## Service Remove Design

### RuntimeStore Helpers

Add:

```ts
getSdkWikiRemoveWorkspaceRoot(opId: string): string
```

Suggested path:

```text
<runtime-store>/tmp/sdk-wiki-remove-<opId>
```

Keep it under the same RuntimeStore root so `fs.renameSync` usually stays on the same volume.

Optional helper:

```ts
assertSdkWikiInstallRootInsideStore(sdkId: string, version: string): string
```

If implemented, it should resolve `getSdkWikiInstallRoot()` and verify it stays under `getSdkWikiRootPath()` using `path.relative`.

### `SdkWikiService.removeInstalled`

Signature:

```ts
removeInstalled(params: { sdkId: string; version: string }): SdkWikiRemoveResult
```

Algorithm:

1. Create `opId` and `startedAt`.
2. Validate `sdkId` and `version` with the same safe segment rules used during import.
3. Read registry.
4. Find exact registry item by `sdkId` + `version`.
5. If not found, return `SDK_WIKI_NOT_INSTALLED`; do not delete any directory.
6. Resolve install root from RuntimeStore only.
7. Verify install root exists and is a directory. If registry points at a missing directory, return `SDK_WIKI_REMOVE_FAILED` and leave registry unchanged.
8. Create clean remove workspace.
9. Move install root to `<removeWorkspace>/pack`.
10. Write registry without the item using `RuntimeStore.writeSdkWikiRegistry`.
11. If registry write fails, move `<removeWorkspace>/pack` back to the original install root before returning `SDK_WIKI_REMOVE_FAILED`.
12. Delete remove workspace best-effort after registry success.
13. Append success/failure log entry through `appendSdkWikiImportLog` or a renamed generic `appendSdkWikiLog` wrapper.
14. Return `{ success: true, removed, installed: listInstalled() }`.

### Consistency Rules

- Remove must never call `fs.rmSync` on a path derived from renderer payload.
- The only delete target after success is the RuntimeStore remove workspace.
- If validation fails, registry and install files are untouched.
- If move to temp fails, registry is untouched.
- If registry write fails, service must attempt rollback before returning.
- If rollback fails, service returns `SDK_WIKI_REMOVE_FAILED` with sanitized details and logs the failure.
- If temp cleanup fails after registry success, service can still return success because the installed root and registry agree that the pack is removed; log cleanup failure internally.

### Log Events

Append NDJSON entries:

```ts
{
  event: 'sdk_wiki.removed',
  opId,
  sdkId,
  version,
  status: 'success',
  startedAt,
  finishedAt
}
```

```ts
{
  event: 'sdk_wiki.remove_failed',
  opId,
  sdkId,
  version,
  status: 'failed',
  errorCode,
  errorMessage,
  startedAt,
  finishedAt
}
```

Renderer-visible errors should not include absolute paths from these log entries or caught filesystem errors.

## Implementation Plan

### Tasks

- [ ] Extend SDK Wiki shared types with remove payload/result/error and renderer-facing summary aliases.
- [ ] Add RuntimeStore remove workspace helper and tests.
- [ ] Implement `SdkWikiService.removeInstalled`.
- [ ] Update or add service tests for remove success, invalid args, not installed, missing install root, registry rollback, and no unrelated deletion.
- [ ] Add `sdk-wiki:remove` main IPC handler.
- [ ] Add preload wrapper and `src/vite-env.d.ts` typing.
- [ ] Add app store SDK Wiki state/actions and normalization helpers.
- [ ] Add Settings `SDK Wiki Packs` section.
- [ ] Add CSS classes for compact SDK Wiki list/message states following existing Settings style.
- [ ] Add renderer tests for store behavior and Settings rendering/helpers.

### Acceptance Criteria Mapping

| AC | Design Coverage |
|:---|:---|
| AC-1 | Settings section, header import/refresh actions, empty state |
| AC-2 | `SdkWikiInstalledSummary` UI state, no absolute path rule |
| AC-3 | dialog import actions, cancel no-op, success refresh/message |
| AC-4 | import failure extraction from validation report |
| AC-5 | remove IPC/service/UI confirmation |
| AC-6 | move-to-temp + atomic registry + rollback consistency rules |
| AC-7 | explicit preload/vite-env/app store types |
| AC-8 | service/store/renderer test plan |

## Testing Strategy

### Service Tests

Add to `electron/services/sdkWikiService.test.ts`:

- removes an installed SDK Wiki and updates registry；
- remove returns `SDK_WIKI_NOT_INSTALLED` for missing `sdkId@version` and leaves files untouched；
- invalid sdk id/version returns `SDK_WIKI_INVALID_ARGS`；
- registry write failure after move rolls install directory back；
- missing install root with registry entry returns `SDK_WIKI_REMOVE_FAILED` and leaves registry unchanged；
- removing `sam@0.1.0` does not delete `sam@0.2.0` or another SDK；
- remove success/failure appends log event。

### RuntimeStore Tests

Add to `electron/stores/runtimeStore.test.ts`:

- `getSdkWikiRemoveWorkspaceRoot('abc')` resolves under `<runtime-store>/tmp/sdk-wiki-remove-abc`；
- SDK Wiki install root helper stays under `sdk-wikis/<sdkId>/<version>` for safe segments。

### App Store Tests

Add to `src/stores/appStore.test.ts`:

- `refreshSdkWikis` normalizes list response and stores summaries；
- directory import cancel does not clear list or set error；
- directory/archive import success updates list and message；
- import failure extracts `failedChecks` from validation report；
- remove success updates list and clears pending key；
- remove failure preserves list and stores code/message。

### Settings UI Tests

If a full Settings test is too expensive, extract small pure helpers for display text/message normalization and test those. Preferred coverage:

- empty state markup contains `SDK Wiki Packs` and import buttons；
- installed pack markup contains `sdkId@version`, page count, root alias, and no absolute path；
- validation failure renders failed check summary；
- remove button calls the confirm/remove handler path in a helper-level test。

### Validation Commands

```bash
cd crewagent-runtime && npm test -- electron/services/sdkWikiService.test.ts
cd crewagent-runtime && npm test -- electron/stores/runtimeStore.test.ts
cd crewagent-runtime && npm test -- src/stores/appStore.test.ts
cd crewagent-runtime && npm test -- src/pages/SettingsPage/SettingsPage.test.tsx
cd crewagent-runtime && npm run build:ci
```

If `SettingsPage.test.tsx` is not introduced, replace that command with the nearest renderer test that covers the extracted SDK Wiki Settings helpers.

## Risk Notes

- `SettingsPage.tsx` is already large. Keep 14.3 changes scoped; extract tiny helper functions only if they make testing feasible.
- Current `vite-env.d.ts` has broad `any` IPC typing. 14.3 should tighten SDK Wiki wrappers without attempting a broad IPC typing refactor.
- Import failure details can carry filesystem messages. Before putting details in UI, render only code/message/check summaries and avoid raw paths.
- Registry rollback can still fail on filesystem permission errors. The service must return structured failure and log enough internal detail for troubleshooting.

## Traceability

- Story: `_bmad-output/implementation-artifacts/14-3-sdk-wiki-pack-management-ui-and-remove.md`
- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Prior story: `_bmad-output/implementation-artifacts/14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
