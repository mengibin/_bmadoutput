# Story 3.5: Configure Agent Prompt Templates

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to configure System Prompt and User Prompt templates for each Agent,  
so that the AI understands each step's task.

## Acceptance Criteria

1. **Given** I am editing an Agent  
   **When** I enter a System Prompt template (with `{{placeholders}}`)  
   **Then** the template is saved to the Agent definition  
   **And** it will be used during Runtime execution

## Design

### Summary

- 扩展 AgentDefinition（存入 `agents.json`）：
  - `system_prompt_template`（string，允许空）
  - `user_prompt_template`（string，允许空）
- Builder（Editor → Manage Agents）表单新增两个 textarea：
  - System Prompt Template
  - User Prompt Template
- 保存策略：复用 Story 3.4 的 `PUT /packages/{id}/agents`，由后端保存到 `workflow_packages.agents_json`。

### UX / UI

- Manage Agents Modal（编辑 Agent 时）：
  - 新增 “Prompt Templates” 区块：
    - System Prompt Template（多行）
    - User Prompt Template（多行）
  - 提示：支持 `{{placeholders}}`；本 Story 仅存储，不做渲染替换（Story 3.6/Runtime 再使用）。

### Errors / Edge Cases

- 旧数据不含新字段：前端解析时默认空字符串；保存时补齐字段。
- 模板过长：前端限制（建议 20k 以内）；后端限制在 schema 中（MVP 可宽松）。

### Test Plan

- 手工冒烟：
  - 在 Manage Agents 中给 Agent 填写 System/User templates 并保存
  - 刷新页面确认 templates 仍存在（来自后端 agentsJson）
  - 选择该 Agent 绑定到 Step（下拉可选）
- 工程校验：
  - Backend：`./.venv/bin/ruff check .`、`./.venv/bin/pytest -q`
  - Frontend：`npm run lint`、`npm run build`

## Tasks / Subtasks

- [x] 1) Backend：扩展 AgentDefinition（AC: 1）
  - [x] schema 增加 `system_prompt_template` / `user_prompt_template`（可选/默认空）
  - [x] 测试覆盖更新 payload 含模板字段

- [x] 2) Frontend：Manage Agents 增加模板字段（AC: 1）
  - [x] Agent 表单增加两个 textarea
  - [x] 保存/编辑/删除流程保持兼容

- [x] 3) 校验
  - [x] Backend：`ruff` + `pytest`
  - [x] Frontend：`npm run lint` + `npm run build`

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.5）
- `_bmad-output/architecture.md`（Prompt Templates in Agents）

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

- 为 AgentDefinition 增加 `system_prompt_template` / `user_prompt_template` 字段并持久化到后端 `agentsJson`。
- Manage Agents 表单支持编辑并保存 Prompt Templates（支持 `{{placeholders}}`，本 Story 仅存储）。
- 手工验证通过：模板可保存并在刷新后从后端 agentsJson 还原。

### File List

- `_bmad-output/implementation-artifacts/3-5-configure-agent-prompt-templates.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/tests/test_packages.py`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
