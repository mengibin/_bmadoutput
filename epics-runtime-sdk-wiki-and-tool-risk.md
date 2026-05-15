---
stepsCompleted: [1, 2, 3, 9]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/README.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/01-sdk-wiki-importer.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/02-sdk-llm-wiki-module.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/03-tool-risk-confirmation-and-logging.md'
workflowType: 'epics'
lastStep: 9
project_name: 'CrewAgent Runtime SDK Wiki and Tool Risk'
user_name: 'Mengbin'
date: '2026-05-13'
---

# CrewAgent Runtime SDK Wiki and Trusted SDK Tool Governance - Epic Breakdown

## Overview

本文件将 Runtime SDK Wiki 与 SDK 工具风险治理需求拆解为 **Epic 14 + 六条 runtime vertical slice Story**，覆盖：

- SDK Wiki Pack 导入、校验和 RuntimeStore 注册；
- SDK Wiki Pack 可见管理入口、导入状态展示和删除；
- SDK Wiki 内部检索、页面读取、符号解析和关系扩展；
- SDK-aware API 使用规划与来源引用；
- SDK 工具风险级别、可信 MCP 边界和审计日志；
- SAM golden path 验证，同时保持 Abaqus、Ansys、OpenFOAM 等 SDK 可复用。

来源文档：

- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/`

---

## Requirements Inventory

### Functional Requirements

- FR-SDKW-01: Runtime 支持从目录或压缩包导入 SDK Wiki Pack。
- FR-SDKW-02: SDK Wiki Pack 必须包含 `sdk-wiki.json`、`raw/`、`wiki/`、`index/`。
- FR-SDKW-03: Runtime 在注册前读取并校验 `sdk-wiki.json` 与 `index/manifest.json`。
- FR-SDKW-04: Runtime 校验 `schemaVersion`、`sdkId`、`version`、`entry`。
- FR-SDKW-05: Runtime 校验 required index files，并拒绝不兼容 schema，不静默重建索引。
- FR-SDKW-06: Runtime 在 pack 提供 hash 时校验 content/index hash。
- FR-SDKW-07: Runtime 导入时解析并校验页面 frontmatter。
- FR-SDKW-08: Runtime 将有效 SDK Wiki Pack 注册到 `RuntimeStore/sdk-wikis/<sdkId>/<version>/`。
- FR-SDKW-09: Runtime 可列出已安装 SDK Wiki。
- FR-SDKW-10: Runtime 将已导入 SDK Wiki 暴露给 SDK LLM Wiki Module。
- FR-SDKW-10A: Runtime 在 Settings 提供 SDK Wiki Pack 的可见导入和列表入口。
- FR-SDKW-10B: Runtime 支持按 `sdkId@version` 删除已安装 SDK Wiki Pack，并保持 registry 与本地文件一致。
- FR-SDKW-11: Runtime 提供 `sdk_wiki.list_sdks/search_pages/read_page/resolve_symbol/expand_relations/plan_api_usage` 内部能力。
- FR-SDKW-12: `sdk_wiki.*` 是 Runtime 内部工具或上下文函数，不是外部 MCP 工具。
- FR-SDKW-13: SDK Wiki 页面发现必须使用导入索引作为权威来源。
- FR-SDKW-14: Runtime 按需读取结构化 Markdown 页面，不在首轮全量注入 SDK Wiki。
- FR-SDKW-15: Module 支持 API、workflow、concept、relation 页面类型。
- FR-SDKW-16: API 推荐与执行计划必须包含来源引用。
- FR-SDKW-17: `sdk_wiki.plan_api_usage` 要求计划引用已存在 Wiki 页面。
- FR-SDKW-18: 页面或符号缺失时，Module 返回 missing knowledge，不编造 API。
- FR-SDKW-19: Module 支持选定 `sdkId` 与版本。
- FR-SDKW-20: 所有 SDK 理解相关 LLM 推理必须发生在 CrewAgent Runtime 内部。
- FR-SDKW-21: SDK Bridge / SAM MCP Server / future SDK execution server 不得调用 LLM 做 SDK 理解。
- FR-SDKW-22: 外部 SDK 工具只返回确定性工具结果、错误和 traceback。
- FR-SDKW-23: Runtime 接收 SDK 工具治理 metadata，risk 为 `read/model_write/file_write/solve/destructive`。
- FR-SDKW-24: MCP 暴露的 SDK execution tool 在 existing effective Runtime tool policy 允许时应自主执行。
- FR-SDKW-25: Runtime 不因 `file_write/solve/destructive` risk metadata 触发用户确认。
- FR-SDKW-26: Runtime 记录 purpose、targetPath、targetObject，并对缺失推荐字段记录 metadata warning。
- FR-SDKW-27: Runtime 保留 existing effective tool policy 作为工具可见性/可用性边界。
- FR-SDKW-28: MCP/集成软件负责路径范围、破坏性操作、求解资源/license、领域确认等执行安全。
- FR-SDKW-29: Runtime 记录 SDK tool call 日志，包含 tool、purpose、risk、governance state/warnings、status、duration、error、traceback summary。
- FR-SDKW-30: metadata resolver failure 或 invalid metadata 必须可在日志中看到，但不单独阻断执行。

### Non-Functional Requirements

- NFR-SDKW-01: SDK Wiki import 必须事务化。
- NFR-SDKW-02: 页面发现、符号解析、关系扩展必须基于导入索引。
- NFR-SDKW-03: SDK Wiki 上下文注入必须按需、预算可控。
- NFR-SDKW-04: API 推荐和计划步骤必须可追溯到 Wiki 页面。
- NFR-SDKW-05: SDK 工具执行必须遵守 Runtime effective tool policy；领域执行安全由 MCP/集成软件负责。
- NFR-SDKW-06: SDK Wiki 导入后可离线检索和读取。
- NFR-SDKW-07: 数据模型不得硬编码 SAM-only 概念。
- NFR-SDKW-08: 所有导入、索引、页面、符号、工具失败必须结构化返回。

---

## FR Coverage Map

| FR/NFR | Epic / Story |
|:---|:---|
| FR-SDKW-01 ~ FR-SDKW-10, NFR-SDKW-01, NFR-SDKW-06, NFR-SDKW-08 | Story 14.1 |
| FR-SDKW-11 ~ FR-SDKW-16, FR-SDKW-19, NFR-SDKW-02 ~ NFR-SDKW-04 | Story 14.2 |
| FR-SDKW-10A ~ FR-SDKW-10B, SDK Wiki Pack management UX, remove lifecycle, validation report display | Story 14.3 |
| FR-SDKW-17 ~ FR-SDKW-22, NFR-SDKW-03, NFR-SDKW-04, NFR-SDKW-07 | Story 14.4 |
| FR-SDKW-23 ~ FR-SDKW-30, NFR-SDKW-05, NFR-SDKW-08 | Story 14.5 |
| Acceptance golden path + cross-SDK adapter contract | Story 14.6 |

---

## Epic List

### Epic 14: Runtime SDK Wiki and Trusted SDK Tool Governance

**Goal**: Let Runtime import versioned SDK Wiki Packs, expose SDK-aware internal knowledge abilities to the main LLM, and observe trusted MCP SDK execution tools with governance metadata and audit.

**FRs covered**: FR-SDKW-01~30, NFR-SDKW-01~08

**Deliverables**:

- RuntimeStore SDK Wiki registry and transactional import;
- Settings UI for SDK Wiki Pack list/import/remove;
- SDK Wiki index reader and internal `sdk_wiki.*` abilities;
- Source-referenced `sdk_wiki.plan_api_usage`;
- trusted MCP SDK tool governance metadata boundary;
- SDK tool audit event pipeline;
- SAM golden path validation fixture and generic SDK adapter contract.

---

## Recommended Implementation Order

建议严格按以下顺序实施：

1. **14.1 SDK Wiki Pack Import and Registry**
   先建立可信存储、事务导入和可列出状态。
2. **14.2 SDK Wiki Search, Read, Symbol, and Relation Module**
   在已注册 SDK Wiki 上建立内部知识读取闭环。
3. **14.3 SDK Wiki Pack Management UI and Remove**
   在 Settings 提供 SDK Wiki Pack 的导入、列表、错误展示和删除入口。
4. **14.4 SDK API Usage Planning with Source References**
   再把检索结果组织为可被主 LLM 使用的小步计划。
5. **14.5 Trusted SDK Tool Governance and Audit**
   移除 Runtime SDK 用户确认门禁，把 SDK 风险 metadata 收敛为 trusted MCP 边界下的审计与可观测能力。
6. **14.6 SAM Golden Path and Generic SDK Adapter Contract**
   最后用 SAM 验证端到端路径，并沉淀跨 SDK 接入契约。

依赖关系：

- 14.2 依赖 14.1；
- 14.3 依赖 14.1/14.2 的 registry/query contract；
- 14.4 依赖 14.2，并应在 14.3 的管理语义稳定后进入开发；
- 14.5 可与 14.4 部分并行设计，但真实端到端验证依赖 14.4；
- 14.6 依赖 14.1~14.5。

## BMAD Execution Status

本 Epic 采用严格串行 BMAD 执行规则：

```text
14.1 -> 14.2 -> 14.3 -> 14.4 -> 14.5 -> 14.6
```

当前状态（2026-05-15）：

| Order | Story | Status | Blocking Dependencies |
|:---|:---|:---|:---|
| 1 | 14.1 SDK Wiki Pack Import and Registry | `done` | Runtime codebase assessment complete |
| 2 | 14.2 SDK Wiki Search, Read, Symbol, and Relation Module | `done` | 14.1 |
| 3 | 14.3 SDK Wiki Pack Management UI and Remove | `done` | 14.1, 14.2 |
| 4 | 14.4 SDK API Usage Planning with Source References | `done` | 14.2, 14.3 |
| 5 | 14.5 Trusted SDK Tool Governance and Audit | `done` | 14.2, 14.4 planning/audit contract |
| 6 | 14.6 SAM Golden Path and Generic SDK Adapter Contract | `done` | 14.1, 14.2, 14.3, 14.4, 14.5 |

Codebase-aware development plan:

- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- First story artifact: `_bmad-output/implementation-artifacts/14-1-sdk-wiki-pack-import-and-registry.md`

---

## Story Testability Notes

| Story | Runtime Result | LLM/Tool Result | Verification Result |
|:---|:---|:---|:---|
| 14.1 | SDK Wiki Pack 被校验并注册到 RuntimeStore | `sdk_wiki.list_sdks` 可列出已安装 SDK | 缺 manifest、hash mismatch、schema unsupported 均失败且不污染 registry |
| 14.2 | Index reader 可搜索、读页、解析符号、扩展关系 | 主 LLM 可按需请求 SDK 页面 | 搜索 `施加压力载荷` 命中 SAM 相关页 |
| 14.3 | Settings 可管理 SDK Wiki Packs | 用户可导入目录/压缩包并删除已安装版本 | 失败导入显示 validation report；删除失败不污染 registry |
| 14.4 | `plan_api_usage` 返回 source-referenced plan scaffold | 主 LLM 不编造缺失 API | 缺页时返回 missingInformation |
| 14.5 | SDK tool call 被治理 metadata 观察并写入 audit log | MCP-exposed SDK tools 自主执行；Runtime 不做 SDK 确认门禁 | requested/executed/failed 与 metadata warning 可追溯 |
| 14.6 | SAM golden path 跑通并形成 adapter contract | Runtime 内部完成 SDK 理解，Bridge 不调用 LLM | SAM 压力载荷例子可生成带引用计划并记录 trusted MCP governance audit |

---

## Epic 14 Stories

### Story 14.1: SDK Wiki Pack Import and Registry

As a **Runtime User**,
I want to import a versioned SDK Wiki Pack into Runtime,
So that CrewAgent can use validated SDK knowledge in later sessions.

**Acceptance Criteria:**

**Given** I select an SDK Wiki Pack directory or archive
**When** Runtime imports it
**Then** Runtime validates `sdk-wiki.json`, `index/manifest.json`, required directories, required index files, schema version, sdk id, version, entry, hashes when present, and page frontmatter
**And** Runtime rejects invalid packs with structured errors.

**Given** the pack is valid
**When** import completes
**Then** Runtime stores it under:

```text
RuntimeStore/sdk-wikis/<sdkId>/<version>/
```

**And** updates `sdk-wikis/registry.json` atomically
**And** `sdk_wiki.list_sdks` or equivalent registry API returns the installed SDK.

**Given** import fails at any validation step
**When** I inspect installed SDK Wikis
**Then** no partially registered SDK is visible
**And** the validation report contains the failure code.

### Story 14.2: SDK Wiki Search, Read, Symbol, and Relation Module

As a **Runtime User**,
I want the main LLM to search and read imported SDK Wiki content on demand,
So that SDK answers and recommendations are grounded in source pages.

**Acceptance Criteria:**

**Given** at least one SDK Wiki is installed
**When** Runtime composes the session context
**Then** it exposes a compact SDK Wiki registry, not full page content.

**Given** the LLM calls `sdk_wiki.search_pages`
**When** the query matches indexed content
**Then** Runtime returns bounded results with page id, page type, title, score/snippet when available, and source refs.

**Given** the LLM calls `sdk_wiki.read_page`
**When** the page id exists
**Then** Runtime reads only from the registered SDK Wiki root
**And** returns structured Markdown content and page metadata.

**Given** the LLM calls `sdk_wiki.resolve_symbol` or `sdk_wiki.expand_relations`
**When** the index contains the symbol or relation
**Then** Runtime returns API/workflow/concept/relation references without scanning arbitrary filesystem paths.

### Story 14.3: SDK Wiki Pack Management UI and Remove

As a **Runtime User**,
I want a visible place to import, inspect, and remove SDK Wiki Packs,
So that SDK Wiki knowledge can be managed without developer IPC calls.

**Acceptance Criteria:**

**Given** I open Settings
**When** SDK Wiki Packs are installed
**Then** Runtime shows each installed pack with `sdkId`, `version`, name, language, status, page count, root alias, and hashes when available
**And** no absolute RuntimeStore path is exposed.

**Given** I choose import directory or import archive
**When** the import succeeds
**Then** the Settings list refreshes and shows the new SDK Wiki Pack.

**Given** import validation fails
**When** Runtime returns the structured validation report
**Then** Settings shows the failure code/message and leaves the installed list unchanged.

**Given** I remove an installed SDK Wiki Pack
**When** I confirm the delete action
**Then** Runtime removes that `sdkId@version` from registry and deletes its installed pack directory
**And** the Settings list refreshes.

**Given** removal fails after validation or filesystem operation
**When** Runtime reports the error
**Then** registry state remains consistent and the user sees a structured error.

### Story 14.4: SDK API Usage Planning with Source References

As a **Runtime User**,
I want Runtime to help the main LLM plan SDK API usage in small, source-referenced steps,
So that model-generated SDK operations remain grounded and debuggable.

**Acceptance Criteria:**

**Given** the user asks for an SDK task such as `给当前板架模型施加向下均布压力`
**When** the LLM calls `sdk_wiki.plan_api_usage` with `sdkId`, intent, model state, and constraints
**Then** Runtime returns a structured plan scaffold with:

- `taskType`
- `requiredApis`
- `executionPlan`
- `missingInformation`
- `confidence`
- source refs for every recommended API or plan step.

**Given** relevant API or workflow pages are missing
**When** `sdk_wiki.plan_api_usage` runs
**Then** Runtime reports missing knowledge
**And** does not invent API names.

**Given** an external SDK Bridge is available
**When** planning is needed
**Then** SDK reasoning still occurs inside CrewAgent Runtime main LLM loop
**And** the Bridge is used only for deterministic execution or state retrieval.

### Story 14.5: Trusted SDK Tool Governance and Audit

As a **Runtime User**,
I want SDK tool calls to be risk-classified and logged without redundant Runtime confirmation,
So that trusted MCP tools can execute autonomously while CrewAgent keeps a traceable audit trail.

**Acceptance Criteria:**

**Given** an SDK tool call is classified as `read`
**When** Runtime receives the tool call
**Then** it executes without SDK governance confirmation if effective tool policy allows it.

**Given** an SDK tool call is classified as `model_write`
**When** Runtime receives the tool call
**Then** Runtime logs purpose when provided
**And** logs a metadata warning when purpose is missing
**And** does not reject solely for missing purpose.

**Given** an SDK tool call is classified as `file_write`
**When** Runtime receives the tool call
**Then** Runtime logs target path when provided
**And** relies on MCP/integration software for path safety.

**Given** an SDK tool call is classified as `solve`
**When** Runtime receives the tool call
**Then** Runtime does not require user confirmation solely because of solve risk
**And** the solve status/duration/error is auditable.

**Given** an SDK tool call is classified as `destructive`
**When** Runtime receives the tool call
**Then** Runtime does not reject solely because of destructive risk
**And** MCP/integration software owns destructive safety.

**Given** an SDK tool fails with a traceback
**When** Runtime records the failure
**Then** the audit log includes error code and traceback summary.

### Story 14.6: SAM Golden Path and Generic SDK Adapter Contract

As an **SDK Integration Developer**,
I want SAM to validate the SDK Wiki and trusted MCP governance path without hardcoding SAM into Runtime,
So that future SDKs can reuse the same contract.

**Acceptance Criteria:**

**Given** a SAM SDK Wiki Pack exists
**When** Runtime imports it
**Then** `sdk_wiki.list_sdks` includes `sam`.

**Given** the user asks to apply a pressure load
**When** Runtime plans API usage
**Then** the plan surfaces Pressure, Surface, and StaticStep related pages from the SAM Wiki
**And** every API recommendation includes source refs.

**Given** the plan leads to a SAM model write or solve tool call
**When** the tool call enters Runtime
**Then** Runtime observes `model_write` or `solve` metadata without adding a user confirmation gate
**And** audit logs include tool name, purpose, risk, governance state/warnings, status, duration, and traceback summary if failed.

**Given** another SDK such as Abaqus, Ansys, or OpenFOAM provides a pack with the same generic structure
**When** Runtime imports it
**Then** Runtime uses the same import, registry, internal wiki module, and risk policy path without SAM-specific branching.

---

## Open Questions

- SDK Wiki Pack 管理首版放在 Settings subsection；后续是否需要独立页面取决于 pack 数量和运维复杂度。
- `sdk_wiki.plan_api_usage` 首版是否只返回 retrieval-backed scaffold，还是直接返回 LLM-visible plan draft？
- `file_write` “当前用户请求已明确要求该输出”的判断是否先采用显式 tool argument 标记，再逐步增强为 intent classifier？
- `destructive` MVP 是全部拒绝，还是允许手工白名单？
