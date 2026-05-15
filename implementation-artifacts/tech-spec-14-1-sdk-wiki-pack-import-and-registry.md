# Tech-Spec: Story 14.1 SDK Wiki Pack Import and Registry

**Created:** 2026-05-13
**Status:** Ready for Development
**Source Story:** `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`

## Overview

### Problem Statement

Runtime 需要导入外部生成的 SDK Wiki Pack，并在注册前验证其结构、manifest、索引、hash 和页面 frontmatter。现有 Project KB 能处理项目资料，但 SDK Wiki 是 SDK 版本级知识源，不能混入 Project KB、Personal KB 或 agent skill。

### Solution

新增 `SdkWikiService` 和 RuntimeStore SDK Wiki 域。导入时统一把目录或压缩包复制/解压到 RuntimeStore temp workspace，完成校验后再 commit 到 `sdk-wikis/<sdkId>/<version>/`，随后原子更新 `sdk-wikis/registry.json` 并追加 `import-log.ndjson`。对 renderer 暴露最小 IPC：list、importPath、importDirectoryDialog、importArchiveDialog。

### Scope

In scope:

- directory/archive import；
- `sdk-wiki.json`、`index/manifest.json`、required dirs、required index files、builder auxiliary wiki files 校验；
- `schemaVersion`、`sdkId`、`version`、`entry` 校验；
- `contentHash`、`indexHash` 存在时校验；
- builder page discovery 后的 `wiki/` Markdown frontmatter 和 page relation 校验；
- transactional commit、registry list、import log；
- service tests、RuntimeStore tests、IPC smoke tests if IPC is implemented。

Out of scope:

- `sdk_wiki.search_pages/read_page/resolve_symbol/expand_relations/plan_api_usage`；
- SDK execution risk gate；
- SDK Wiki index rebuild/generation；
- 可视化 SDK Wiki 管理页；
- SAM-only hardcoding。

## Context for Development

### Codebase Patterns

- `RuntimeStore` 是私有持久化根，位于 Electron `app.getPath('userData')/runtime-store`。
- `RuntimeStore.importPackage()` 已有 zip wrapper normalization、safe extraction、temp install path、cleanup pattern。
- `RuntimeStore.writeJsonAtomic()` 是 registry、install metadata 的原子写参考。
- `ProjectKbService` 是 service pattern 参考：RuntimeStore dependency、`AdmZip`、workspace staging、archive validation、manifest/log、service tests。
- `main.ts` 使用 `ipcOk` / `ipcErr` 包装 renderer API。14.1 IPC 应保持 thin handler，不承载 import 业务逻辑。
- `preload.ts` 通过 `contextBridge.exposeInMainWorld('ipcRenderer', ...)` 暴露 typed wrappers。

### Files to Reference

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/package.json`
- 外部 Pack 样例：
  - `/Users/mengbin/personal/research/tool_wiki/sam-agent-workspace/sdk-wiki-packs/sam/sdk-wiki.json`
  - `/Users/mengbin/personal/research/tool_wiki/sam-agent-workspace/sdk-wiki-packs/sam/index/manifest.json`
  - `/Users/mengbin/personal/research/tool_wiki/sam-agent-workspace/sdk-wiki-packs/abaqus/sdk-wiki.json`
  - `/Users/mengbin/personal/research/tool_wiki/sam-agent-workspace/sdk-wiki-packs/abaqus/index/manifest.json`

### Technical Decisions

- **TD-01**: 14.1 exposes minimal IPC but no visible management UI.
- **TD-02**: Duplicate `(sdkId, version)` is rejected with `SDK_WIKI_ALREADY_INSTALLED`; replace/reinstall is out of scope.
- **TD-03**: Directory import is copied into temp before validation; Runtime never registers the user-selected directory directly.
- **TD-04**: Archive import allows one top-level wrapper directory and normalizes it to Pack root.
- **TD-05**: Manifest MVP follows observed builder output: `schemaVersion`, `sdkId`, `wikiVersion`, `contentHash`, `indexHash`, `builderVersion`, `files`.
- **TD-06**: Core index files required for future 14.2 are `index/pages.json`, `index/symbols.json`, `index/relations.json`, `index/terms.json`.
- **TD-07**: Hash validation uses SDK Wiki Builder algorithm: sort relative paths, update hash with `relPath`, NUL, file bytes, NUL. `contentHash` covers builder-published content files such as `sdk-wiki.json`, `raw/`, and `wiki/`; `indexHash` is recomputed only from `index/pages.json`, `index/symbols.json`, `index/terms.json`, and `index/relations.json`. Runtime must not include `index/manifest.json`, `index/README.md`, or `reports/` in `indexHash`.
- **TD-08**: Page discovery must match SDK Wiki Builder: include `wiki/**/*.md`, exclude any `README.md`, exclude `wiki/index.md`, and exclude `wiki/log.md`. Only discovered pages require page frontmatter validation.
- **TD-09**: Page frontmatter validation requires `id`, `type`, `title`; `type` must be one of `api`, `workflow`, `concept`, `relation` unless `sdk-wiki.json.pageTypes` narrows the allowed list. Optional relation fields `requires`, `related`, and `apis`, when present, must contain page ids that exist in the discovered page set.

## Contracts

### Pack Root

Minimum required structure:

```text
sdk-wiki.json
raw/
wiki/
wiki/api/
wiki/workflows/
wiki/concepts/
wiki/relations/
wiki/index.md
wiki/log.md
index/
index/manifest.json
index/pages.json
index/symbols.json
index/relations.json
index/terms.json
```

`reports/` may exist but is not a required publishing surface and is not part of import hash validation.

### `sdk-wiki.json`

Runtime must accept the current builder shape:

```ts
interface SdkWikiPackManifest {
  schemaVersion: '1.0'
  sdkId: string
  name?: string
  version: string
  language?: string
  runtime?: Record<string, unknown>
  entry: string
  indexManifest?: string
  pageTypes?: Array<'api' | 'workflow' | 'concept' | 'relation'>
  sourceDocuments?: string[]
}
```

Validation:

- `schemaVersion` must equal `1.0` for MVP；
- `sdkId` and `version` must be non-empty safe path segments；
- `entry` must be a safe Pack-relative path and must exist；
- `indexManifest` defaults to `index/manifest.json` when absent and must resolve inside Pack root；
- `sourceDocuments` entries, when present, must be safe Pack-relative paths and must exist。

### `index/manifest.json`

Runtime must accept the observed builder shape:

```ts
interface SdkWikiIndexManifest {
  schemaVersion: '1.0'
  sdkId: string
  wikiVersion: string
  builtAt?: string
  contentHash?: `sha256:${string}`
  indexHash?: `sha256:${string}`
  embeddingModel?: string | null
  builderVersion?: string
  files: string[]
}
```

Validation:

- `schemaVersion` must equal `1.0`；
- `sdkId` must match `sdk-wiki.json.sdkId`；
- `wikiVersion` must match `sdk-wiki.json.version`；
- every `files[]` entry must be a safe Pack-relative file path and must exist；
- `files[]` must include the core index files listed above；
- `contentHash` / `indexHash`, when present, must be valid `sha256:<64 hex>` values and must match recomputed values。

### Builder Page Discovery

Runtime importer must use the same package/page rule as `sdk-wiki-builder`:

```text
include: wiki/**/*.md
exclude: **/README.md
exclude: wiki/index.md
exclude: wiki/log.md
```

The following files are auxiliary docs, not pages, and must not receive page frontmatter validation:

```text
wiki/README.md
wiki/api/README.md
wiki/concepts/README.md
wiki/workflows/README.md
wiki/relations/README.md
wiki/index.md
wiki/log.md
```

Valid page examples include:

```text
wiki/api/*.md
wiki/workflows/*.md
wiki/concepts/*.md
wiki/relations/*.md
```

### Page Frontmatter

For every Markdown page discovered by the builder page discovery rule:

```ts
interface SdkWikiPageFrontmatter {
  id: string
  type: 'api' | 'workflow' | 'concept' | 'relation'
  title: string
  aliases?: string[]
  terms?: string[]
  requires?: string[]
  related?: string[]
  apis?: string[]
  source_refs?: string[]
  [key: string]: unknown
}
```

Validation:

- parse with `gray-matter`；
- require `id`, `type`, `title` as strings；
- reject invalid `type`；
- reject duplicate page `id`；
- parse `index/pages.json` as an array and ensure each referenced page path exists under `wiki/`；
- when page index and frontmatter both declare `id/type/title`, they must match；
- `source_refs`, when required by the pack policy, must use `raw/<file>:<line>` or `raw/<file>`；
- `requires`, `related`, and `apis`, when present, must be arrays whose ids point to existing discovered page ids。

Runtime should read generated `index/*.json` and must not hand-edit or rewrite them during import.

### RuntimeStore Registry

`<RuntimeStoreRoot>/sdk-wikis/registry.json`:

```ts
interface SdkWikiRegistry {
  schemaVersion: '1.0'
  installed: InstalledSdkWiki[]
}

interface InstalledSdkWiki {
  sdkId: string
  version: string
  name?: string
  language?: string
  runtime?: Record<string, unknown>
  schemaVersion: string
  wikiVersion: string
  entry: string
  indexManifest: string
  pageTypes: string[]
  sourceDocuments: string[]
  installedAt: string
  status: 'ready'
  rootAlias: string
  validation: {
    fileCount: number
    pageCount: number
    contentHash?: string
    indexHash?: string
  }
}
```

`rootAlias` must be `@sdk_wiki/<sdkId>/<version>`. Renderer payloads should expose aliases and metadata, not absolute private filesystem paths.

### `install.json`

`<RuntimeStoreRoot>/sdk-wikis/<sdkId>/<version>/install.json`:

```ts
interface SdkWikiInstallMetadata {
  schemaVersion: '1.0'
  opId: string
  installedAt: string
  source: {
    kind: 'directory' | 'archive'
    basename: string
  }
  registryItem: InstalledSdkWiki
  validationReport: SdkWikiValidationReport
}
```

### Service API

```ts
type SdkWikiImportKind = 'auto' | 'directory' | 'archive'

type SdkWikiErrorCode =
  | 'SDK_WIKI_INVALID_ARGS'
  | 'SDK_WIKI_SOURCE_NOT_FOUND'
  | 'SDK_WIKI_SOURCE_UNSUPPORTED'
  | 'SDK_WIKI_SCHEMA_INVALID'
  | 'SDK_WIKI_INDEX_INVALID'
  | 'SDK_WIKI_HASH_MISMATCH'
  | 'SDK_WIKI_PAGE_INVALID'
  | 'SDK_WIKI_VERSION_UNSUPPORTED'
  | 'SDK_WIKI_ALREADY_INSTALLED'
  | 'SDK_WIKI_IMPORT_FAILED'

interface SdkWikiServiceError {
  code: SdkWikiErrorCode
  message: string
  details?: unknown
}

interface SdkWikiValidationReport {
  opId: string
  sourceKind: 'directory' | 'archive'
  sdkId?: string
  version?: string
  checks: Array<{
    name: string
    status: 'passed' | 'failed' | 'skipped'
    code?: SdkWikiErrorCode
    message?: string
  }>
  fileCount?: number
  pageCount?: number
  contentHash?: string
  indexHash?: string
}

interface SdkWikiImportResult {
  success: boolean
  sdk?: InstalledSdkWiki
  installed: InstalledSdkWiki[]
  report: SdkWikiValidationReport
  error?: SdkWikiServiceError
}
```

`SdkWikiService` methods:

- `listInstalled(): InstalledSdkWiki[]`
- `importFromPath(params: { path: string; kind?: SdkWikiImportKind }): Promise<SdkWikiImportResult>`
- optional private helpers: `validatePackRoot`, `extractArchiveToTemp`, `copyDirectoryToTemp`, `commitValidatedPack`。

### IPC API

Handlers:

- `sdk-wiki:list`
- `sdk-wiki:importPath`
- `sdk-wiki:importDirectoryDialog`
- `sdk-wiki:importArchiveDialog`

Payloads:

```ts
type SdkWikiImportPathPayload = {
  path: string
  kind?: 'auto' | 'directory' | 'archive'
}
```

Returns:

- list: `ipcOk({ sdks: InstalledSdkWiki[] })`
- import success: `ipcOk({ result: SdkWikiImportResult })`
- dialog cancel: `ipcOk({ canceled: true })`
- import failure: `ipcErr(error.code, error.message, { report: result.report, details: error.details })`

Preload wrappers:

- `sdkWikiList()`
- `sdkWikiImportPath(payload)`
- `sdkWikiImportDirectoryDialog()`
- `sdkWikiImportArchiveDialog()`

## Implementation Plan

### Tasks

- [ ] Add SDK Wiki shared types if needed, preferably under `crewagent-runtime/shared/sdkWikiTypes.ts`.
- [ ] Add RuntimeStore SDK Wiki helpers: root paths, temp paths, registry read/write, import log append, installed list.
- [ ] Add `SdkWikiService` with path kind detection, directory copy, archive extraction, validation, transactional commit.
- [ ] Implement safe path helpers or extract reusable private helpers from existing package/project KB patterns.
- [ ] Implement manifest validators for `sdk-wiki.json`, `index/manifest.json`, required files, hash checks, page frontmatter and pages index consistency.
- [ ] Implement duplicate handling: reject existing `(sdkId, version)` before final commit.
- [ ] Add thin IPC handlers and preload wrappers.
- [ ] Add tests and compact fixtures.

### Acceptance Criteria

- [ ] AC-1 directory import and archive import both work.
- [ ] AC-2 required structure and manifest fields are validated.
- [ ] AC-3 index file existence, hash, and page frontmatter invalid cases fail.
- [ ] AC-4 valid import commits to RuntimeStore, updates registry atomically, and list returns alias metadata.
- [ ] AC-5 failed import leaves registry unchanged and returns structured errors.

## Additional Context

### Dependencies

Use existing dependencies only:

- Node `>=20.9.0`
- Electron `^30.0.1`
- TypeScript `^5.2.2`
- Vitest `^4.0.16`
- `adm-zip` `^0.5.16`
- `ajv` `^8.17.1`
- `ajv-formats` `^3.0.1`
- `gray-matter` `^4.0.3`

### Testing Strategy

Unit/service tests:

- valid directory import；
- valid archive import with single top-level wrapper；
- missing `sdk-wiki.json`；
- missing `index/manifest.json`；
- unsupported `schemaVersion`；
- sdk id/version mismatch between pack and index manifest；
- missing core index file；
- `contentHash` mismatch；
- `indexHash` mismatch；
- invalid page frontmatter；
- duplicate page id；
- unsafe archive path traversal；
- duplicate `(sdkId, version)` rejected；
- failure leaves registry unchanged and temp cleaned。

RuntimeStore tests:

- registry defaults to empty；
- atomic registry update preserves existing entries；
- SDK Wiki root/temp/import log paths are under RuntimeStore root；
- list returns `rootAlias` and no absolute path。

IPC smoke tests if existing test harness supports it:

- `sdk-wiki:list` returns empty list on new store；
- `sdk-wiki:importPath` maps service failure to `ipcErr` with structured code；
- dialog cancel returns `success: true, canceled: true`。

### Notes

- Do not add `sdk_wiki.*` tool schemas to `FileSystemToolHost` in 14.1.
- Do not inject SDK Wiki registry into prompts in 14.1.
- Recompute `indexHash` only from `index/pages.json`, `index/symbols.json`, `index/terms.json`, and `index/relations.json`; observed builder excludes `index/manifest.json`, `index/README.md`, and `reports/`.
- Existing `crewagent-runtime` dirty files must not be reverted.

## Traceability

- Story: `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`
- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- Design: Story `## Design` section after `design-story 14-1`
