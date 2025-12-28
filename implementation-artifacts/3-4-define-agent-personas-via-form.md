# Story 3.4: Define Agent Personas via Form

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to define Agent personas through a form-based UI,  
so that each step has a specialized AI assistant.

## Acceptance Criteria

1. **Given** I click "Manage Agents" in the Editor  
   **When** I fill out the Agent form (name, role, communication_style, persona)  
   **Then** the Agent is added to `agents.json`  
   **And** it appears in the Agent dropdown for Step assignment

## Design

### Summary

- Editor（`/editor/[workflowId]`）新增 “Manage Agents” 入口（弹窗）。
- 弹窗提供 Agent 列表 + 表单：
  - name / role / communication_style / persona
  - 新增/编辑/删除（MVP：至少新增）
- Agents 保存策略：
  - 前端维护 `agents` 内存态，并生成 `agents.json (preview)`。
  - 同步写入后端 `workflow_packages.agents_json`（持久化，刷新仍保留）。
- Step 编辑弹窗中 Agent 字段改为 dropdown（从 `agents` 列表选择）。

### UX / UI

- Editor Header：
  - Button：`Manage Agents`
- Manage Agents Modal：
  - 左侧：Agent 列表（name）
  - 右侧：表单（name/role/communication_style/persona）
  - 操作：New / Save / Delete / Close
- Step Edit Modal：
  - Agent：`<select>`（None + agents）

### API / Contracts

- `PUT /packages/{id}/agents`（auth required）
  - Request: `{ "agents": Array<{ "name": string, "role": string, "communication_style": string, "persona": string }> }`
  - Success (200): `{ "data": { "id": number, "name": string, "workflowMd": string, "agentsJson": string }, "error": null }`
  - Failure:
    - 400：字段校验失败
    - 401：未登录 / token 无效
    - 404：不存在或无权限

### Errors / Edge Cases

- 后端返回的 `agentsJson` 非法 JSON：前端提示并回退到空列表（不阻塞编辑）。
- name 重名：MVP 允许，但前端可提示（建议唯一）。
- Agent 列表为空：Step 编辑 dropdown 显示 “No agents yet”，并提示去 Manage Agents 创建。

### Test Plan

- 手工冒烟：
  - Editor → Manage Agents → 新建 Agent 保存 → Step 编辑 dropdown 可选择该 Agent
  - 刷新页面 → Agents 仍存在（后端持久化）
- 工程校验：
  - Backend：`./.venv/bin/ruff check .`、`./.venv/bin/pytest -q`
  - Frontend：`npm run lint`、`npm run build`

## Tasks / Subtasks

- [x] 1) Backend：更新 agents_json（AC: 1）
  - [x] 新增 schema：AgentDefinition + AgentsUpdate
  - [x] 新增 `PUT /packages/{id}/agents`（auth + owner scope）
  - [x] pytest：owner 可更新 / 非 owner 404 / 校验错误 400

- [x] 2) Frontend：Manage Agents UI（AC: 1）
  - [x] Editor header 增加入口（modal）
  - [x] Agent 表单新增/编辑/删除（MVP：至少新增）
  - [x] 调用 `PUT /packages/{id}/agents` 持久化并更新 `data.agentsJson`

- [x] 3) Frontend：Step Agent dropdown（AC: 1）
  - [x] Step 编辑弹窗 Agent 字段改为 dropdown
  - [x] Node data.agent 保存 agent name

- [x] 4) 校验
  - [x] Backend：`ruff` + `pytest`
  - [x] Frontend：`npm run lint` + `npm run build`

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.4）
- `_bmad-output/architecture.md`（Agent Personas + Builder UI）

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

- Editor 新增 “Manage Agents” 弹窗：支持新增/编辑/删除 Agent，并持久化到后端 `workflow_packages.agents_json`。
- Step 编辑弹窗 Agent 字段改为 dropdown（从已定义 Agent 列表选择）。
- Editor 增加 `agents.json (preview)` 与 `agents.json (stored)` 展示，便于确认保存结果。
- 手工验证通过：新增 Agent 后可在 Step 下拉选择；刷新后仍可从后端加载 agentsJson。

### File List

- `_bmad-output/implementation-artifacts/3-4-define-agent-personas-via-form.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/tests/test_packages.py`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/lib/api-client.ts`
