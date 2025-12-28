# Story 3.9: ProjectBuilder Multi-Workflow Management (Create/Select)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to create and manage multiple workflows within a single project,  
so that I can build multi-workflow packages and keep workflows organized.

## Acceptance Criteria

1. **Given** I am in ProjectBuilder  
   **When** I create a workflow (name required)  
   **Then** the workflow appears in the workflows list  
   **And** it is persisted and visible after refresh

2. **Given** a project contains multiple workflows  
   **When** I open workflow A then workflow B  
   **Then** each workflow loads and saves its own graph/content independently

3. **Given** I open an existing legacy project (created before multi-workflow)  
   **When** I open ProjectBuilder  
   **Then** I see exactly one default workflow containing the legacy editor content  
   **And** I can create additional workflows without breaking the legacy one

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- 引入 Project→Workflows 存储模型：Project（现 `workflow_packages`）共享 agents；Workflow 独立保存 graph/workflowMd/stepFiles。
- ProjectBuilder 支持创建/列表/打开多个 workflow；WorkflowEditor 切换为按 `projectId + workflowId` 加载与保存。
- 迁移 legacy 单 workflow：每个旧项目自动生成 1 个“默认工作流”，内容与旧 editor 完全一致，且可继续新增其他 workflows。

### UX / UI

- ProjectBuilder（`/builder/[projectId]`）
  - Workflows 面板新增按钮：`新建工作流`
    - 点击后弹窗：工作流名称（必填，1–200）
    - 提交成功：workflows 列表立即出现新条目；刷新后仍存在
  - Workflows 列表项：
    - 展示：工作流名称 + “打开”
    - 点击后进入 WorkflowEditor（本 story 建议路由：`/editor/[projectId]/[workflowId]`）
  - Legacy 项目：
    - Workflows 列表应恰好有 1 条 `默认工作流`（内容为旧 editor 的 graph/workflowMd/stepFiles）

### API / Contracts

> 约定：所有接口保持统一响应 `{ data, error }`；鉴权使用 `Authorization: Bearer <token>`。
> 说明：本期为降低改动量，沿用 `/packages` 作为“Project”命名空间；可在后续再引入 `/projects` 别名。

- `GET /packages/{projectId}`
  - 用途：ProjectBuilder 获取 project 基础信息与 `agentsJson`（agents 共享在 project 级；story 3.10 再升级结构）
  - 成功：`{ data: { id, name, agentsJson, ... }, error: null }`
  - 失败：404 `PACKAGE_NOT_FOUND`（不存在/无权限）

- `GET /packages/{projectId}/workflows`
  - 返回该 project 的 workflows 列表（用于 ProjectBuilder 渲染）
  - 成功：`{ data: Array<{ id: number, name: string, isDefault: boolean }>, error: null }`

- `POST /packages/{projectId}/workflows`
  - Request：`{ name: string }`（1–200）
  - 成功（201）：`{ data: { id: number, name: string, isDefault: false }, error: null }`
  - 失败：
    - 400 `VALIDATION_ERROR`（name 缺失/过长）
    - 404 `PACKAGE_NOT_FOUND`

- `GET /packages/{projectId}/workflows/{workflowId}`
  - 返回 workflow 独立内容（供 WorkflowEditor 加载）
  - 成功：`{ data: { id, projectId, name, workflowMd, graphJson, stepFilesJson }, error: null }`
  - 失败：404 `WORKFLOW_NOT_FOUND`（或 `PACKAGE_NOT_FOUND`，取决于后端实现策略）

- `PUT /packages/{projectId}/workflows/{workflowId}/content`
  - Request：沿用现有保存结构：
    - `workflow_md: string`
    - `graph: { nodes: [], edges: [] }`
    - `step_files: Record<string, string>`
  - 成功：返回更新后的 workflow detail（同 `GET`）

### Data / Storage

- DB 演进（建议复用现有表名，降低迁移成本）：
  - `workflow_packages`：视为 Project 表（已有 `owner_user_id/name/agents_json` 等）
  - 新增表：`project_workflows`（或命名 `package_workflows`，与代码风格一致）
    - `id`（PK）
    - `project_id`（FK → `workflow_packages.id`，index）
    - `name`（String(200)）
    - `is_default`（bool，默认 false）
    - `workflow_md`（Text）
    - `graph_json`（Text）
    - `step_files_json`（Text）
    - `created_at`（timestamp, default now）
- 迁移策略（Alembic）：
  - 对每条既有 `workflow_packages`：
    - 创建 1 条 workflow：`name="默认工作流"`, `is_default=true`，并把旧的 `workflow_md/graph_json/step_files_json` 迁移进去
  - 新创建项目：
    - 创建 project 后自动创建 1 条默认 workflow（初始内容可复用现有模板/空 graph）
- 兼容策略（建议）：
  - 保留现有 `/packages/{id}` / `/packages/{id}/content` 一段时间，映射到默认 workflow（避免旧链接/旧逻辑立刻断裂）

### Errors / Edge Cases

- 创建 workflow：
  - name 为空/超长：前端表单即时校验 + 后端 400 `VALIDATION_ERROR`（显示字段级错误）
- 打开 workflow：
  - workflow 不属于该 project：后端返回 404 `WORKFLOW_NOT_FOUND`
  - project 不存在/无权限：404 `PACKAGE_NOT_FOUND`
- 迁移/legacy：
  - 确保每个 project 只有一个默认 workflow（迁移脚本需避免重复插入；必要时以 `is_default` 记录并在代码层做幂等保护）
- 数据隔离：
  - `PUT .../content` 必须只更新该 workflow 行，禁止覆盖同 project 的其他 workflow

### Test Plan

- Backend（pytest）：
  - Auth：未登录访问 workflows 相关接口 → 401
  - 创建 project 后：`GET /packages/{projectId}/workflows` 返回 1 个默认 workflow
  - 创建 workflow A/B：列表返回 3 条（含 default）
  - 分别更新 A/B content：再读取确认互不污染（graphJson/stepFilesJson 不串）
  - 访问他人 project/workflow → 404
- Frontend（手工冒烟）：
  - ProjectBuilder：新建 workflow → 列表出现 → 刷新仍在
  - 进入 workflow A 编辑并保存 → 切到 workflow B 编辑并保存 → 切回 A 内容不变
  - legacy 项目：默认 workflow 内容与旧 editor 一致且可继续编辑

## Tasks / Subtasks

- [x] 1) 后端：Project/Workflow 数据模型与 migration
- [x] 2) 后端：workflows CRUD API（owner scope）
- [x] 3) 前端：ProjectBuilder Create Workflow + list + open
- [x] 4) 旧数据迁移/兼容：legacy 单 workflow → 默认 workflow

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- `_bmad-output/epics.md`（Epic 3 / Story 3.9）
- `_bmad-output/implementation-artifacts/3-8-project-builder-shell-full-screen-and-navigation.md`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `crewagent-builder-backend/.venv/bin/pytest -q`
- Frontend:
  - `npm run build`
  - `npm run lint`

### Completion Notes List

- 新增 `package_workflows` 存储多工作流，并提供 `/packages/{projectId}/workflows*` API（owner scope）。
- ProjectBuilder 支持创建/列表/打开 workflows；新增 editor 路由 `/editor/[projectId]/[workflowId]`，每个 workflow 独立加载/保存内容。
- legacy 兼容：新项目自动创建 1 个默认 workflow；旧项目若缺失 workflows，将按 `workflow_packages` 旧字段懒创建默认 workflow；默认 workflow 与旧 `/packages/{id}` content 双向同步，避免旧 editor/导出断裂。

### File List

- `_bmad-output/implementation-artifacts/3-9-project-builder-multi-workflow-management.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/crewagent.db`
- `crewagent-builder-backend/README.md`
- `crewagent-builder-backend/app/main.py`
- `crewagent-builder-backend/app/models/package_workflow.py`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/schemas/package_workflow.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/app/services/package_workflow_service.py`
- `crewagent-builder-backend/migrations/versions/8f3b2c1a6d7e_add_package_workflows_table.py`
- `crewagent-builder-backend/tests/test_package_workflows_api.py`
- `crewagent-builder-backend/tests/test_package_workflows_legacy_compat.py`
- `crewagent-builder-backend/tests/test_package_workflows_model.py`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`

### Change Log

- 实现 multi-workflow 后端模型与 API；补齐 legacy/default workflow 兼容与双向同步；前端支持创建/切换 workflows 并进入新 editor 路由。
- 升级本地 SQLite DB（`crewagent-builder-backend/crewagent.db`）到最新 Alembic revision，创建 `package_workflows` 表，避免运行时报 “no such table”。
