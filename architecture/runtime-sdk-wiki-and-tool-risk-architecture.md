# Runtime SDK Wiki and Trusted SDK Tool Governance Architecture

> Parent Architecture: `_bmad-output/architecture.md`
> Parent Runtime Architecture: `_bmad-output/architecture/runtime-architecture.md`
> Sub-PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`

---

## 1. Context and Constraints

本架构增量定义 Runtime 如何消费外部 SDK Wiki Pack，并在执行 SDK 工具时提供可信 MCP 边界下的治理 metadata 与审计。目标是让主 LLM 能基于已导入 SDK Wiki 做 API 选择、依赖推理和小步执行规划，同时让真正会修改工程模型或启动求解的工具调用由 MCP/集成软件负责执行安全，Runtime 负责工具可见性和可追溯日志。

已知约束：

1. Runtime 已有 Project KB / Personal KB，但 SDK Wiki 是 SDK 版本级知识源，不能混入项目知识库；
2. Runtime 已有 `fs.*`、`ui.ask_user`、`terminal.run`、`python.run`、`node.run`、MCP、tool policy、widget、execution log 等基础能力；
3. `sdk_wiki.*` 是 Runtime 内部能力，不是外部 MCP Server；
4. SAM 只是第一条 golden path，架构必须保持 SDK-generic；
5. SAM MCP Server / SAM Bridge / future SDK Bridge 不得自行调用 LLM 做 SDK 理解；
6. SDK 工具调用的领域执行安全由 MCP/集成软件保证。Runtime 不再基于 SDK risk metadata 做用户确认门禁；Runtime 只保留 existing effective tool policy、metadata normalization 和 audit。

---

## 2. Core Decisions

### AD-SDKW-01: SDK Wiki is a first-class RuntimeStore domain

- **Decision**: SDK Wiki Pack 导入后存储在 RuntimeStore 独立命名空间 `sdk-wikis/<sdkId>/<version>/`。
- **Rationale**: SDK Wiki 是 SDK 版本级知识，不属于单个 project，也不应污染用户 ProjectRoot。

### AD-SDKW-02: Import validates, never silently rebuilds

- **Decision**: Runtime 导入时只校验外部 pack 的 schema、manifest、hash、index 和页面 frontmatter；MVP 不在导入失败时自动重建索引。
- **Rationale**: SDK Wiki Pack 是外部构建产物。静默重建会掩盖生成器问题，并导致 Runtime 与 pack 作者认知不一致。

### AD-SDKW-03: `sdk_wiki.*` is internal to Runtime

- **Decision**: `sdk_wiki.list_sdks/search_pages/read_page/resolve_symbol/expand_relations/plan_api_usage` 由 Runtime 内部 ToolHost 或 context function 提供，不注册为外部 MCP tools。
- **Rationale**: SDK 知识读取需要遵守 Runtime 上下文预算、来源引用和 LLM 边界；外部 MCP Server 只负责执行确定性 SDK 操作。

### AD-SDKW-04: Main LLM owns SDK reasoning

- **Decision**: 所有 SDK API 选择、依赖推理、小步执行规划都发生在 CrewAgent Runtime 主 LLM loop 内。
- **Rationale**: 避免 SAM MCP Server / Bridge 内部再引入不可观察的 LLM 决策链，保持 prompt、日志、确认和审计都在 Runtime 侧可追踪。

### AD-SDKW-05: Trusted MCP SDK tools are observed, not re-confirmed

- **Decision**: SDK execution tools 在 ToolHost 执行前可提供 `SdkToolRiskEnvelope`。Runtime 只规范化 risk metadata、记录缺失/异常字段 warning，并把 requested/executed/failed 写入 audit；只要 existing effective tool policy 允许底层 MCP/local tool，SDK risk metadata 不触发 Runtime 用户确认或拒绝。
- **Rationale**: MCP/集成软件是执行安全边界。只要 MCP 暴露某个 tool，就表示该工具已经在集成层完成安全约束。Runtime 再做确认门禁会破坏 Agent 自主执行闭环，并把领域安全责任放错层级。

---

## 3. Storage Layout

### 3.1 RuntimeStore SDK Wiki Layout

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
```

`registry.json` records installed packs:

```json
{
  "schemaVersion": "1.0",
  "installed": [
    {
      "sdkId": "sam",
      "version": "1.0.0",
      "displayName": "SAM SDK",
      "schemaVersion": "1.0",
      "entry": "wiki/index.md",
      "installedAt": "2026-05-13T00:00:00+08:00",
      "status": "ready",
      "rootAlias": "@sdk_wiki/sam/1.0.0"
    }
  ]
}
```

`install.json` stores Runtime-side registration metadata and validation summary. It does not replace `sdk-wiki.json`.

### 3.2 Import Transaction Layout

Imports use a temp directory before commit:

```text
<RuntimeStoreRoot>/
  tmp/
    sdk-wiki-import-<opId>/
      extracted/
      validation-report.json
```

Commit rule:

1. Extract or copy to temp;
2. Validate all required metadata, hashes, indexes, and pages;
3. Move temp `extracted/` into `sdk-wikis/<sdkId>/<version>/`;
4. Update `registry.json` atomically;
5. Append `import-log.ndjson`.

If any validation fails, Runtime removes temp and leaves registry unchanged.

---

## 4. Component Model

### 4.1 `SdkWikiImportService`

Responsibilities:

- Accept directory or archive import input;
- Safely extract archives without path traversal;
- Read `sdk-wiki.json`;
- Read `index/manifest.json`;
- Validate schema compatibility, `sdkId`, `version`, and `entry`;
- Validate required directories and index files;
- Validate content/index hashes when present;
- Parse Wiki page frontmatter;
- Write validation report and structured errors;
- Commit valid packs into RuntimeStore transactionally.

Structured errors include:

- `SDK_WIKI_SCHEMA_INVALID`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_HASH_MISMATCH`
- `SDK_WIKI_PAGE_INVALID`
- `SDK_WIKI_VERSION_UNSUPPORTED`

### 4.2 `SdkWikiRegistryService`

Responsibilities:

- Read and maintain `sdk-wikis/registry.json`;
- List installed SDK Wikis;
- Remove installed SDK Wikis by `(sdkId, version)` while keeping registry and installed files consistent;
- Resolve `(sdkId, version?)` to a RuntimeStore root;
- Provide stable aliases such as `@sdk_wiki/<sdkId>/<version>/...`;
- Prevent duplicate `(sdkId, version)` overwrite unless the caller explicitly chooses reinstall semantics.

### 4.3 `SdkWikiIndexReader`

Responsibilities:

- Load `index/manifest.json`;
- Load page, symbol, and relation indexes defined by the manifest;
- Resolve page ids to Markdown paths under `wiki/`;
- Resolve symbols to API pages;
- Expand relation edges across API / workflow / concept / relation pages;
- Return source metadata for every result.

The reader uses index files as authority. It does not discover arbitrary pages by recursively scanning `wiki/` after registration.

### 4.4 `SdkWikiModule`

Runtime internal module exposed to the main LLM loop as internal tools or equivalent context functions:

```text
sdk_wiki.list_sdks
sdk_wiki.search_pages
sdk_wiki.read_page
sdk_wiki.resolve_symbol
sdk_wiki.expand_relations
sdk_wiki.plan_api_usage
```

Return values must be compact and source-referenced. Large Markdown pages should be returned with page metadata and bounded content slices unless the caller explicitly requests a full page within policy limits.

### 4.5 `SdkApiUsagePlanner`

Responsibilities:

- Accept intent, selected SDK, optional model state, and constraints;
- Use `SdkWikiIndexReader` to retrieve candidate pages and related symbols;
- Build a structured planning payload for the main LLM;
- Require each proposed API or step to reference known SDK Wiki pages;
- Return `missingInformation` when required concepts, pages, or symbols are absent.

Important boundary: this planner does not call an external LLM. It either returns deterministic retrieval/planning scaffolding or participates in the existing Runtime main LLM function-call loop.

### 4.6 `SdkToolGovernanceService`

Responsibilities:

- Normalize SDK tool risk metadata;
- Build stable argument fingerprints for audit correlation;
- Record metadata warnings, for example missing `purpose` on `model_write`, missing `targetPath` on `file_write`, invalid risk, or resolver failure;
- Always return an allow/observe decision because SDK risk metadata is not an execution blocker;
- Preserve the existing effective tool policy boundary by never enabling a disabled underlying tool.

### 4.7 `SdkToolAuditService`

Responsibilities:

- Append SDK tool audit events into the existing execution log path for the active project/run/conversation;
- Preserve SDK traceback summaries for debugging;
- Record requested, executed, failed, and metadata warning states;
- Keep logs source-linked to tool call id, sdk id, risk, purpose, governance state, and argument fingerprint.

---

## 5. Data Contracts

### 5.1 SDK Wiki Pack Contract

Minimum expected pack:

```text
sdk-wiki.json
raw/
wiki/
index/
  manifest.json
```

Runtime expects `sdk-wiki.json` and `index/manifest.json` to provide enough information to validate:

- pack schema version;
- SDK id and SDK version;
- entry page;
- required index files;
- optional content and index hashes.

### 5.2 Internal Models

```ts
interface InstalledSdkWiki {
  sdkId: string
  version: string
  displayName?: string
  schemaVersion: string
  entry: string
  rootAlias: string
  installedAt: string
  status: 'ready' | 'invalid'
}

interface SdkWikiPageRef {
  sdkId: string
  version: string
  pageId: string
  pageType: 'api' | 'workflow' | 'concept' | 'relation'
  title: string
  path: string
  hash?: string
}

interface SdkApiUsagePlan {
  ok: boolean
  sdkId: string
  version: string
  taskType?: string
  requiredApis: Array<{
    symbol: string
    pageRef: SdkWikiPageRef
    reason: string
  }>
  executionPlan: Array<{
    step: string
    sourceRefs: SdkWikiPageRef[]
    riskHint?: 'read' | 'model_write' | 'file_write' | 'solve' | 'destructive'
  }>
  missingInformation: string[]
  confidence: number
}
```

### 5.3 SDK Tool Governance Metadata

SDK execution tools may provide or be wrapped with governance metadata:

```ts
type SdkToolRisk = 'read' | 'model_write' | 'file_write' | 'solve' | 'destructive'

interface SdkToolRiskEnvelope {
  sdkId: string
  toolName: string
  risk: SdkToolRisk
  purpose?: string
  targetPath?: string
  targetObject?: string
}
```

If an external MCP tool cannot declare risk metadata directly, Runtime may map it through a local adapter registry for observability. Missing metadata is a logging gap, not a Runtime execution blocker.

---

## 6. Runtime Flows

### 6.1 Import Flow

1. User chooses SDK Wiki Pack directory/archive;
2. `SdkWikiImportService` copies/extracts into temp;
3. Service validates schema, manifest, hash, required files, and pages;
4. On success, service commits to `RuntimeStore/sdk-wikis/<sdkId>/<version>/`;
5. `SdkWikiRegistryService` updates `registry.json`;
6. Runtime emits `sdk_wiki.imported` audit event and exposes the SDK through `sdk_wiki.list_sdks`.

Failure response includes error code, human-readable message, and validation detail. Runtime does not auto-rebuild indexes.

### 6.1a Management UI and Remove Flow

1. User opens Settings -> SDK Wiki Packs;
2. Runtime lists installed packs using registry metadata only, without exposing absolute RuntimeStore paths;
3. User imports a directory/archive through explicit actions, then the list refreshes from registry;
4. User removes a pack by `sdkId@version` after confirmation;
5. Service deletes the installed pack directory and updates `registry.json` transactionally;
6. Remove failure returns a structured error and must not leave registry pointing to missing pack files.

### 6.2 SDK Wiki Query Flow

1. Main LLM sees compact SDK Wiki availability in Runtime context or calls `sdk_wiki.list_sdks`;
2. Main LLM calls `sdk_wiki.search_pages` or `sdk_wiki.resolve_symbol`;
3. `SdkWikiModule` uses `SdkWikiIndexReader`;
4. Tool result returns page refs, snippets, scores, and page types;
5. Main LLM calls `sdk_wiki.read_page` or `sdk_wiki.expand_relations` as needed;
6. Final answer or plan includes source refs.

### 6.3 API Usage Planning Flow

1. User asks an SDK-specific task, such as `给当前板架模型施加向下均布压力`;
2. Main LLM calls `sdk_wiki.plan_api_usage` with `sdkId`, intent, model state, and constraints;
3. Runtime retrieves candidate API/workflow/concept/relation pages;
4. Runtime returns a structured plan scaffold with source refs and missing information;
5. Main LLM converts the scaffold into user-facing plan or tool calls;
6. Any execution tool call may be observed by `SdkToolGovernanceService` for audit, while MCP/integration software remains the execution safety boundary.

### 6.4 Trusted SDK Tool Governance Flow

1. Main LLM requests an SDK execution tool;
2. ToolHost resolves existing effective tool policy first; disabled MCP/local tools remain disabled;
3. SDK tool wrapper optionally supplies `SdkToolRiskEnvelope`;
4. `SdkToolGovernanceService` normalizes metadata and creates warnings when recommended metadata is missing or invalid;
5. ToolHost dispatches the underlying tool without Runtime SDK confirmation when effective tool policy allows it;
6. `SdkToolAuditService` logs requested/metadata-warning, executed, and failed states.

---

## 7. Prompt and Tool Surface

### 7.1 Prompt Injection

Runtime should inject only a compact SDK Wiki registry:

- installed SDK ids and versions;
- short display names;
- entry page title if available;
- reminder that SDK API claims must use `sdk_wiki.*` and source refs.

Runtime must not inject full SDK Wiki pages at session start.

### 7.2 Internal Tool Visibility

`sdk_wiki.*` visibility should be gated by:

- at least one installed SDK Wiki;
- current conversation mode allowing knowledge tools;
- effective tool policy not disabling internal knowledge tools if such policy exists later.

SDK execution tools remain separate from SDK Wiki tools. Their availability is governed by existing effective tool policy and MCP exposure, while SDK governance metadata adds observability only.

---

## 8. Observability

Recommended event names:

- `sdk_wiki.import_started`
- `sdk_wiki.import_failed`
- `sdk_wiki.imported`
- `sdk_wiki.search`
- `sdk_wiki.page_read`
- `sdk_wiki.symbol_resolved`
- `sdk_wiki.relations_expanded`
- `sdk_wiki.plan_generated`
- `sdk_tool.requested`
- `sdk_tool.metadata_warning`
- `sdk_tool.executed`
- `sdk_tool.failed`

Each SDK tool execution audit record includes:

- `timestamp`
- `projectId` / `conversationId` / `runId` when available
- `sdkId`
- `toolName`
- `purpose`
- `risk`
- `governanceState`
- `warnings`
- `status`
- `durationMs`
- `errorCode`
- `tracebackSummary`

---

## 9. Security Boundaries

- Archive extraction must reject absolute paths and `..` traversal;
- Imported SDK Wiki files live under RuntimeStore and are read-only after registration unless reinstall is explicit;
- SDK Wiki page reads must stay under the registered pack root;
- SDK execution tools must obey existing effective tool policy before dispatch;
- SDK governance metadata cannot turn a disabled tool back on;
- Runtime does not issue SDK confirmation tokens;
- Destructive/solve/path safety belongs to MCP/integration software before or during tool execution.

---

## 10. Implementation Notes for First Slice

Recommended first implementation sequence:

1. `SdkWikiImportService` + RuntimeStore layout + registry listing;
2. `SdkWikiIndexReader` + `sdk_wiki.search_pages/read_page/resolve_symbol`;
3. `sdk_wiki.expand_relations/plan_api_usage` with source refs and missing information;
4. SDK governance metadata wrapper + audit logging;
5. SAM golden path validation fixture.

This order lets Runtime validate knowledge import before coupling it to real SDK execution risk.
