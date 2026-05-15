# PRD: Runtime SDK Wiki and Trusted SDK Tool Governance

> **Parent Document**: `_bmad-output/prd.md`
> **Source Requirements**: `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/`
> **Scope Boundary**: 本文覆盖 Runtime 对外部 SDK Wiki Pack 的导入、校验、注册、检索、API 使用规划，以及 SDK 工具调用的治理元数据与审计。本文不定义 SAM/Abaqus/Ansys/OpenFOAM 任何单一 SDK 的私有 API 细节，也不要求外部 MCP Server 或 Bridge 自行调用 LLM。执行安全由集成 MCP 软件负责；Runtime 不再基于 SDK risk metadata 做用户确认门禁。

---

## 1. Problem Statement

CrewAgent 正在从通用 BMAD Runtime 扩展到工程软件 SDK 集成场景。对 SAM、Abaqus、Ansys、OpenFOAM 等 SDK 来说，LLM 需要稳定理解 SDK API、概念、工作流和依赖关系，但这些知识不能只依赖模型预训练或临时上传文档。

当前缺口：

1. Runtime 缺少标准方式导入外部生成的 SDK Wiki Pack，并验证其 schema、索引和内容完整性；
2. Runtime 现有 Project KB 更适合项目资料检索，不适合作为 SDK API 符号、关系和执行规划的权威知识源；
3. Runtime 缺少 SDK-aware 内部知识模块，无法让主 LLM 按页、按符号、按关系读取 SDK Wiki，并要求计划引用真实来源；
4. 工程 SDK 工具调用可能修改模型、写文件、启动求解或执行破坏性操作，需要统一风险级别和审计日志，但执行安全由暴露该工具的 MCP/集成软件负责；
5. SDK 知识理解必须发生在 CrewAgent Runtime 内部，不能下放给 SAM MCP Server、SAM Bridge 或其他外部工具服务。

本 PRD 的目标是把“SDK Wiki 知识接入”和“可信 MCP 边界下的 SDK 工具治理审计”定义为 Runtime 的通用能力，而不是 SAM 专用功能。

---

## 2. Goals / Non-Goals

### Goals

- 支持导入目录或压缩包形式的外部 SDK Wiki Pack；
- 校验 `sdk-wiki.json`、`index/manifest.json`、schema/version/entry、索引文件、hash 和页面 frontmatter；
- 将 SDK Wiki Pack 注册到 RuntimeStore，并允许 Runtime 列出已安装 SDK Wiki；
- 提供 Runtime 内部 `sdk_wiki.*` 知识能力，用于检索、读取、符号解析、关系扩展和 API 使用规划；
- 支持 API / workflow / concept / relation 等页面类型；
- 要求 SDK API 建议和执行计划带来源引用；
- 当相关 Wiki 页面缺失时返回缺失知识，而不是编造 API；
- 明确 LLM 边界：所有 SDK 理解和规划都由 CrewAgent Runtime 主 LLM 完成；
- 为 SDK 工具调用定义通用风险级别、治理元数据和审计日志；
- 以 SAM 为第一条 golden path，但保持对 Abaqus、Ansys、OpenFOAM 等未来 SDK 的通用性。

### Non-Goals

- 不在本 PRD 中定义 SDK Wiki Pack 的生成器或爬取器；
- 不要求 Runtime 在导入失败时自动重建索引；
- 不把 SDK Wiki 内容混入 Project KB 或 Personal KB 的存储模型；
- 不把 `sdk_wiki.*` 暴露为外部 MCP Server 能力；
- 不让 SAM MCP Server、SAM Bridge 或任何外部 SDK Bridge 调用 LLM 做 SDK 理解；
- 不在 MVP 中实现跨 SDK 复杂冲突合并或多版本自动升级策略；
- 不在 Runtime 中重新实现 MCP/集成软件已经负责的执行安全、路径安全、求解安全、license/资源安全或 destructive 操作拦截。

---

## 3. User Roles

- **Runtime User**
  - 导入某个 SDK Wiki Pack，希望在 Chat/Agent/Run 中让模型可靠使用该 SDK。
- **SDK Pack Author**
  - 生成符合约定结构的 SDK Wiki Pack，并希望 Runtime 能校验和消费。
- **Runtime Developer**
  - 实现 SDK Wiki 导入、注册、内部知识能力、风险确认和审计。
- **SDK Bridge / MCP Developer**
  - 提供真正执行 SDK 操作的工具，但不负责 SDK 语义理解和 LLM 规划。

---

## 4. Functional Requirements

### SDK Wiki Import and Registry

- **FR-SDKW-01**: Runtime must import an SDK Wiki Pack from either a directory or an archive.
- **FR-SDKW-02**: Imported SDK Wiki Pack must contain `sdk-wiki.json`, `raw/`, `wiki/`, and `index/`.
- **FR-SDKW-03**: Runtime must read and validate `sdk-wiki.json` and `index/manifest.json` before registration.
- **FR-SDKW-04**: Runtime must validate `schemaVersion`, `sdkId`, `version`, and `entry` for compatibility.
- **FR-SDKW-05**: Runtime must validate required index files exist and refuse incompatible index schema instead of silently rebuilding it.
- **FR-SDKW-06**: Runtime must validate content hash and index hash when the pack provides them.
- **FR-SDKW-07**: Runtime must parse page frontmatter during import validation and reject invalid pages with structured errors.
- **FR-SDKW-08**: Runtime must register valid SDK Wiki Packs under RuntimeStore at `sdk-wikis/<sdkId>/<version>/`.
- **FR-SDKW-09**: Runtime must list installed SDK Wikis with `sdkId`, `version`, display metadata, status, and install location alias.
- **FR-SDKW-10**: Runtime must expose imported SDK Wikis to the SDK LLM Wiki Module without requiring Project KB import.

### SDK Wiki Pack Management UI

- **FR-SDKW-10A**: Runtime must provide a visible Settings entry to list installed SDK Wiki Packs and import a pack from either directory or archive.
- **FR-SDKW-10B**: Runtime must allow users to remove an installed SDK Wiki Pack by `sdkId` and `version`, with confirmation, structured failure reporting, and registry consistency.

### SDK LLM Wiki Module

- **FR-SDKW-11**: Runtime must provide internal SDK Wiki abilities equivalent to:
  - `sdk_wiki.list_sdks`
  - `sdk_wiki.search_pages`
  - `sdk_wiki.read_page`
  - `sdk_wiki.resolve_symbol`
  - `sdk_wiki.expand_relations`
  - `sdk_wiki.plan_api_usage`
- **FR-SDKW-12**: These abilities are Runtime internal function-call tools or context functions; they are not external MCP tools.
- **FR-SDKW-13**: SDK Wiki page discovery must use imported index files as the authoritative source.
- **FR-SDKW-14**: Runtime must read structured Markdown pages on demand and avoid preloading an entire SDK Wiki into prompt context.
- **FR-SDKW-15**: The module must support API, workflow, concept, and relation page types.
- **FR-SDKW-16**: API recommendations and execution plans must include source references to existing Wiki pages.
- **FR-SDKW-17**: `sdk_wiki.plan_api_usage` must require plans to reference existing Wiki pages.
- **FR-SDKW-18**: If required pages or symbols are missing, the module must return `missingInformation` or equivalent structured gaps instead of inventing API names.
- **FR-SDKW-19**: The module must support a selected `sdkId` and version, while allowing future project/package defaults.

### LLM Boundary

- **FR-SDKW-20**: All LLM reasoning for SDK understanding must happen inside CrewAgent Runtime.
- **FR-SDKW-21**: SDK Bridge, SAM MCP Server, and future SDK execution servers must not call LLMs for SDK understanding.
- **FR-SDKW-22**: External SDK tools may return deterministic tool results, errors, and tracebacks, but semantic planning belongs to Runtime.

### Trusted SDK Tool Governance and Logging

- **FR-SDKW-23**: Runtime must accept SDK tool governance metadata with risk levels: `read`, `model_write`, `file_write`, `solve`, `destructive`.
- **FR-SDKW-24**: Runtime must treat MCP-exposed SDK execution tools as trusted to execute when the existing effective Runtime tool policy allows the underlying tool.
- **FR-SDKW-25**: Runtime must not require user confirmation solely because an SDK tool is classified as `file_write`, `solve`, or `destructive`.
- **FR-SDKW-26**: Runtime should log `purpose`, `targetPath`, and `targetObject` when adapters provide them, and log metadata warnings when recommended fields are missing.
- **FR-SDKW-27**: Runtime must preserve the existing effective tool policy boundary: SDK governance metadata cannot enable a disabled MCP/local tool and should not block an enabled one.
- **FR-SDKW-28**: MCP/integration software owns execution safety, including path scope, destructive operation policy, solve resource/license guards, and any domain-specific confirmation behavior.
- **FR-SDKW-29**: Runtime must record SDK tool call logs with tool name, purpose, risk, governance state or warnings, execution status, duration, error code, and traceback summary when failed.
- **FR-SDKW-30**: SDK metadata resolver failures or invalid metadata must be visible in logs without preventing execution by themselves.

---

## 5. Non-Functional Requirements

- **NFR-SDKW-01 Integrity**: Import must be transactional. A failed validation must not leave a partially registered SDK Wiki.
- **NFR-SDKW-02 Determinism**: SDK page discovery, symbol resolution, and relation expansion must be based on imported indexes, not fuzzy filesystem scans after registration.
- **NFR-SDKW-03 Context Efficiency**: Runtime must retrieve only the relevant SDK Wiki pages or snippets needed for the current turn.
- **NFR-SDKW-04 Traceability**: Every API recommendation used in a plan must be traceable to a Wiki page and page type.
- **NFR-SDKW-05 Trust Boundary**: SDK tool execution must obey existing Runtime effective tool policy. Domain execution safety is delegated to MCP/integration software, while Runtime provides tool visibility control and auditability.
- **NFR-SDKW-06 Offline Operation**: Once an SDK Wiki Pack is imported, SDK Wiki search/read/plan support must work offline, assuming the configured main LLM is available.
- **NFR-SDKW-07 Extensibility**: The data model must not hardcode SAM-only concepts; SAM may be a fixture and first integration.
- **NFR-SDKW-08 Graceful Failure**: Invalid schema, missing index files, hash mismatch, invalid page frontmatter, unsupported versions, missing symbols, and SDK tool failures must return structured errors.

---

## 6. Error Codes

Runtime must preserve the source requirement error vocabulary and may extend it with implementation-specific diagnostics:

- `SDK_WIKI_SCHEMA_INVALID`
- `SDK_WIKI_INDEX_INVALID`
- `SDK_WIKI_HASH_MISMATCH`
- `SDK_WIKI_PAGE_INVALID`
- `SDK_WIKI_VERSION_UNSUPPORTED`
- `SDK_WIKI_NOT_INSTALLED`
- `SDK_WIKI_SYMBOL_NOT_FOUND`
- `SDK_TOOL_GOVERNANCE_METADATA_INVALID`
- `SDK_TOOL_GOVERNANCE_RESOLVER_FAILED`

---

## 7. Acceptance Summary

The capability is acceptable when:

1. Importing `sdk-wiki-packs/sam` registers SDK id `sam`;
2. Runtime can list installed SDK Wikis;
3. Settings provides visible SDK Wiki Pack import/list/remove controls;
4. Removing an installed SDK Wiki Pack updates registry and local installed assets consistently;
5. Runtime refuses a pack with missing `index/manifest.json`;
6. Runtime refuses incompatible index schema instead of silently rebuilding it;
7. Given a query such as `施加压力载荷`, the SDK Wiki Module can surface relevant Pressure, Surface, and StaticStep pages from the SAM SDK Wiki;
8. `sdk_wiki.plan_api_usage` returns a plan with source references when enough knowledge exists;
9. If relevant Wiki pages are missing, the module reports missing knowledge instead of inventing API names;
10. Submitting a solve job through an MCP-exposed SDK tool does not trigger a Runtime SDK confirmation gate solely because its risk is `solve`;
11. SDK governance audit events include requested/executed/failed states and metadata warnings when applicable;
12. SDK tracebacks are retained in summarized form for debugging.

---

## 8. Compatibility and Relationship to Existing Runtime Capabilities

- **Project KB / Personal KB**: SDK Wiki is a distinct, SDK-scoped knowledge substrate. Project KB stores project references and extracted materials; SDK Wiki stores versioned SDK API/reference knowledge.
- **Claude Code Skills**: Skills can guide how to use workflows and supporting scripts. SDK Wiki provides authoritative SDK API knowledge and should be callable from the same main LLM loop.
- **Tool Policy**: Existing effective tool policy remains the enforcement layer for tool visibility and availability. SDK governance metadata cannot grant tools that system or agent policy disabled.
- **Widgets**: Runtime SDK governance does not introduce a user confirmation widget. Domain-specific confirmations, if needed, belong inside MCP/integration tools.
- **MCP Tools**: External SDK execution may arrive through MCP or built-in adapters, but `sdk_wiki.*` remains Runtime internal and SDK semantic reasoning stays inside Runtime.

---

## 9. Source Requirement Archive

The original generic requirement documents have been copied into:

```text
_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/
  README.md
  01-sdk-wiki-importer.md
  02-sdk-llm-wiki-module.md
  03-tool-risk-confirmation-and-logging.md
```

These files are treated as the imported planning source for this PRD.
