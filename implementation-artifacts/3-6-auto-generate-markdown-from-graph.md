# Story 3.6: Auto-Generate Markdown from Graph

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the system to automatically generate Markdown files from my graph edits,  
so that I don't need to write Markdown manually.

## Acceptance Criteria

1. **Given** I have made changes to the workflow graph  
   **When** I save the project or navigate away  
   **Then** `workflow.md` is regenerated with YAML Frontmatter and step references  
   **And** step files (`step-01.md`, etc.) are created/updated based on nodes

## Design

### Summary

- 前端 Editor 维护 graph（nodes/edges）与 agents，并基于图自动生成：
  - `workflow.md`（含 `stepsCompleted` frontmatter 与 steps 列表）
  - step files（`step-01.md`…，从节点 name/instructions/agent 生成）
- 保存策略：
  - 增加后端 `PUT /packages/{id}/content`：保存 `workflow_md`、`graph_json`、`step_files_json`
  - Editor 采用 debounce auto-save：停止编辑一段时间后自动保存；同时提供保存状态（Saved/Saving/Failed）

### Data Model

- `workflow_packages` 新增列：
  - `graph_json`（React Flow nodes/edges JSON）
  - `step_files_json`（`{ "step-01.md": "...", ... }`）

### API / Contracts

- `PUT /packages/{id}/content`（auth required）
  - Request:
    - `workflow_md: string`
    - `graph: { nodes: any[], edges: any[] }`
    - `step_files: Record<string, string>`
  - Success (200): `{ "data": { id, name, workflowMd, agentsJson, graphJson, stepFilesJson }, "error": null }`
  - Failure: 400/401/404

### Errors / Edge Cases

- 循环依赖：阻止保存并提示“检测到循环依赖”。
- graph/step_files 反序列化失败：前端回退到默认初始图，同时提示。

### Test Plan

- 手工冒烟：
  - 画布新增/编辑节点、连线调整顺序 → `workflow.md (preview)` 与 step files 预览更新
  - 刷新页面 → graph 仍能从后端恢复；`workflow.md (stored)` 与 step files（stored）一致
- 工程校验：
  - Backend：`ruff` + `pytest`
  - Frontend：`npm run lint` + `npm run build`

## Tasks / Subtasks

- [x] 1) Backend：持久化 workflow/graph/steps（AC: 1）
  - [x] Alembic migration：新增 `graph_json` / `step_files_json`
  - [x] `PUT /packages/{id}/content`（auth + owner）
  - [x] pytest 覆盖：owner 可更新 / 非 owner 404

- [x] 2) Frontend：Graph→Markdown 生成与保存（AC: 1）
  - [x] 从后端加载 graph_json 还原 nodes/edges
  - [x] 生成 `workflow.md` 与 step files（preview）
  - [x] debounce auto-save + 保存状态展示

- [x] 3) 校验
  - [x] Backend：`ruff` + `pytest`
  - [x] Frontend：`npm run lint` + `npm run build`

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.6）
- `_bmad-output/architecture.md`（Graph-First Editing → Markdown generation）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `./.venv/bin/ruff check .`
  - `./.venv/bin/pytest -q`
  - `./.venv/bin/alembic upgrade head`
- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- 新增 `PUT /packages/{id}/content`：保存 `workflow_md`、`graph_json`、`step_files_json`。
- Editor 生成 `workflow.md (preview)` 与 `step-01.md (preview)`，并在停止编辑后 debounce 自动保存到后端。
- Editor 支持从后端恢复 graphJson，并展示 stored 内容用于对比。
- 手工验证通过：编辑/连线后可自动保存，刷新后 stored 内容可还原。

### File List

- `_bmad-output/implementation-artifacts/3-6-auto-generate-markdown-from-graph.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/models/workflow_package.py`
- `crewagent-builder-backend/app/routers/packages.py`
- `crewagent-builder-backend/app/schemas/workflow_package.py`
- `crewagent-builder-backend/app/services/package_service.py`
- `crewagent-builder-backend/migrations/versions/4c2ddf09c2c6_add_graph_and_step_files_to_workflow_.py`
- `crewagent-builder-backend/tests/test_packages.py`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
