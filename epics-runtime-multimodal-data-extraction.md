---
stepsCompleted: [1, 2, 3, 9]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-multimodal-data-extraction.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture/runtime-multimodal-data-extraction-architecture.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/architecture.md'
workflowType: 'epics'
lastStep: 9
project_name: 'CrewAgent Runtime Multimodal'
user_name: 'Mengbin'
date: '2026-02-27'
---

# CrewAgent Runtime Multimodal Data Extraction - Epic Breakdown

## Overview

本文件将多模态子需求拆解为**单 Epic + 合并 Story**方案，覆盖：

- 聊天面板拖拽文件/文件夹输入
- `media.extract` first-class function-call 直连路径
- schema 传入/意图生成 + strict 校验
- 多模态模型配置与能力校验
- `fs.read` 二进制 fallback

来源文档：

- `_bmad-output/prd-runtime-multimodal-data-extraction.md`
- `_bmad-output/architecture/runtime-multimodal-data-extraction-architecture.md`

---

## Requirements Inventory

### Functional Requirements

- FR-MULTI-01: 支持图片/固定版式文档多模态提取，聊天面板拖拽输入，输出结构化数据。
- FR-MULTI-02: `media.extract` 直接暴露给 LLM 作为 first-class function call；`fs.read` 仅 fallback，且不得返回乱码。
- FR-MULTI-03: 详细要求见子 PRD（工具契约、验证路径、输出约束）。
- FR-MULTI-04: 提供多模态 LLM 配置（provider/baseUrl/model/apiKey/timeout），并在不支持时返回明确错误码。

### Non-Functional Requirements

- NFR-MULTI-01: 可追溯性（sourceFile/page）。
- NFR-MULTI-02: 稳定性（单文件失败不阻断整批）。
- NFR-MULTI-03: 可观测性（每文件/页日志与指标）。
- NFR-MULTI-04: 沙箱安全（`@project/@pkg/@state`）。

---

## FR Coverage Map

| FR/NFR | Epic |
|:---|:---|
| FR-MULTI-01~04 | Epic MDE-1 |
| NFR-MULTI-01~04 | Epic MDE-1 |

---

## Epic List

### Epic MDE-1: Runtime Multimodal Data Extraction

**Goal**: 在 Runtime 中一次性交付多模态数据提取核心能力：模型配置、first-class `media.extract`、拖拽输入、schema 驱动抽取、fallback 与可观测性。  
**FRs covered**: FR-MULTI-01~04, NFR-MULTI-01~04

**Deliverables**:

- 多模态模型配置（provider/baseUrl/model/apiKey/timeout）与能力校验。
- `media.extract` first-class tool + `fs.read` 二进制 fallback 策略。
- 聊天拖拽文件/文件夹输入 + 递归发现 + 批处理抽取。
- schema provided/generated 双路径 + strict 校验。
- 默认返回结构化提取结果（可选持久化到 `@project/artifacts/extracted-data.json`）。
- 提取日志与关键指标。
- E2E 验收用例（混合文件、模型不支持、损坏文件）。

---

## Epic MDE-1 Stories

### Story MDE-1.1: Multimodal Model Configuration + Capability Guard

Story Artifact: `_bmad-output/implementation-artifacts/mde-1-1-multimodal-model-configuration-capability-guard.md`

As a **User**,  
I want to configure and validate a dedicated multimodal model profile,  
So that extraction uses a supported provider/model and fails deterministically when unsupported.

**Acceptance Criteria:**

**Given** Runtime settings are opened  
**When** I save multimodal configuration  
**Then** fields `provider/baseUrl/model/apiKey/timeout` are persisted  
**And** invalid required fields are rejected with actionable errors.

**Given** a configured model without multimodal capability  
**When** extraction starts  
**Then** system returns `LLM_MULTIMODAL_NOT_SUPPORTED`  
**And** no extraction call is sent to provider.

### Story MDE-1.2: First-Class `media.extract` + Binary Fallback Policy

Story Artifact: `_bmad-output/implementation-artifacts/mde-1-2-first-class-media-extract-binary-fallback-policy.md`

As a **System**,  
I want `media.extract` exposed directly in tool registry and `fs.read` constrained to fallback behavior for binaries,  
So that LLM follows the direct multimodal path safely.

**Acceptance Criteria:**

**Given** a chat/agent run with tool access  
**When** LLM requests available tools  
**Then** `media.extract` appears in tool definitions  
**And** includes args `path/instruction/schema/schemaIntent/page/strict/documentTypeHint`.

**Given** an image or PDF path  
**When** `media.extract` is called  
**Then** tool returns `ok/data/path/page` (or structured error)  
**And** output contains `schemaSource` and `schemaUsed` when applicable.

**Given** an OpenAI-compatible provider that rewrites tool names with hyphens  
**When** assistant tool calls return names like `media-extract`  
**Then** runtime decodes/routes them back to `media.extract` deterministically  
**And** extraction does not get misrouted to unrelated tools.

**Given** `fs.read` is called on binary/image/PDF  
**When** tool processes request  
**Then** it does not return garbled text  
**And** returns structured hint metadata pointing to `media.extract`.

### Story MDE-1.3: Drag-and-Drop Batch Extraction + Schema Contract + Observability

Story Artifact: `_bmad-output/implementation-artifacts/mde-1-3-drag-and-drop-batch-extraction-schema-contract-observability.md`

As a **User**,  
I want to drag files/folders into chat and let LLM decide read vs extract, with optional schema-valid batch output,  
So that multimodal processing is directly usable by downstream systems without forced extraction.

**Acceptance Criteria:**

**Given** I drag files/folders into chat  
**When** I send message  
**Then** payload includes attachment metadata  
**And** backend receives alias-resolved paths.

**Given** a dropped folder with nested subfolders  
**When** ingestion runs  
**Then** supported files are discovered recursively  
**And** processing order is deterministic (filename asc by default).

**Given** `schema` is provided  
**When** extraction runs  
**Then** schema source is marked as `provided`.

**Given** `schema` is absent and `schemaIntent` exists  
**When** extraction runs  
**Then** schema is generated and source is marked `generated`.

**Given** strict mode is enabled  
**When** extracted output violates schema  
**Then** run returns structured validation error  
**And** invalid record is not silently written.

**Given** multiple files/pages are processed  
**When** run completes  
**Then** runtime returns structured extraction payload (`runId/generatedAt/records/errors/stats`)  
**And** each record contains at least `sourceFile/page/data`  
**And** persistence is optional (`persistArtifact=true`).

**Given** extraction is executed  
**When** logs are written  
**Then** each entry includes `runId/sourceFile/page/model/status/duration/errorCode`.

**Given** a test fixture with mixed images + PDFs  
**When** E2E suite runs  
**Then** it validates output file existence, schema compliance, and deterministic error behavior (`LLM_MULTIMODAL_NOT_SUPPORTED` path).
