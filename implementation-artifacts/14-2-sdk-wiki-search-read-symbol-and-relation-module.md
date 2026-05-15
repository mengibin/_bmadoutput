# Story 14.2: SDK Wiki Search, Read, Symbol, and Relation Module

Status: done

<!-- Created after Story 14.1 import/registry implementation and code review were approved. -->

## Story

作为 **Runtime User**，
我希望主 LLM 能按需搜索和读取已导入的 SDK Wiki 内容，
以便 SDK 回答、API 选择和关系推理都能基于已注册的来源页面。

## Acceptance Criteria

### AC-1: Compact SDK Wiki registry in session context

**Given** 至少一个 SDK Wiki 已安装且状态为 `ready`
**When** Runtime 组合 chat/agent/run session context
**Then** Runtime 必须暴露 compact SDK Wiki registry
**And** registry 只包含 `sdkId`、`version`、名称、语言、page types、source documents、root alias、可用工具提示等元数据
**And** 不得把完整 wiki 页面内容预加载进 prompt。

### AC-2: `sdk_wiki.list_sdks`

**Given** LLM 调用 `sdk_wiki.list_sdks`
**When** Runtime 查询已安装 SDK Wiki registry
**Then** Runtime 返回 bounded installed SDK list
**And** 每个 SDK 至少包含 `sdkId`、`version`、`status`、`rootAlias`、`pageCount`、`contentHash`/`indexHash`（如果存在）
**And** 不暴露 RuntimeStore 绝对路径。

### AC-3: `sdk_wiki.search_pages`

**Given** LLM 调用 `sdk_wiki.search_pages`，并提供 `query`，可选 `sdkId`、`version`、`pageTypes`、`topK`
**When** query 命中 `index/pages.json`、`index/terms.json` 或 `index/symbols.json`
**Then** Runtime 返回 bounded results
**And** 每条结果包含 `sdkId`、`version`、`pageId`、`pageType`、`title`、`path`、`score`、`summary/snippet`（如果可用）和 `sourceRefs`（如果页面 frontmatter 提供）
**And** 搜索必须使用已导入 index，不得递归扫描任意 filesystem path。

### AC-4: `sdk_wiki.read_page`

**Given** LLM 调用 `sdk_wiki.read_page` 并提供 `sdkId`、`pageId`，可选 `version`、`mode`、`maxBytes`
**When** page id 存在于已注册 SDK Wiki 的 `index/pages.json`
**Then** Runtime 只从 registered SDK Wiki root 下读取对应 Markdown 文件
**And** 返回 structured page metadata、frontmatter、Markdown content 或 bounded content slice
**And** 返回结果必须标明是否截断、原始 `pageId` 和 source refs。

### AC-5: `sdk_wiki.resolve_symbol`

**Given** LLM 调用 `sdk_wiki.resolve_symbol` 并提供 `symbol`，可选 `sdkId`、`version`
**When** `index/symbols.json` 包含该 symbol 或可做大小写/短名精确归一匹配
**Then** Runtime 返回对应 page refs
**And** 不命中时返回结构化 `no_match`，不得编造 API 名称。

### AC-6: `sdk_wiki.expand_relations`

**Given** LLM 调用 `sdk_wiki.expand_relations` 并提供 `pageId` 或 `symbol`
**When** `index/relations.json` 包含 requires/related/apis 等关系
**Then** Runtime 返回 bounded relation graph refs
**And** 每个 relation target 必须能解析到已注册 page id，无法解析的 target 要进入 `missingTargets`。

### AC-7: Internal tool integration and failure safety

**Given** 当前没有 ready SDK Wiki
**When** Runtime 暴露工具或 context function
**Then** `sdk_wiki.*` 能力应不可见或返回 empty/no_match，而不是失败泄漏内部路径。

**Given** index file 损坏、page 路径越界、page 缺失或 JSON 解析失败
**When** LLM 调用任一 `sdk_wiki.*` 能力
**Then** Runtime 返回结构化错误码
**And** 不修改 registry、project files、KB、skills 或 package 文件。

## Design

### Summary

- 14.2 采用 read-only index reader + service facade + ToolHost callback + compact system message 方案。
- 新增 `SdkWikiIndexReader` 负责读取已注册 pack 的 `index/*.json` 和 `wiki/*.md`，并把 root containment、version selection、bounded output 收拢在 service 层。
- `FileSystemToolHost` 暴露 internal tools：`sdk_wiki.list_sdks/search_pages/read_page/resolve_symbol/expand_relations`。
- `ExecutionEngine` 在 chat/agent/run prompt 中注入 compact SDK Wiki registry system message；不把完整页面内容放入初始上下文。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-14-2-sdk-wiki-search-read-symbol-and-relation-module.md`

### API / Tool Contracts

Tool names:

- `sdk_wiki.list_sdks`
- `sdk_wiki.search_pages`
- `sdk_wiki.read_page`
- `sdk_wiki.resolve_symbol`
- `sdk_wiki.expand_relations`

Payloads:

```ts
type SdkWikiVersionSelector = {
  sdkId?: string
  version?: string
}

type SdkWikiSearchPagesPayload = SdkWikiVersionSelector & {
  query: string
  pageTypes?: Array<'api' | 'workflow' | 'concept' | 'relation'>
  topK?: number
}

type SdkWikiReadPagePayload = {
  sdkId: string
  version?: string
  pageId: string
  mode?: 'content' | 'full'
  maxBytes?: number
}

type SdkWikiResolveSymbolPayload = SdkWikiVersionSelector & {
  symbol: string
  topK?: number
}

type SdkWikiExpandRelationsPayload = SdkWikiVersionSelector & {
  pageId?: string
  symbol?: string
  depth?: 1 | 2
}
```

Service methods:

- `listSdks(params?: SdkWikiVersionSelector)`
- `searchPages(params: SdkWikiSearchPagesPayload)`
- `readPage(params: SdkWikiReadPagePayload)`
- `resolveSymbol(params: SdkWikiResolveSymbolPayload)`
- `expandRelations(params: SdkWikiExpandRelationsPayload)`
- `formatSdkWikiRegistrySystemMessage(): string | null`

Result rules:

- Tool results return `ok: true` with `mode`/`items`/`page` payloads, or `ok: false` with structured `error`。
- No result may include RuntimeStore absolute paths.
- `search_pages` and `resolve_symbol` return bounded lists.
- `read_page` returns bounded content by default and indicates `truncated`.
- `expand_relations` returns resolved refs plus `missingTargets`.

### Index Reader

`SdkWikiIndexReader` reads from:

```text
<RuntimeStoreRoot>/sdk-wikis/<sdkId>/<version>/
  sdk-wiki.json
  wiki/
  index/
    pages.json
    symbols.json
    relations.json
    terms.json
```

Reader responsibilities:

- load and normalize pages index;
- load symbol/term/relation maps;
- resolve page id to a wiki path from `pages.json`;
- enforce root containment before reading Markdown;
- parse frontmatter with `gray-matter`;
- expose deterministic read-only query primitives.

No recursive wiki scanning is needed in 14.2; `index/pages.json` remains authoritative for readable pages.

### ToolHost Integration

- Add SDK Wiki result types to `toolHost.ts` and extend `ToolResult`.
- Add optional `onSdkWikiToolCall` callback to `FileSystemToolHostOptions`.
- Add tool schemas in `getVisibleTools()` when ready SDK Wikis exist, or expose them always with empty/no_match semantics if design implementation chooses simpler behavior.
- Add dispatch branch for `sdk_wiki.*`, validate primitive args, clamp `topK` and `maxBytes`, then call `onSdkWikiToolCall`.
- Wire `main.ts` callback to `sdkWikiService`.

### Compact Registry Context

- Add `formatSdkWikiRegistrySystemMessage(sdks)` style helper.
- Inject via existing `ExecutionEngine` extra system message path, similar to skill registry context.
- Message should list installed SDKs and available SDK Wiki tool names.
- Message must instruct the LLM to search/read/resolve before making SDK API claims.
- Message must not include full Markdown page content.

### Error Codes

Minimum structured errors:

- `SDK_WIKI_INVALID_ARGS`
- `SDK_WIKI_NOT_INSTALLED`
- `SDK_WIKI_VERSION_REQUIRED`
- `SDK_WIKI_PAGE_NOT_FOUND`
- `SDK_WIKI_SYMBOL_NOT_FOUND`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_READ_FAILED`

### Test Plan

Service tests:

- compact list;
- search by title/id/summary;
- search by terms index;
- search by symbols index;
- read page bounded/full modes;
- read page not found and path containment failure;
- resolve symbol hit/no_match;
- expand relations with resolved refs and missing targets;
- multi-version ambiguity.

ToolHost tests:

- visible tools with ready SDK Wiki registry;
- dispatch validates and forwards list/search/read/resolve/expand;
- callback absence returns structured internal error;
- no absolute paths in results.

Prompt/context tests:

- compact registry message appears when SDK Wiki ready entries exist;
- full page Markdown is not injected;
- guidance tells model to use SDK Wiki tools before SDK API claims.

## Developer Context

### Story Boundary

14.2 只实现已安装 SDK Wiki 的只读搜索、读取、symbol 解析和 relation 展开。

In scope:

- `sdk_wiki.list_sdks`
- `sdk_wiki.search_pages`
- `sdk_wiki.read_page`
- `sdk_wiki.resolve_symbol`
- `sdk_wiki.expand_relations`
- compact SDK Wiki registry session context message
- ToolHost / main process callback integration
- bounded read/search result shape and structured errors

Out of scope:

- SDK Wiki Pack management UI and remove，属于 Story 14.3；
- `sdk_wiki.plan_api_usage`，属于 Story 14.4；
- SDK tool risk confirmation gate，属于 Story 14.5；
- SAM golden path / generic adapter contract，属于 Story 14.6；
- index rebuild/generation、embedding、BM25/SQLite；
- 可视化 SDK Wiki 管理页；
- 把 SDK Wiki 导入 Project KB、Personal KB 或 skill；
- 外部 MCP tool 注册。`sdk_wiki.*` 是 Runtime internal ability。

### Runtime Code Intelligence

- Story 14.1 已新增 `SdkWikiService`、`shared/sdkWikiTypes.ts`、RuntimeStore SDK Wiki registry/helpers、IPC import/list 能力。
- `RuntimeStore.listInstalledSdkWikis()` 返回不含绝对路径的 registry item；读取页面时应使用 `RuntimeStore.getSdkWikiInstallRoot(sdkId, version)` 作为唯一 root。
- `SdkWikiService` 当前只负责 import/list。14.2 可在同一 service 中添加只读 query 方法，或新增 `SdkWikiIndexReader` 并由 service 委托。
- 已观察本地 `sam` / `abaqus` pack index 形状：
  - `index/pages.json`: array of `{ id, type, title, path, summary?, module?, risk? }`
  - `index/symbols.json`: object map `symbol -> pageId[]`
  - `index/terms.json`: object map `term -> pageId[]`
  - `index/relations.json`: object map `pageId -> { requires?: string[], related?: string[], apis?: string[] }`
- `FileSystemToolHost` 已用 callback 暴露 `project_kb.retrieve`，14.2 可复用该模式添加 `onSdkWikiToolCall` 或多个 typed callbacks。
- `ToolResult` union 目前只包含 Project KB result，不含 SDK Wiki result；14.2 需要扩展 shared tool result types。
- `getVisibleTools()` 是暴露 OpenAI function tools 的入口；`dispatch()` 是 tool call 路由入口。
- `main.ts` 已实例化 `sdkWikiService`，但 `FileSystemToolHost` 当前还没有 SDK Wiki callback 注入。
- `PromptComposer` 的 tool policy/system context 可增加 compact SDK Wiki registry guidance；不要把 full page content 放入系统消息。

### Technical Requirements

- Read-only: 所有 `sdk_wiki.*` 调用不得写入 ProjectRoot、RuntimeStore registry、Package、KB、Skill 或 MCP state。
- Root containment: `read_page` 必须通过 `index/pages.json` 的 page path 解析，并确认最终路径在 `RuntimeStore/sdk-wikis/<sdkId>/<version>/` 内。
- Version selection:
  - 如果指定 `version`，只查该版本；
  - 如果未指定且同一 `sdkId` 只有一个 ready version，可自动选中；
  - 如果未指定且同一 `sdkId` 有多个 ready versions，返回 `SDK_WIKI_VERSION_REQUIRED` 或等价结构化错误。
- Bounded output:
  - `topK` 建议默认 5，上限 12；
  - snippets 建议每条不超过 500 chars；
  - `read_page` 默认 bounded content，允许 `mode: "full"` 但必须受 `maxBytes`/hard limit 保护。
- Search scoring MVP 可用 deterministic heuristic：
  - exact page id/title/symbol match 权重最高；
  - terms/symbols index 命中次之；
  - page summary/title/path 包含 query 作为补充；
  - 不引入 embedding 或 LLM 调用。
- `sourceRefs` 优先来自 page frontmatter `source_refs`，没有时可为空数组。
- Relation expansion 只使用 `index/relations.json` 和 page/symbol index，不递归扫描 `wiki/`。
- Structured errors should be assertable in tests, e.g.:
  - `SDK_WIKI_NOT_INSTALLED`
  - `SDK_WIKI_VERSION_REQUIRED`
  - `SDK_WIKI_PAGE_NOT_FOUND`
  - `SDK_WIKI_SYMBOL_NOT_FOUND`
  - `SDK_WIKI_INDEX_INVALID`
  - `SDK_WIKI_READ_FAILED`
  - `SDK_WIKI_INVALID_ARGS`

### Architecture Compliance

- SDK Wiki remains a RuntimeStore first-class domain.
- Do not expose RuntimeStore absolute paths to renderer or LLM tool results.
- Do not add SDK Wiki content to Project KB or Personal KB.
- Do not register `sdk_wiki.*` as external MCP tools.
- Do not call LLMs inside `SdkWikiService` or index reader.
- Tool visibility should depend on ready installed SDK Wikis or should degrade to empty list/no_match safely.

### File Structure Requirements

Expected implementation files:

- `crewagent-runtime/shared/sdkWikiTypes.ts` extend query/result/error contracts
- `crewagent-runtime/electron/services/sdkWikiService.ts` add read-only query methods or delegate to a new reader
- `crewagent-runtime/electron/services/sdkWikiService.test.ts` add service-level query tests
- `crewagent-runtime/electron/services/toolHost.ts` add SDK Wiki ToolResult union types
- `crewagent-runtime/electron/services/fileSystemToolHost.ts` add tool schemas and dispatch
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` add tool visibility/dispatch tests
- `crewagent-runtime/electron/main.ts` wire FileSystemToolHost callback to `sdkWikiService`
- `crewagent-runtime/electron/services/promptComposer.ts` or adjacent context helper add compact registry guidance if design chooses PromptComposer route
- tests for prompt/context injection if compact registry is implemented there

Avoid modifying for 14.2:

- SDK import validation semantics except for bugs discovered while testing;
- Project KB retrieval internals;
- MCP execution/risk policy;
- `sdk_wiki.plan_api_usage` implementation.

### Testing Requirements

Required service tests:

- list installed SDKs returns compact metadata without absolute paths;
- search by exact API id/title returns bounded results;
- search by term from `index/terms.json` returns page refs;
- search by symbol alias from `index/symbols.json` returns page refs;
- read_page returns metadata/frontmatter/content for an indexed page;
- read_page rejects unknown page id;
- read_page rejects or fails safely if indexed path escapes root or file is missing;
- resolve_symbol exact/alias hit and no_match;
- expand_relations returns requires/related refs and missingTargets for unresolved targets;
- multi-version ambiguity requires explicit version.

Required ToolHost/main integration tests:

- `sdk_wiki.*` tools visible only when SDK Wiki registry has ready entries or design-chosen fallback behavior is verified;
- `sdk_wiki.search_pages/read_page/resolve_symbol/expand_relations` forward validated args to service callback;
- invalid args return structured tool errors;
- result payloads do not include absolute RuntimeStore paths.

Required prompt/context tests:

- compact registry appears when ready SDK Wiki is installed;
- full Markdown page content is not injected into initial context;
- registry message mentions using `sdk_wiki.search_pages/read_page/resolve_symbol/expand_relations` before making SDK API claims.

### Source Documents and Code Read

- `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/code-review-14-1-sdk-wiki-pack-import-and-registry.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/02-sdk-llm-wiki-module.md`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/main.ts`
- local sample SDK Wiki packs under `/Users/mengbin/personal/research/tool_wiki/sam-agent-workspace/sdk-wiki-packs/`

## Tasks / Subtasks

- [x] Task 1: SDK Wiki query/read contracts (AC: 2-6)
  - [x] 1.1 Define result types for list/search/read/resolve/expand
  - [x] 1.2 Define structured SDK Wiki query error codes
  - [x] 1.3 Extend ToolResult union with SDK Wiki result types

- [x] Task 2: Index reader and service query methods (AC: 2-6,7)
  - [x] 2.1 Load registered pack index files through RuntimeStore install root
  - [x] 2.2 Implement version selection and ambiguity handling
  - [x] 2.3 Implement bounded `list_sdks`
  - [x] 2.4 Implement deterministic `search_pages`
  - [x] 2.5 Implement root-contained `read_page`
  - [x] 2.6 Implement `resolve_symbol`
  - [x] 2.7 Implement `expand_relations`
  - [x] 2.8 Ensure all query methods return no absolute private paths

- [x] Task 3: ToolHost integration (AC: 1-7)
  - [x] 3.1 Add SDK Wiki tool schemas to `getVisibleTools`
  - [x] 3.2 Add `sdk_wiki.*` dispatch routes
  - [x] 3.3 Add FileSystemToolHost callback or service dependency
  - [x] 3.4 Wire callback in `main.ts`
  - [x] 3.5 Keep tools read-only and independent of Project KB/MCP

- [x] Task 4: Compact context guidance (AC: 1)
  - [x] 4.1 Add compact SDK Wiki registry message
  - [x] 4.2 Ensure full Markdown content is never injected at session start
  - [x] 4.3 Tell LLM to use SDK Wiki tools before making SDK API claims

- [x] Task 5: Tests and fixtures (AC: 1-7)
  - [x] 5.1 Extend compact SDK Wiki fixture for query/read/relations
  - [x] 5.2 Add service tests for list/search/read/resolve/expand
  - [x] 5.3 Add root containment and missing/corrupt index tests
  - [x] 5.4 Add multi-version ambiguity tests
  - [x] 5.5 Add ToolHost visibility/dispatch tests
  - [x] 5.6 Add prompt/context compact registry tests

## Dev Agent Record

### Agent Model Used

GPT-5

### Debug Log References

- `npm test -- electron/services/sdkWikiService.test.ts` - passed, 26 tests.
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "SDK Wiki"` - passed, 2 tests.
- `npm test -- electron/services/executionEngine.test.ts -t "injects skill registry"` - passed, 1 test.
- `npm exec tsc -- --noEmit` - passed.
- `./node_modules/.bin/eslint electron/services/sdkWikiService.ts electron/services/sdkWikiIndexReader.ts electron/services/fileSystemToolHost.ts electron/main.ts shared/sdkWikiTypes.ts electron/services/sdkWikiService.test.ts electron/services/fileSystemToolHost.test.ts electron/services/executionEngine.test.ts --ext ts,tsx --report-unused-disable-directives --max-warnings 0` - passed.
- `npm run build:ci` - passed; only existing npm/Vite warnings were emitted.
- Attempted combined `npm test -- electron/services/sdkWikiService.test.ts electron/services/fileSystemToolHost.test.ts electron/services/executionEngine.test.ts`; the broad `fileSystemToolHost.test.ts` run exposed long-running media/Python/Node timeout failures outside 14.2 scope, so final verification used the 14.2-scoped tests above plus TypeScript and ESLint.

### Completion Notes List

- Added read-only SDK Wiki query contracts and structured result/error types.
- Added `SdkWikiIndexReader` and service facade methods for list/search/read/resolve/expand, including bounded output, version ambiguity handling, source refs, missing targets, and no absolute private paths in results.
- Exposed `sdk_wiki.*` internal tools through `FileSystemToolHost`, wired main-process callback to `SdkWikiService`, and kept the tools independent of fs/project write policy.
- Injected compact SDK Wiki registry guidance into run/chat/agent context paths without loading Markdown page bodies.
- Added service, ToolHost, and prompt/context regression tests for the 14.2 acceptance criteria.
- Review follow-up fixed damaged-index path sanitization, malformed symbol/term/relation validation, and missing-target relation bounding.

### File List

- `_bmad-output/implementation-artifacts/14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
- `_bmad-output/implementation-artifacts/code-review-14-2-sdk-wiki-search-read-symbol-and-relation-module.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/shared/sdkWikiTypes.ts`
- `crewagent-runtime/electron/services/sdkWikiIndexReader.ts`
- `crewagent-runtime/electron/services/sdkWikiService.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`

### Change Log

- 2026-05-14 - Created story and design/tech spec; story moved to ready-for-dev.
- 2026-05-14 - Implemented 14.2 SDK Wiki read/search/symbol/relation module; story moved to review.
- 2026-05-14 - Code review completed; changes requested for damaged-index path sanitization, malformed-index validation, and bounded missing relation targets.
- 2026-05-14 - Review follow-up completed and approved; story moved to done.
