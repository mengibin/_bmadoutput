# Story 12.3: Project KB Import / Extract / Index / Retrieval Tool

Status: review

## Story

As a **Project Executor**,  
I want each project to have an isolated local knowledge space that can ingest reference files and solidify them into project knowledge,  
So that Runtime can keep project knowledge durable and expose a project-scoped retrieval tool for LLM use without depending on the original external files.

## Acceptance Criteria

### AC-1: 项目知识库初始化与空态

**Given** a new project is created  
**When** Runtime initializes project data  
**Then** it creates an empty project knowledge directory at `runtime-store/projects/<projectId>/knowledge/`  
**And** project-level `Knowledge` panel（与 `Files` / `Works` 同级）shows an empty knowledge state

### AC-2: 文件与文件夹导入

**Given** I am in the project `Knowledge` panel  
**When** I start ingestion  
**Then** I can choose either files or a folder  
**And** supported content types include `.pdf`, `.docx`, `.md`, `.txt`, `.png`, `.jpg`, `.jpeg`, `.webp`  
**And** imported sources are copied into runtime-private storage instead of relying on the original path

### AC-3: 知识固化与处理状态可见（运营级）

**Given** ingestion runs  
**When** extraction and indexing complete or partially fail  
**Then** the panel shows total knowledge file count  
**And** the panel shows the knowledge file list  
**And** each file exposes at least processing status, import time, and failure reason when failed  
**And** partial failure of one file does not block the rest of the batch

### AC-4: Embedding 开关与配置

**Given** I open the `Knowledge` panel with no embedding config yet  
**When** I want semantic indexing enabled  
**Then** the `Knowledge` panel provides an `embedding is enable` switch  
**And** turning it on reveals embedding LLM configuration fields  
**And** this story only requires OpenAI-style embedding interfaces (`OpenAI` / `OpenAI Compatible`)  
**And** saving here persists the same global setting used by `Settings`  
**And** if embedding has already been configured, the switch defaults to enabled

### AC-5: 提供给 LLM 的项目知识检索 Tool

**Given** a project has ready knowledge sources  
**When** Runtime receives a project knowledge retrieval request from the LLM tool layer  
**Then** it queries the project-scoped knowledge index and returns relevant chunks plus source metadata  
**And** it only returns content from `ready` knowledge sources  
**And** this story does not require a human-facing search UI

## Technical Notes

- Delivery pattern: vertical full-stack in one story (Main/IPC/Renderer/Test together).
- Project KB storage is runtime-private; do not place source files under project root.
- UI only serves onboarding and operational visibility; the primary consumer of Project KB is the retrieval tool invoked by LLM.
- This story does not require a human-facing retrieval/search UI or human-oriented knowledge reading experience.
- Import must support both file selection and folder selection.
- Extraction pipeline should be best-effort per file; partial failures cannot abort the whole batch.
- `embeddingLlm` remains a global runtime setting, but the `Knowledge` panel may inline-edit that same setting for onboarding convenience.
- Story 12.3 的 embedding 配置范围已收敛为 OpenAI-style interface only（`OpenAI` / `OpenAI Compatible`）；Gemini / Azure / Ollama embedding UI 不在本 story 范围内。
- 图片与扫描型 PDF 在多模态可用时优先走多模态提取；本地 OCR / PDF text-layer 只作为降级路径或纯文本 PDF 快路径。
- Project KB retrieval must be project-scoped and served from the Runtime tool layer, returning only `ready` indexed content.
- Design: `_bmad-output/implementation-artifacts/design-12-3-project-kb-import-extract-index-search.md`
- Single-Story Review: `_bmad-output/implementation-artifacts/code-review-12-3-project-kb-import-extract-index-search.md`

## Tasks / Subtasks

- [x] Task 1: Runtime Main + Store（后端）实现（AC: 1,2,3,4,5）
  - [x] 1.1 新项目创建时初始化 `knowledge/` 目录结构
  - [x] 1.2 实现文件导入流水线（source -> extracted -> index）
  - [x] 1.3 实现文件夹导入流水线（directory scan -> file ingest）
  - [x] 1.4 实现按文件失败隔离与错误归档
  - [x] 1.5 实现 knowledge 状态与文件清单读取
  - [x] 1.6 实现 project-scoped retrieval query helper，仅检索 `ready` sources

- [x] Task 2: IPC + Tool Surface（前后端接口）实现（AC: 1,2,3,4,5）
  - [x] 2.1 暴露文件选择与文件夹选择接口
  - [x] 2.2 暴露导入任务进度与逐文件结果
  - [x] 2.3 暴露空态/统计/文件列表接口
  - [x] 2.4 复用或扩展设置保存接口，支持从 `Knowledge` 面板保存 embedding 配置
  - [x] 2.5 在 Runtime ToolHost 暴露 project knowledge retrieval tool

- [x] Task 3: Renderer UI（前端）实现（AC: 1,3,4）
  - [x] 3.1 项目级 `Knowledge` 面板新增 KB 空态、文件总数与文件列表
  - [x] 3.2 导入面板展示逐文件进度、成功/失败原因
  - [x] 3.3 `Knowledge` 面板中增加 `embedding is enable` 开关与内联配置区
  - [x] 3.4 若全局 embedding 已配置，则 `Knowledge` 面板默认显示开关开启状态

- [x] Task 4: Settings 对齐（AC: 4）
  - [x] 4.1 在 `Settings` 中增加 `embeddingLlm` 配置
  - [x] 4.2 `Knowledge` 面板保存的 embedding 配置与 `Settings` 读写同一份状态
  - [x] 4.3 embedding 请求失败时不阻断导入，并正确反馈状态

- [x] Task 5: Integration & E2E 验证（AC: 1,2,3,4）
  - [x] 5.1 混合文件导入与部分失败场景验证
  - [x] 5.2 文件夹导入验证
  - [x] 5.3 文件列表/文件状态/UI 映射验证
  - [x] 5.4 embedding 开关默认值与保存路径验证
  - [x] 5.5 项目隔离验证（跨 project 不串读）
  - [x] 5.6 LLM retrieval tool 验证（仅返回 ready sources，且带 source metadata）

- [x] Task 6: 回归与文档
  - [x] 6.1 不影响既有文件系统工具能力
  - [x] 6.2 更新 file list 与运行说明

- [x] Task 7: Single-Story Review Follow-ups（Blocking）（AC: 4,5）
  - [x] 7.1 将 `Settings` 与 `Knowledge` 面板统一收敛到 OpenAI-style embedding 接口范围（`OpenAI` / `OpenAI Compatible`）
  - [x] 7.2 打开或更新 embedding 配置后，为已导入的 project knowledge 提供显式 reindex / backfill 行为，或明确收窄产品契约
  - [x] 7.3 `project_kb.retrieve` 返回结果移除 `originalPath`，仅保留 runtime-private metadata

- [x] Task 8: Second Review Follow-ups（Blocking）（AC: 3,4,5）
  - [x] 8.1 当全局 embedding 配置发生漂移时，为已有 project knowledge 检测 stale vectors，并对非 active project 也执行 rebuild / invalidation 策略
  - [x] 8.2 从 `kb:project:getStatus` 与 renderer store 的 `ProjectKnowledgeSourceRecord` 中移除 `originalPath`，避免宿主机路径继续暴露到前端

## Definition of Done

- [x] AC-1~AC-5 对应实现已完成并接入主进程、预加载、渲染层。
- [x] `Knowledge` 面板只承担导入、状态和 embedding onboarding，不提供人工检索 UI。
- [x] `project_kb.retrieve` 已暴露给 Runtime ToolHost，并只返回 project-scoped ready chunks。
- [x] TypeScript 全量检查通过。
- [x] targeted verification 已通过，见下方 `Verification`。

## Dev Notes (2026-03-12)

- 新增 `ProjectKbService`，负责 project knowledge 的 source copy、提取、chunk 化、`index.sqlite` 建表/写入，以及 retrieval query。
- 项目知识导入支持文件和文件夹；支持 `.pdf/.docx/.md/.txt/.png/.jpg/.jpeg/.webp`，按文件 best-effort 处理，单文件失败不会阻断整批。
- `media.extract` 被复用于图片/扫描 PDF 的提取路径；纯文本 `.md/.txt` 和 `.docx` 走本地快速路径。
- Runtime ToolHost 新增 `project_kb.retrieve`，供 LLM 按当前 project 做检索；只返回 `ready` chunks 与 source metadata。
- Renderer 新增项目级 `Knowledge` 面板，与 `Files / Works` 同级；左侧卡片支持 `embedding is enable` 开关和内联 embedding LLM 配置，导入后显示知识文件总数、文件列表与处理状态。
- `Settings` 新增全局 `Embedding LLM` 配置区，与 `Knowledge` 空态共用同一份持久化设置。
- Review 决议补充：12.3 的 embedding onboarding 范围现已收敛为 OpenAI-style interface only；provider-specific embedding UI 不再作为本 story 的目标能力。

## Current Single-Story Review Follow-ups (2026-03-12)

- 当前结论：第二轮 review follow-ups 已完成，story 回到 `review`。
- 修复结果：
  - Project KB manifest 现在会记录 embedding config fingerprint；当全局 embedding 配置漂移时，`Knowledge` 状态会把 embedding 标记为 not ready，retrieval 会在使用前自动 rebuild stale vectors；
  - `kb:project:getStatus` / 导入返回状态 已切换到 renderer-safe payload，不再把 `originalPath` 暴露到前端 store。

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/projectKbService.ts` (new)
- `crewagent-runtime/electron/services/projectKbService.test.ts` (new)
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx` (new)
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css` (new)
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.css`
- `crewagent-runtime/src/components/layout/Sidebar.tsx`
- `crewagent-runtime/src/components/layout/AppShell.tsx`
- `crewagent-runtime/src/vite-env.d.ts`

## Verification

- `cd crewagent-runtime && ./node_modules/.bin/tsc --noEmit --pretty false`
  - TypeScript 全量检查通过（2026-03-12）。
- `cd crewagent-runtime && npx vitest run --testTimeout=30000 electron/services/projectKbService.test.ts`
  - 7 个 Project KB 服务测试全部通过（2026-03-12），包含 stale embedding drift rebuild 与 renderer-safe status payload 相关回归。
- `cd crewagent-runtime && npx vitest run --testTimeout=30000 electron/services/fileSystemToolHost.test.ts -t "exposes project_kb.retrieve and forwards retrieval requests"`
  - `project_kb.retrieve` tool surface 定向测试通过（2026-03-12）。
- `cd crewagent-runtime && npx vitest run electron/stores/runtimeStore.test.ts -t "initializes and persists embedding settings independently|normalizes embedding providers to OpenAI-style interfaces"`
  - embedding 设置持久化与 OpenAI-style provider 归一化定向测试通过（2026-03-12）。

## Change Log

- 2026-03-12: 完成 12-3 vertical slice，实现 project knowledge import / extract / index / retrieval tool、Knowledge 面板和 embedding 配置联动，状态推进到 `review`。
- 2026-03-12: 完成 single-story review follow-ups，修复 OpenAI-style embedding scope 对齐、已导入知识的 embedding backfill / rebuild，以及 `project_kb.retrieve` 的 `originalPath` 泄露问题。
- 2026-03-12: 完成第二轮 code review follow-ups，修复跨项目 stale embedding vectors 检测/重建，以及 `kb:project:getStatus` 的 `originalPath` 暴露问题，状态恢复到 `review`。
