# Tech-Spec: ProjectBuilder（多 Workflow + 多 Agent + 多节点类型编辑器）

**Created:** 2025-12-23  
**Status:** Ready for Development
**Covers:** Stories 3.8–3.11

## Overview

### Problem Statement

当前 Builder 的编辑体验以单个 `Workflow Editor` 为中心：

- 一个“项目/包”只能编辑 **单个 workflow**（无 workflows 列表）
- 只能拖拽 **Step** 一种节点类型（与 v1.1 `step/decision/merge/end/subworkflow` 不匹配）
- Agents 管理与 workflow 编辑耦合在同一页，且缺少 Project 级“多 workflow 共享 agents”的视图
- 页面宽度使用 `max-w-*` 容器，不符合“全屏 ProjectBuilder”的目标

这会直接阻碍后续 v1.1 导出（Story 3.12–3.17）：无法在 Builder 内形成稳定的 project/workflow/node 数据模型与编辑能力。

### Solution

实现 **ProjectBuilder**（全屏）作为新的入口：

- **Project 级视图**：同屏管理 `workflows[]` + `agents[]`（可创建/选择）
- **Workflow 级视图**：进入某 workflow 的编辑器：
  - Palette 支持多 node types（`step/decision/merge/end/subworkflow`）
  - Node 配置（title/instructions/agent assignment/分支 label 等）
  - Node 拥有稳定 `nodeId`（不依赖拓扑排序生成 `step-01.md` 等）

### Scope (In/Out)

**In Scope（Stories 3.8–3.11）**

- 前端：ProjectBuilder 页面（全屏）+ Workflow Editor 路由拆分
- 后端：支持“一个 project 多个 workflows + 共享 agents”的最小 CRUD
- 前端：Palette 多节点类型 + Node settings（最小可用）
- 数据迁移：将现有单 workflow 数据迁移为 project+workflow 结构（保持用户数据可用）

**Out of Scope（后续故事 3.12–3.17）**

- v1.1 文件生成：`workflow.md` / `steps/<nodeId>.md`（Story 3.12）
- v1.1 `workflow.graph.json` / `bmad.json` / `agents.json` 生成（Story 3.13–3.15）
- v1.1 ZIP 导出与 schema 校验/错误展示（Story 3.16–3.17）
- Runtime 导入与执行（Epic 4）

## Context for Development

### Codebase Patterns

**Frontend**

- Next.js App Router + Client Components
- API 调用统一走 `crewagent-builder-frontend/src/lib/api-client.ts`
- 鉴权通过 `useRequireAuth()`，token 存在 local storage（见 `crewagent-builder-frontend/src/lib/auth.ts`）
- React Flow 用于画布与节点编辑（见 `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`）

**Backend**

- FastAPI + SQLAlchemy + Alembic
- 统一响应格式 `{ data, error }`
- 路由：`crewagent-builder-backend/app/routers/packages.py`（现有单 workflow 版本）
- 模型：`crewagent-builder-backend/app/models/workflow_package.py`

### Files to Reference

- `crewagent-builder-frontend/src/app/dashboard/page.tsx`（创建项目/列表入口）
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`（现 Workflow Editor：ReactFlow + Agents modal + 导出）
- `crewagent-builder-backend/app/models/workflow_package.py`（现 DB 结构）
- `crewagent-builder-backend/app/routers/packages.py`（现 API）
- `_bmad-output/implementation-artifacts/3-8-project-builder-shell-full-screen-and-navigation.md`（Story 3.8）
- `_bmad-output/implementation-artifacts/3-9-project-builder-multi-workflow-management.md`（Story 3.9）
- `_bmad-output/implementation-artifacts/3-10-project-builder-agent-management-v1-1-required-fields.md`（Story 3.10）
- `_bmad-output/implementation-artifacts/3-11-workflow-editor-node-types-edge-labels-default-branch.md`（Story 3.11）

### Technical Decisions

1) **Project vs Workflow 数据模型**

为支持“多 workflow + 共享 agents”，建议引入明确的 `Project` 与 `Workflow`：

- Project：归属用户、名称、共享 `agents_json`
- Workflow：属于某 Project，包含 `graph_json/workflow_md/step_files_json`

2) **API 兼容策略**

- 新增 `/projects` 与 `/projects/{projectId}/workflows` API（推荐），并逐步让前端从 `/packages` 迁移
- 或复用 `/packages` 但语义改为 project（不推荐，长期会混淆）

3) **nodeId 生成规则（稳定）**

- 新增 node 时即确定 `nodeId`，后续不因拓扑顺序变化而重编号
- 推荐：按类型前缀 + 自增序号（在 workflow 内唯一）
  - `step-1`, `decide-1`, `merge-1`, `end-1`, `subworkflow-1`
- nodeId 与未来导出的 `steps/<nodeId>.md` 直接对应（Story 3.12）

## Implementation Plan

### Tasks

- [ ] Task 1: Backend 数据模型升级（Project + Workflow）
  - [ ] 新增表 `projects`（或复用 `workflow_packages` 作为 project 表）
  - [ ] 新增表 `project_workflows`（每个 workflow 一行）
  - [ ] Alembic migration：创建新表 + 外键 + index

- [ ] Task 2: Backend API（ProjectBuilder 最小 CRUD）
  - [ ] `GET /projects`：列出项目
  - [ ] `POST /projects`：创建项目（可默认创建一个 workflow）
  - [ ] `GET /projects/{id}`：项目详情（含 workflows 列表 + agentsJson）
  - [ ] `PUT /projects/{id}/agents`：更新共享 agents
  - [ ] `POST /projects/{id}/workflows`：创建 workflow
  - [ ] `GET /projects/{id}/workflows/{workflowId}`：workflow 详情（graph/workflowMd/stepFiles）
  - [ ] `PUT /projects/{id}/workflows/{workflowId}/content`：保存 workflow 内容

- [ ] Task 3: 数据迁移（从单 workflow 到多 workflow）
  - [ ] 对每条现有 `workflow_packages`：
    - [ ] 将其视为一个 Project（保留 id 或映射新 project id）
    - [ ] 创建一个默认 Workflow，并把旧的 `graph_json/workflow_md/step_files_json` 迁移进去
  - [ ] 迁移后旧字段保留一段时间（或迁移后置空并仅用于兼容读取）

- [ ] Task 4: Frontend 路由与页面重构
  - [ ] 新增 `ProjectBuilder` 页面（全屏宽度）
  - [ ] 新增 `WorkflowEditor` 页面（以 projectId + workflowId 作为路由参数）
  - [ ] Dashboard：创建项目后跳转到 ProjectBuilder；列表“打开”链接更新

- [ ] Task 5: ProjectBuilder UI（workflows + agents）
  - [ ] Workflows panel：列表 + Create Workflow（modal/form）
  - [ ] Agents panel：列表 + Create/Edit Agent（modal/form）
  - [ ] 点击某 workflow 进入 WorkflowEditor

- [ ] Task 6: WorkflowEditor 多节点类型与配置
  - [ ] Palette：展示 `step/decision/merge/end/subworkflow`
  - [ ] 画布拖拽创建：按 node type 生成默认 node data + 稳定 nodeId
  - [ ] Node settings：支持按类型渲染表单（至少 title/instructions/agentId；decision 支持 edge label 编辑的最小版本）

- [ ] Task 7: 校验与回归
  - [ ] Backend：pytest 覆盖（owner/权限/404/400）
  - [ ] Frontend：手工冒烟（创建/切换 workflow、创建 agent、拖拽 node、保存/刷新）

### Acceptance Criteria

- [ ] AC1：ProjectBuilder 全屏布局，显示 workflows 列表 + agents 列表
- [ ] AC2：可创建 workflow，并能点击进入该 workflow 编辑器
- [ ] AC3：可创建 agent，并能在任意 workflow 的 node 中选择该 agent
- [ ] AC4：WorkflowEditor palette 至少包含 `step/decision/merge/end/subworkflow`，可拖入画布
- [ ] AC5：node 可打开 settings 并编辑关键字段；nodeId 稳定（不随连线/排序变化而变）

## Additional Context

### Dependencies

- Frontend：继续使用现有依赖 `reactflow`；无需新增依赖即可先实现多 node types
- Backend：延续 SQLAlchemy/Alembic 迁移方式

### Testing Strategy

- Backend：`pytest` 覆盖新路由（auth/owner scope/数据一致性）
- Frontend：手工冒烟优先（当前无前端测试框架）

### Notes

- Story 3.8–3.11 的目标是 **把“Project/Workflow/Node/Agent”的编辑与存储模型做正确**，为 Story 3.12–3.17 的 v1.1 生成与导出奠定基础。
- 如果你希望先降低风险，可先实现 “ProjectBuilder UI + 单 workflow 兼容模式”，再逐步启用多 workflow；但最终数据模型仍需支持多 workflow。  
