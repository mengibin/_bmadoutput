# Story 3.1: Create New Workflow Project

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to create a new Workflow project in the Builder,  
so that I can start designing my workflow.

## Acceptance Criteria

1. **Given** I am logged into the Builder Dashboard  
   **When** I click "New Workflow" and enter a project name  
   **Then** a new project is created with default `workflow.md` and `agents.json`  
   **And** I am taken to the Editor page for this project

## Design

### Summary

- Frontend Dashboard 增加 “New Workflow” 创建入口：输入项目名后调用后端创建项目 API。
- Backend 增加受保护的 `packages`（Workflow Project）API：
  - `POST /packages` 创建项目（默认生成 `workflow.md` + `agents.json` 内容并落库）
  - `GET /packages` 列表（Dashboard 可展示）
  - `GET /packages/{id}` 详情（Editor 页面加载）
- Editor 页面采用动态路由：`/editor/[workflowId]`，并在无 token 时重定向回 `/login`。

### UX / UI

- `/dashboard`
  - 按钮：`New Workflow`
  - 弹窗（或 inline form）：Project Name（必填，1-200）
  - 成功：跳转到 `/editor/{workflowId}`
  - 失败：显示错误提示（不暴露内部堆栈）

### API / Contracts

- Headers：
  - `Authorization: Bearer <accessToken>`
- Backend API：
  - `POST /packages`
    - Request: `{ "name": string }`
    - Success (201): `{ "data": { "id": number, "name": string }, "error": null }`
    - Failure:
      - 400：参数校验失败
      - 401：未登录 / token 无效
  - `GET /packages`
    - Success (200): `{ "data": Array<{ "id": number, "name": string }>, "error": null }`
  - `GET /packages/{id}`
    - Success (200): `{ "data": { "id": number, "name": string, "workflowMd": string, "agentsJson": string }, "error": null }`
    - Failure:
      - 401：未登录 / token 无效
      - 404：不存在或无权限

### Data / Storage

- DB：复用 `workflow_packages` 作为 Workflow Project 存储（本期扩展字段）：
  - `id`
  - `owner_user_id`
  - `name`
  - `workflow_md`（默认模板）
  - `agents_json`（默认 `[]`）
  - `created_at`（UTC）

### Errors / Edge Cases

- Project name 重名：允许（MVP）；后续可加唯一约束（按用户范围）。
- 401：缺少/无效/过期 token。
- 404：访问非本人的项目。

### Test Plan

- Backend：
  - 未登录调用 `/packages` → 401 + 结构化错误
  - 创建项目成功 → 返回 id + name
  - 创建后可通过 `/packages/{id}` 获取默认内容
  - 访问他人项目 → 404
- Frontend：
  - Dashboard 创建成功 → 跳转 `/editor/{id}`
  - 刷新 Editor 页面仍能加载（token 存在）

## Tasks / Subtasks

- [x] 1) Backend：Workflow Project CRUD（AC: 1）
  - [x] 扩展 `workflow_packages` 模型与 Alembic migration
  - [x] `POST /packages`（auth required）
  - [x] `GET /packages`（auth required）
  - [x] `GET /packages/{id}`（auth required）
  - [x] pytest 覆盖

- [x] 2) Frontend：Dashboard 创建入口（AC: 1）
  - [x] 创建表单 + 调用 `POST /packages`（携带 Authorization）
  - [x] 成功跳转 `/editor/{id}`

- [x] 3) Frontend：Editor 动态路由占位（AC: 1）
  - [x] 新增 `/editor/[workflowId]` 页面（auth guard）
  - [x] 调用 `GET /packages/{id}` 拉取并显示项目名/默认内容占位

- [x] 4) 校验
  - [x] Backend：`ruff` + `pytest`
  - [x] Frontend：`npm run lint` + `npm run build`

## Dev Notes

- API 响应必须使用统一 `{ data, error }` 格式（全局异常处理器已实现）。
- 后端鉴权复用 Story 2.3 的 `get_current_user`。
- Editor 页面路由结构对齐 `_bmad-output/architecture.md`：`/editor/[workflow_id]`。

### References

- `_bmad-output/architecture.md`（Builder Frontend/Backend structure）
- `_bmad-output/epics.md`（Epic 3 / Story 3.1）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `./.venv/bin/ruff check .`
  - `./.venv/bin/pytest -q`
- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- 新增 Workflow Project（复用 `workflow_packages`）的创建/列表/详情 API（均需 JWT）。
- Dashboard 支持创建项目并展示项目列表；创建成功跳转 `/editor/{workflowId}`。
- Editor 使用动态路由 `/editor/[workflowId]`，加载并展示默认 `workflow.md` / `agents.json`（占位）。
- 手工验证通过：登录后可创建项目并跳转 Editor；刷新后仍可加载项目。

### File List

- `_bmad-output/implementation-artifacts/3-1-create-new-workflow-project.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/models/workflow_package.py`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/migrations/versions/9b1a2f0c3f4d_add_workflow_package_fields.py`
- `crewagent-builder-backend/tests/test_packages.py`
- `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/lib/api-client.ts`
