# Story 14.1: SDK Wiki Pack Import and Registry

Status: done

<!-- Note: This story was created with BMAD create-story after reading crewagent-runtime code, then completed with design-story. -->

## Story

作为 **Runtime User**，
我希望把一个版本化的 SDK Wiki Pack 导入 Runtime，
以便 CrewAgent 后续会话可以使用经过校验的 SDK 知识。

## Acceptance Criteria

### AC-1: Directory or archive import

**Given** 我选择一个 SDK Wiki Pack 目录或压缩包
**When** Runtime 执行导入
**Then** Runtime 必须先完成校验再注册
**And** 输入形式必须支持已展开目录和压缩包。

### AC-2: Required structure and manifest validation

**Given** Runtime 正在校验导入内容
**When** Runtime 读取 SDK Wiki Pack
**Then** Pack 必须包含：

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
index/terms.json
index/relations.json
```

**And** Runtime 必须校验 `schemaVersion`、`sdkId`、`version`、`entry` 的兼容性与有效性。
**And** `reports/` 可以存在，但不属于发布校验内容或 hash 计算输入。

### AC-3: Index, hash, and page validation

**Given** `index/manifest.json` 声明了必需索引文件或 hash
**When** Runtime 校验 SDK Wiki Pack
**Then** 必需索引文件必须存在
**And** Pack 提供 content/index hash 时必须校验 hash
**And** Runtime 必须按 SDK Wiki Builder 的 page discovery 规则识别页面：

- include: `wiki/**/*.md`
- exclude: 任意目录下的 `README.md`
- exclude: `wiki/index.md`
- exclude: `wiki/log.md`

**And** Runtime 不得把以下辅助文件当作 page，也不得对它们执行 page frontmatter 校验：

- `wiki/README.md`
- `wiki/api/README.md`
- `wiki/concepts/README.md`
- `wiki/workflows/README.md`
- `wiki/relations/README.md`
- `wiki/index.md`
- `wiki/log.md`

**And** 对 discovery 得到的结构化 Markdown page，frontmatter 必须可解析且 `id`、`type`、`title` 字段有效，否则拒绝导入。
**And** `requires`、`related`、`apis` 中的每个 id 必须指向已存在 page id。
**And** `indexHash` 重算只使用 `index/pages.json`、`index/symbols.json`、`index/terms.json`、`index/relations.json`，不得包含 `index/manifest.json` 或 `index/README.md`。

### AC-4: Transactional RuntimeStore registration

**Given** SDK Wiki Pack 通过校验
**When** 导入完成
**Then** Runtime 必须把 Pack 存储在：

```text
RuntimeStore/sdk-wikis/<sdkId>/<version>/
```

**And** 必须原子更新 SDK Wiki registry
**And** 已安装 SDK Wiki 可被列出，列表项至少包含 `sdkId`、`version`、状态和 Runtime alias。

### AC-5: Failure safety

**Given** 导入在任一校验步骤失败
**When** 我查看已安装 SDK Wiki
**Then** 失败 Pack 不得出现在 registry 中
**And** 不得留下部分注册状态
**And** 失败结果必须返回结构化错误码，例如：

- `SDK_WIKI_SCHEMA_INVALID`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_HASH_MISMATCH`
- `SDK_WIKI_PAGE_INVALID`
- `SDK_WIKI_VERSION_UNSUPPORTED`

## Design

### Summary

- 14.1 采用 service + RuntimeStore + 最小 IPC 方案，不新增可视化 SDK Wiki 管理页。
- 导入统一经过 temp workspace：目录先复制，压缩包先安全解压，再校验并事务提交。
- Pack contract 对齐当前 `sam` / `abaqus` SDK Wiki Pack 样例：`sdk-wiki.json` + `index/manifest.json` + `index/pages.json|symbols.json|relations.json|terms.json`。
- 重复 `(sdkId, version)` 默认拒绝，不做 replace/reinstall。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-1-sdk-wiki-pack-import-and-registry.md`

### UX / UI

- 不新增可见页面、设置项或管理面板。
- Runtime 用户选择目录/压缩包的入口通过最小 IPC dialog 支持，未来 UI 可复用：
  - `sdk-wiki:importDirectoryDialog`
  - `sdk-wiki:importArchiveDialog`
- 用户取消选择时返回 `success: true, canceled: true`，不写 log、不修改 registry。
- 导入失败时 IPC 返回结构化错误码和 validation report；具体展示方式留给后续 UI story。

### API / Contracts

新增 `SdkWikiService`：

```ts
type SdkWikiImportKind = 'auto' | 'directory' | 'archive'

interface SdkWikiImportPathPayload {
  path: string
  kind?: SdkWikiImportKind
}

interface SdkWikiImportResult {
  success: boolean
  sdk?: InstalledSdkWiki
  installed: InstalledSdkWiki[]
  report: SdkWikiValidationReport
  error?: SdkWikiServiceError
}
```

Service methods:

- `listInstalled(): InstalledSdkWiki[]`
- `importFromPath(params: { path: string; kind?: SdkWikiImportKind }): Promise<SdkWikiImportResult>`

IPC handlers:

- `sdk-wiki:list` -> `ipcOk({ sdks })`
- `sdk-wiki:importPath` with `{ path, kind? }`
- `sdk-wiki:importDirectoryDialog`
- `sdk-wiki:importArchiveDialog`

Preload wrappers:

- `sdkWikiList()`
- `sdkWikiImportPath(payload)`
- `sdkWikiImportDirectoryDialog()`
- `sdkWikiImportArchiveDialog()`

Pack manifest contract:

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

Index manifest contract:

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

Structured error codes for this story:

- `SDK_WIKI_INVALID_ARGS`
- `SDK_WIKI_SOURCE_NOT_FOUND`
- `SDK_WIKI_SOURCE_UNSUPPORTED`
- `SDK_WIKI_SCHEMA_INVALID`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_HASH_MISMATCH`
- `SDK_WIKI_PAGE_INVALID`
- `SDK_WIKI_VERSION_UNSUPPORTED`
- `SDK_WIKI_ALREADY_INSTALLED`
- `SDK_WIKI_IMPORT_FAILED`

### Data / Storage

Use RuntimeStore private storage only:

```text
<RuntimeStoreRoot>/
  sdk-wikis/
    registry.json
    import-log.ndjson
    <sdkId>/
      <version>/
        sdk-wiki.json
        raw/
        wiki/
        index/
        install.json
  tmp/
    sdk-wiki-import-<opId>/
      extracted/
      validation-report.json
```

`registry.json`:

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

Storage rules:

- `rootAlias` must be `@sdk_wiki/<sdkId>/<version>`。
- Renderer payloads expose alias and metadata, not absolute RuntimeStore paths。
- `install.json` stores `opId`, source kind/basename, registry item, and validation report。
- `import-log.ndjson` is append-only and records success/failure with `opId`, `sdkId`, `version`, error code, timestamps。
- `contentHash` validation follows SDK Wiki Builder algorithm over sorted manifest files in `sdk-wiki.json`, `raw/`, `wiki/`。
- `indexHash` validation follows the same algorithm but only over `index/pages.json`、`index/symbols.json`、`index/terms.json`、`index/relations.json`; it must exclude `index/manifest.json`、`index/README.md`、`reports/`。

### Errors / Edge Cases

- Missing source path -> `SDK_WIKI_SOURCE_NOT_FOUND`
- Source is neither directory nor supported archive -> `SDK_WIKI_SOURCE_UNSUPPORTED`
- Archive entry has `..`, absolute path, Windows drive path, NUL byte, or escapes extraction dir -> `SDK_WIKI_IMPORT_FAILED`
- Missing `sdk-wiki.json` or invalid JSON -> `SDK_WIKI_SCHEMA_INVALID`
- `schemaVersion !== "1.0"` in pack or index manifest -> `SDK_WIKI_VERSION_UNSUPPORTED`
- Unsafe or missing `sdkId`, `version`, `entry`, `indexManifest`, `sourceDocuments` path -> `SDK_WIKI_SCHEMA_INVALID`
- Missing `index/manifest.json`, invalid `files[]`, missing core index files, or pack/index sdk mismatch -> `SDK_WIKI_INDEX_INVALID`
- Missing file listed by `index/manifest.json.files` -> `SDK_WIKI_INDEX_INVALID`
- `contentHash` or `indexHash` invalid/mismatched -> `SDK_WIKI_HASH_MISMATCH`
- Markdown frontmatter cannot parse, lacks `id/type/title`, has invalid page type, or duplicate page id -> `SDK_WIKI_PAGE_INVALID`
- `index/pages.json` entry path missing or inconsistent with page frontmatter -> `SDK_WIKI_PAGE_INVALID`
- Duplicate `(sdkId, version)` -> `SDK_WIKI_ALREADY_INSTALLED`
- Commit, registry write, or cleanup failure -> `SDK_WIKI_IMPORT_FAILED`; registry must remain unchanged when import is not complete。

### Test Plan

Service tests:

- valid directory import registers SDK Wiki and returns list item；
- valid archive import registers SDK Wiki and normalizes single top-level wrapper；
- missing `sdk-wiki.json` fails；
- missing `index/manifest.json` fails；
- unsupported pack/index schema fails；
- sdk id/version mismatch between pack and index manifest fails；
- missing core index file fails；
- `contentHash` mismatch fails；
- `indexHash` mismatch fails；
- `index/README.md` does not affect `indexHash`；
- builder auxiliary Markdown files (`README.md`, `wiki/index.md`, `wiki/log.md`) are not treated as pages；
- invalid page frontmatter fails；
- page/index relation targets that reference unknown page ids fail；
- duplicate page id fails；
- unsafe archive path traversal fails；
- duplicate `(sdkId, version)` fails；
- failed validation leaves registry unchanged and temp workspace removed。

RuntimeStore tests:

- SDK Wiki registry defaults to empty；
- atomic registry update preserves existing entries；
- SDK Wiki root/temp/import log paths stay under RuntimeStore root；
- list payload includes stable `rootAlias` and no absolute private path。

IPC smoke tests if existing harness supports them:

- `sdk-wiki:list` returns empty list on new store；
- `sdk-wiki:importPath` maps service failure to structured `ipcErr`；
- dialog cancel returns `{ success: true, canceled: true }`。

## Developer Context

### Story Boundary

14.1 只建立 SDK Wiki Pack 的导入、校验、注册和列表基础能力。

In scope:

- 从目录或压缩包导入 SDK Wiki Pack；
- 校验 `sdk-wiki.json`、`index/manifest.json`、必需目录、必需索引文件、schema/version、hash 和 `wiki/` frontmatter；
- 以事务方式写入 RuntimeStore `sdk-wikis/<sdkId>/<version>/`；
- 原子更新 `sdk-wikis/registry.json`；
- 记录导入日志和 validation report；
- 提供 installed SDK Wikis 的 list payload，供 14.2 暴露 `sdk_wiki.list_sdks` 或等价能力。

Out of scope:

- 不实现 `sdk_wiki.search_pages/read_page/resolve_symbol/expand_relations/plan_api_usage`，这些属于 14.2/14.3；
- 不实现 SDK execution risk confirmation gate，这属于 14.4；
- 不生成或重建 SDK Wiki index，Runtime 只消费外部 Pack 作者提供的索引；
- 不把 SDK Wiki 内容导入 Project KB、Personal KB 或 agent skill；
- 不要求完成 UI，除非 design-story 明确选择最小管理入口；
- 不硬编码 SAM 私有 API。SAM 只能作为 fixture/example。

### Runtime Code Intelligence

- `RuntimeStore` 是新增 SDK Wiki 域的正确落点。它已经把 private root 放在 Electron `app.getPath('userData')/runtime-store`，并已有 package、project、skills、settings、KB 等域的持久化边界。
- `RuntimeStore.importPackage()` 已有 staged extraction、zip root prefix 处理和 `finally` cleanup 模式。14.1 应复用这个事务思路，但不要把 SDK Wiki Pack 当 BMAD package 注册。
- `RuntimeStore.writeJsonAtomic()` 已是 JSON 原子写 helper。SDK Wiki registry 和 `install.json` 应通过同类原子写，避免手写非原子 JSON 写入。
- `ProjectKbService` 是最接近的 service pattern：构造函数注入 RuntimeStore、使用 `AdmZip`、使用 workspace staging、校验 archive 内容、写 manifest/log、测试 service 层。14.1 应新增 `SdkWikiService`，不要把导入逻辑塞进 `main.ts` 或 `FileSystemToolHost`。
- `FileSystemToolHost` 当前通过 callback 暴露 `project_kb.retrieve`。14.1 不应直接添加 `sdk_wiki.*` tool dispatch；后续 14.2 可采用同样 callback 注入 `SdkWikiService` 查询能力。
- `main.ts` 的 `ipcMain.handle(...)` 分组模式可作为 IPC 入口参考。如果 14.1 暴露 renderer API，应把业务逻辑留在 service，IPC 只做 payload 校验、dialog、调用和错误包装。
- `SkillRegistryService` 的 compact registry injection 是后续 14.2 的参考。14.1 只需 registry/list 数据结构对齐，不做 prompt injection。

### Technical Requirements

- Pack schema 版本 MVP 只接受当前文档约定的兼容版本。unsupported version 必须返回结构化错误，不得 best-effort 导入。
- Archive extraction 必须先进入 temp workspace，并对每个 entry 做 safe relative path 校验。必须拒绝 `..`、absolute path、Windows drive path、NUL byte 和解压后逃逸目标目录的路径。
- Archive 允许包含单一顶层目录 wrapper；导入服务应像现有 package import 一样把 wrapper 归一化为 Pack root。
- Directory import 也必须复制到 temp 后校验，不得直接把用户选择路径注册为 RuntimeStore 根。
- `entry` 必须解析为 Pack 内相对路径，并且必须落在 `wiki/` 或 design-story 明确允许的入口范围内。
- `sdkId` 和 `version` 必须可用于目录名和 alias。必须拒绝空值、路径分隔符、父目录、协议样式值和大小写/空白导致的歧义。
- 已安装 registry item 的 alias 必须稳定，建议：`@sdk_wiki/<sdkId>/<version>`。
- Registry update 必须满足：校验成功前不可写 registry；commit 失败不可写 registry；registry 写失败时不得留下 visible installed item。
- `install.json` 必须记录 Runtime 侧 metadata 和 validation summary，不替代 Pack 自带的 `sdk-wiki.json`。
- Import log 使用 append-only `import-log.ndjson`，至少记录 op id、sdk id、version、status、error code、startedAt/finishedAt。

### Architecture Compliance

- SDK Wiki 是 RuntimeStore first-class domain，不能放在 ProjectRoot，也不能复用 Project KB manifest。
- Runtime 不自动 rebuild SDK Wiki index。缺失、损坏或不兼容索引必须导入失败。
- 14.1 的 list payload 必须足够让 14.2 在 session context 中注入 compact registry，但不能预加载 wiki page 内容。
- 导入成功后的 Pack 文件应视为 read-only registered asset。后续读取必须始终限制在 registered pack root 内。
- 结构化错误需要同时适合 IPC 返回和 service test 断言，避免只抛人类可读字符串。

### Library and Framework Requirements

使用 `crewagent-runtime/package.json` 中已有版本，不为 14.1 引入新依赖：

- Node engine: `>=20.9.0`
- Electron: `^30.0.1`
- TypeScript: `^5.2.2`
- Vitest: `^4.0.16`
- `adm-zip`: `^0.5.16`
- `ajv`: `^8.17.1`
- `ajv-formats`: `^3.0.1`
- `gray-matter`: `^4.0.3`

Implementation guidance:

- 用 `AdmZip` 处理 zip；
- 用 Node `fs/path/crypto` 处理文件、路径、SHA-256；
- 用 `gray-matter` 解析 wiki Markdown frontmatter；
- 如果 design-story 定义 JSON schema，则用现有 `Ajv2020` + `addFormats` 模式；
- 不引入 SQLite、BM25、embedding 或 LLM 调用。这些不是 14.1 的需求。

### File Structure Requirements

Expected implementation files:

- `crewagent-runtime/electron/services/sdkWikiService.ts` 新增 service；
- `crewagent-runtime/electron/services/sdkWikiService.test.ts` 新增 service tests；
- `crewagent-runtime/electron/stores/runtimeStore.ts` 添加 SDK Wiki storage helpers 和 registry helpers；
- `crewagent-runtime/electron/stores/runtimeStore.test.ts` 添加 store-level tests；
- `crewagent-runtime/electron/main.ts` 仅在 design-story 选择 IPC/import entrypoint 时修改；
- `crewagent-runtime/electron/preload.ts` 仅在 renderer 需要访问 API 时修改；
- `crewagent-runtime/shared/...` 仅在需要共享 typed contract 时新增或修改。

Do not modify for 14.1:

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`，除非 design-story 明确要求 list API 暴露为 tool；
- `crewagent-runtime/electron/services/chatToolLoop.ts`；
- `crewagent-runtime/electron/services/executionEngine.ts`；
- SDK risk policy 或 MCP execution code。

### Testing Requirements

Required test coverage:

- valid directory import registers SDK Wiki and returns list item；
- valid archive import registers SDK Wiki and normalizes single top-level wrapper；
- missing `sdk-wiki.json` fails；
- missing `index/manifest.json` fails with `SDK_WIKI_INDEX_INVALID`；
- unsupported pack/index schema fails with `SDK_WIKI_VERSION_UNSUPPORTED` or `SDK_WIKI_SCHEMA_INVALID`；
- missing required index file fails；
- hash mismatch fails with `SDK_WIKI_HASH_MISMATCH`；
- invalid Markdown frontmatter fails with `SDK_WIKI_PAGE_INVALID`；
- unsafe archive path traversal fails and leaves registry unchanged；
- duplicate `(sdkId, version)` follows design contract；
- failed validation removes temp workspace and does not create visible installed item；
- registry write uses atomic helper path and preserves existing installed entries。

Use compact fixtures under the existing test style. Fixtures should be generic; SAM naming is allowed only for example metadata, not for hardcoded logic.

### Previous Story Intelligence

- 这是 Epic 14 的第一条 story，没有前序 Epic 14 story 可继承。
- 需要吸收现有 Epic 12 Project KB 和 Epic 13 Skill Registry 的代码模式，但不能把 SDK Wiki 建成 KB 或 Skill 的子功能。

### Git / Brownfield Intelligence

- `crewagent-runtime` 当前工作区已有与本 story 无关的 dirty changes：`package-lock.json`、`package.json`、`src/pages/WorksPage/...`。后续 dev-story 必须避免回退这些用户/既有变更。
- 最近 `crewagent-runtime` 相关提交主要是 runtime submodule/package update，没有为 SDK Wiki 建立过现成实现。
- 本 create-story 阶段不修改 runtime 源码。

### Project Context Reference

Source documents and code read for this story:

- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/README.md`
- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/01-sdk-wiki-importer.md`
- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/02-sdk-llm-wiki-module.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `crewagent-runtime/package.json`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/skillRegistryService.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`

## Tasks / Subtasks

- [x] Task 1: RuntimeStore SDK Wiki storage helpers (AC: 4,5)
  - [x] 1.1 Add SDK Wiki root, registry path, import temp path, and import log path helpers
  - [x] 1.2 Add atomic registry read/write helpers
  - [x] 1.3 Add installed SDK Wiki list helper with stable `@sdk_wiki/<sdkId>/<version>` alias
  - [x] 1.4 Add append-only SDK Wiki import log helper

- [x] Task 2: SDK Wiki import service (AC: 1,2,3,5)
  - [x] 2.1 Add `SdkWikiService` with directory and archive import entrypoints
  - [x] 2.2 Safely extract archives and reject path traversal
  - [x] 2.3 Copy directory imports into temp workspace before validation
  - [x] 2.4 Validate required pack structure and parse `sdk-wiki.json`
  - [x] 2.5 Validate `index/manifest.json`, required index files, and schema/version compatibility
  - [x] 2.6 Validate hashes when present
  - [x] 2.7 Parse and validate `wiki/` page frontmatter
  - [x] 2.8 Return structured validation report and error codes
  - [x] 2.9 Align page discovery, relation target validation, and `indexHash` calculation with SDK Wiki Builder package rules

- [x] Task 3: Transactional commit and listing (AC: 4,5)
  - [x] 3.1 Stage import under temp workspace
  - [x] 3.2 Commit valid pack into `sdk-wikis/<sdkId>/<version>/`
  - [x] 3.3 Write `install.json` with validation summary
  - [x] 3.4 Update registry atomically
  - [x] 3.5 Return installed SDK Wiki list payload
  - [x] 3.6 Remove temp workspace after success or failure

- [x] Task 4: IPC / developer entrypoint (AC: 1,4)
  - [x] 4.1 Add main-process IPC handlers for import/list if design-story keeps developer import in scope
  - [x] 4.2 Add preload wrappers only if renderer access is included
  - [x] 4.3 Keep UI optional for this story unless explicitly chosen in design-story

- [x] Task 5: Tests and fixtures (AC: 1-5)
  - [x] 5.1 Add minimal valid generic/SAM-like SDK Wiki Pack fixture
  - [x] 5.2 Test successful directory import and list
  - [x] 5.3 Test successful archive import and wrapper normalization
  - [x] 5.4 Test missing `index/manifest.json`
  - [x] 5.5 Test unsupported schema/index version
  - [x] 5.6 Test missing required index file
  - [x] 5.7 Test hash mismatch
  - [x] 5.8 Test invalid page frontmatter
  - [x] 5.9 Test archive path traversal rejection
  - [x] 5.10 Test failed import leaves registry unchanged
  - [x] 5.11 Test builder auxiliary Markdown exclusion, `index/README.md` hash exclusion, and relation target rejection

## Dev Notes

### Codebase Findings

- `RuntimeStore` already owns private paths under Electron `app.getPath('userData')/runtime-store`, including `packages`, `projects`, `skills/global`, settings, personal KB, and project KB.
- `RuntimeStore.importPackage()` already uses a staged extraction path and cleans temp install directories in `finally`.
- `ProjectKbService` is the best model for a service with RuntimeStore dependency, manifest IO, safe archive handling, and tests.
- `project_kb.retrieve` is exposed through `FileSystemToolHost` by callback, but 14.1 should not add `sdk_wiki.*` tools yet. Tool exposure belongs to Story 14.2.
- `AdmZip`, `gray-matter`, `Ajv2020`, and `addFormats` are already present in runtime code and can be reused.

### Project Structure Notes

- Keep SDK Wiki storage as a new RuntimeStore domain, not a subfolder of project KB.
- Keep SDK Wiki import logic in `electron/services/sdkWikiService.ts`; keep persistence helpers in `electron/stores/runtimeStore.ts`.
- Keep IPC handlers thin and grouped near existing package/KB handlers if included.
- Keep all test fixtures small and deterministic; do not require real SAM SDK content.

### References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/01-sdk-wiki-importer.md`
- `crewagent-runtime/package.json`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/projectKbService.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/main.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `npm run test -- electron/services/sdkWikiService.test.ts electron/stores/runtimeStore.test.ts` passed: 2 test files, 101 tests.
- Review follow-up `npm run test -- electron/services/sdkWikiService.test.ts electron/stores/runtimeStore.test.ts` passed: 2 test files, 105 tests.
- `npm run build:ci` passed: TypeScript and Vite production build completed; warnings were existing package/Vite warnings only.
- Review follow-up `npm run build:ci` passed after H1/M1/M2 fixes.
- Targeted ESLint passed for 14.1 touched files with `--max-warnings 0`.
- Review follow-up targeted ESLint passed for 14.1 touched files with `--max-warnings 0`.
- `git diff --check` passed for 14.1 touched runtime files.
- Full `npm run lint` is blocked by an existing unrelated warning in `src/components/workflow/WorkflowGraphView.tsx:52`.
- Full `npm run test` currently fails only in existing unrelated `electron/services/fileSystemToolHost.test.ts` case `runs shell.exec when explicitly enabled`; that single case passes when run directly.
- Builder-rule follow-up `npm test -- electron/services/sdkWikiService.test.ts` passed: 1 test file, 35 tests.
- Builder-rule follow-up `npm test -- electron/services/sdkWikiService.test.ts electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts src/pages/SettingsPage/SettingsPage.test.tsx` passed: 4 test files, 142 tests.
- Builder-rule follow-up `npm run build:ci` passed; remaining warnings are existing Vite package/chunk warnings.
- Builder-rule follow-up targeted ESLint passed for `electron/services/sdkWikiService.ts` and `electron/services/sdkWikiService.test.ts` with `--max-warnings 0`.
- Builder-rule follow-up `npm exec tsc -- --noEmit` passed.

### Completion Notes List

- Implemented first-class SDK Wiki RuntimeStore domain with private storage paths, atomic registry helpers, stable root aliases, install metadata, and append-only import logs.
- Implemented `SdkWikiService` for staged directory/archive imports, safe archive extraction, required structure/schema/index/hash/page validation, transactional commit, structured reports, and cleanup on failure.
- Added main-process IPC handlers and preload/renderer typings for list/import path/dialog entrypoints; no visible UI was added for this story.
- Added compact service and store tests covering successful imports, archive wrapper normalization, schema/index/hash/page failures, unsafe archives, duplicate installs, and registry safety.
- Addressed code review findings: all builder-discovered `wiki/**/*.md` pages are validated, archive entries containing `..` are rejected before normalization, and staged packs containing symlinks are rejected before registration.
- Aligned importer with SDK Wiki Builder package rules: `README.md`, `wiki/index.md`, and `wiki/log.md` are auxiliary files, only builder-discovered pages get frontmatter validation, `indexHash` uses the four core index JSON files, and relation targets must resolve to existing page ids.

### File List

- `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/code-review-14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/src/vite-env.d.ts`

### Change Log

- 2026-05-13 - Implemented SDK Wiki import/registry runtime support with tests; story moved to review.
- 2026-05-14 - Addressed code review H1/M1/M2 import validation safety findings; review outcome updated to approved.
- 2026-05-14 - Story accepted after approved code review; status moved to done.
- 2026-05-14 - Aligned 14.1 importer and docs with SDK Wiki Builder package/page/hash rules; added regression coverage.
