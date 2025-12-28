# Story 3.8: ProjectBuilder Shell (Full-Screen + Navigation)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want a full-screen **ProjectBuilder** shell that shows my project’s workflows and agents and lets me navigate into a workflow editor,  
so that I can manage my project at a glance before doing detailed editing.

## Acceptance Criteria

1. **Given** I open the Builder for a project  
   **When** the page loads  
   **Then** I see a **ProjectBuilder** view in full-width layout (no max-width container) with:  
   - a Workflows panel (list + open)  
   - an Agents panel (list)

2. **Given** the project has workflows  
   **When** I click a workflow in the list  
   **Then** I enter that workflow’s editor view and can see its canvas

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- 将现有单页 `Workflow Editor` 拆成两层路由：ProjectBuilder（项目级壳）→ WorkflowEditor（工作流级画布）。
- 本 Story 仅交付“全屏布局 + workflows/agents 可见 + 可进入编辑器”，不引入 multi-workflow 创建（见 Story 3.9）。
- 先基于现有 `/packages/{id}` 读取数据：把 legacy package 映射成“单 workflow 列表项”，保证可渐进迁移。

### UX / UI

- 路由与入口：
  - Dashboard 点击某项目后进入 ProjectBuilder（项目级视图）
  - ProjectBuilder 点击某 workflow 后进入 WorkflowEditor（画布页）
- 布局（全屏）：
  - 页面主容器使用 `w-full max-w-none`（避免 `max-w-*` 约束）
  - 左侧：Workflows 面板（列表 + Open）
  - 右侧：Agents 面板（列表）
  - 中间：选中 workflow 的摘要区（名称/说明）+ “Open Editor” 主按钮（可选）
- 交互（本 Story 范围）：
  - Workflows 列表至少展示 1 个条目（legacy 默认 workflow）
  - 点击条目或 Open 按钮 → 跳转到现有 WorkflowEditor 画布页并可见 ReactFlow 画布
  - Agents 面板展示 agents 列表；创建/编辑入口可先隐藏或置灰（在 Story 3.10 实现）

### API / Contracts

- 本 Story 不新增后端接口，复用现有 `packages`：
  - `GET /packages/{packageId}` → 返回 `{ id, name, workflowMd, agentsJson, graphJson, stepFilesJson }`
- 前端映射规则（MVP）：
  - `workflows[]`：固定映射为 1 个条目（legacy 单 workflow）：
    - `workflowId = packageId`（沿用现有 editor 路由参数）
    - `name = package.name`（或 `Default Workflow`）
  - `agents[]`：从 `agentsJson` 解析得到（仅用于列表展示；结构升级在 Story 3.10）

### Data / Storage

- 不引入新表/新字段；仅新增前端视图模型（内存态）用于渲染 ProjectBuilder。
- 数据来源：
  - package detail：后端 `workflow_packages`（`agents_json/graph_json/...`）
  - 后续在 Story 3.9 迁移为 `project + workflows` 结构时，ProjectBuilder 只需替换数据加载层，UI 结构保持。

### Errors / Edge Cases

- 401/未登录：按现有鉴权逻辑跳转登录（或显示“需要登录”提示）。
- 404 项目不存在：显示空态（“项目不存在/已被删除”）+ 返回 Dashboard。
- `agentsJson` 解析失败：阻断渲染 agents 列表并提示“agents 数据损坏”，同时提供重试/返回入口。
- agents 为空数组：Agents 面板显示空态（“暂无 Agents”）。
- workflows 列表为空（理论上不会出现，但需防御）：显示空态并提示在 Story 3.9 支持创建 workflow。

### Test Plan

- 手工冒烟：
  - 从 Dashboard 打开任一项目 → 进入 ProjectBuilder，确认页面为全屏宽度（无 max-width 限制）。
  - Workflows/Agents 面板可见；workflows 至少有 1 条（legacy 映射）。
  - 点击 workflow 条目 → 跳转到 WorkflowEditor，ReactFlow 画布可见且不报错。
  - 人为构造 404（改 URL id）→ 显示“项目不存在”空态并可返回。
  - 人为构造 agentsJson 非法（后端返回脏数据）→ agents 列表失败时有可理解错误提示。

## Tasks / Subtasks

- [x] 1) 前端：新增 ProjectBuilder 页面（全屏布局）
- [x] 2) 前端：Workflows/Agents 列表展示（读取后端数据）
- [x] 3) 前端：workflow 选择与导航到 workflow editor
- [x] 4) 回归：现有项目仍可打开并进入编辑器

### Review Follow-ups (AI)

- [x] [AI-Review][MEDIUM] 为 ProjectBuilder 的 data fetch 增加取消/忽略机制（避免 unmount 后 setState 警告），对齐 editor 页面的取消模式（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:62`）
- [x] [AI-Review][MEDIUM] Loading/Error 时仍保持三栏布局可见（Workflows/Overview/Agents），将“加载中/错误”放进面板内部，避免 AC1 在首屏阶段表现为“只有一段文字”（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:181`）
- [x] [AI-Review][MEDIUM] `GET /packages/{id}` 失败时提供显式“重试”入口（按钮），并区分 404/422 的用户提示（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:172`）
- [x] [AI-Review][MEDIUM] 统一语言与命名：Workflows 列表中的 `Default Workflow` / `Open` 等英文文案与中文 UI 不一致（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:106`）
- [x] [AI-Review][LOW] Dashboard 弹窗仍混用 Workflow/Project 文案（如 placeholder `My Workflow`、按钮 `Close/Cancel`），与 “New Project” 不一致（`crewagent-builder-frontend/src/app/dashboard/page.tsx:226`）
- [x] [AI-Review][LOW] Editor 顶部“返回 Dashboard”不利于在 ProjectBuilder 与 Editor 间来回（可考虑增加“返回 ProjectBuilder”，或在 Story 3.9 路由升级后补齐）（`crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx:980`）
- [x] [AI-Review][LOW] 当前各子仓库存在大量未提交/未追踪变更，Story 3.8 的 File List 无法与 git 变更一一对应；建议按 story 拆分提交或补齐 File List 以便可审计（`_bmad-output/implementation-artifacts/3-8-project-builder-shell-full-screen-and-navigation.md:114`）

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-project-builder.md`
- `_bmad-output/epics.md`（Epic 3 / Story 3.8）
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`（现有 Workflow Editor）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- 新增 `ProjectBuilder` 页面：`/builder/[projectId]`（全屏布局），加载 `GET /packages/{id}` 并展示 Workflows/Agents 列表。
- 点击 workflow 条目可进入现有 `WorkflowEditor`：`/editor/[workflowId]` 并看到画布。
- Dashboard 的创建/打开项目入口改为进入 `ProjectBuilder`（为后续 Story 3.9–3.11 预留扩展空间）。
- QA 修复：保持三栏布局在 loading/error 可见、错误提示区分 404/参数错误并提供重试、统一关键文案、Editor 增加返回 ProjectBuilder。

### File List

- `_bmad-output/implementation-artifacts/3-8-project-builder-shell-full-screen-and-navigation.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/.env.local.example`
- `crewagent-builder-frontend/package-lock.json`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/src/app/globals.css`
- `crewagent-builder-frontend/src/app/page.tsx`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/page.tsx`
- `crewagent-builder-frontend/src/app/login/page.tsx`
- `crewagent-builder-frontend/src/lib/api-client.ts`
- `crewagent-builder-frontend/src/lib/auth.ts`
- `crewagent-builder-frontend/src/lib/use-require-auth.ts`

### Change Log

- 新增 ProjectBuilder 路由与全屏壳；Dashboard 导航改为先进入 ProjectBuilder 再进入 editor。
- QA 修复：loading/error 仍展示三栏、提供重试与更清晰错误提示、统一关键文案、Editor 增加返回 ProjectBuilder。

## Senior Developer Review (AI)

### Review Outcome

- Date: 2025-12-24
- Outcome: Approved
- Issues: 0
- Acceptance Criteria: PASS

### Action Items

- [x] [AI-Review][MEDIUM] 为 ProjectBuilder 的 data fetch 增加取消/忽略机制（避免 unmount 后 setState 警告），对齐 editor 页面的取消模式（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:62`）
- [x] [AI-Review][MEDIUM] Loading/Error 时仍保持三栏布局可见（Workflows/Overview/Agents），将“加载中/错误”放进面板内部，避免 AC1 在首屏阶段表现为“只有一段文字”（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:181`）
- [x] [AI-Review][MEDIUM] `GET /packages/{id}` 失败时提供显式“重试”入口（按钮），并区分 404/422 的用户提示（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:172`）
- [x] [AI-Review][MEDIUM] 统一语言与命名：Workflows 列表中的 `Default Workflow` / `Open` 等英文文案与中文 UI 不一致（`crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx:106`）
- [x] [AI-Review][LOW] Dashboard 弹窗仍混用 Workflow/Project 文案（如 placeholder `My Workflow`、按钮 `Close/Cancel`），与 “New Project” 不一致（`crewagent-builder-frontend/src/app/dashboard/page.tsx:226`）
- [x] [AI-Review][LOW] Editor 顶部“返回 Dashboard”不利于在 ProjectBuilder 与 Editor 间来回（可考虑增加“返回 ProjectBuilder”，或在 Story 3.9 路由升级后补齐）（`crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx:980`）
- [x] [AI-Review][LOW] 当前各子仓库存在大量未提交/未追踪变更，Story 3.8 的 File List 无法与 git 变更一一对应；建议按 story 拆分提交或补齐 File List 以便可审计（`_bmad-output/implementation-artifacts/3-8-project-builder-shell-full-screen-and-navigation.md:114`）
