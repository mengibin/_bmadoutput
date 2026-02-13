# Story BAI-4.5: Connect Four Entrypoints

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Builder User**,  
I want Workflow/Step/Agent/Asset to open the same AI workbench,  
So that interaction model is consistent.

## Acceptance Criteria

### AC-1: 四入口可达
- **Given** Workflow/Step/Agent/Asset 任意入口
- **When** 用户点击 AI 入口按钮
- **Then** 跳转到 Workbench 路由并带上 `targetType/targetId/mode`

### AC-2: 目标上下文正确
- **Given** Workbench 打开
- **When** 页面渲染 header
- **Then** targetType 与 targetId 匹配来源入口对象

### AC-3: 跳转不破坏原页面
- **Given** 用户从原页面进入 Workbench
- **When** 返回 Builder/Editor
- **Then** 原页面状态保持不变（不触发清空）

## Design

### UX / UI
- 入口按钮沿用现有 Builder/Editor 视觉体系：圆角按钮 + 细边框/主色强调。
- Workflow/Asset 入口放在 Builder 项目页右侧工具区；Step 入口放在工作流编辑器的节点卡片底部；Agent 入口放在 Agent 列表/详情页头部。
- 按钮文案统一为 “AI workbench”，必要时附简短说明（例如 “Open AI workbench”）。

### API / Contracts
- Workbench URL 统一格式：`/builder/{projectId}/ai-workbench?targetType={workflow|step|agent|asset}&targetId={id}&mode={create|optimize}`。
- `mode` 默认 `optimize`；若入口处为创建场景则传 `create`（例如新建 Agent/Asset 场景）。
- 可选 `returnTo`：入口页面传入当前路径，用于 Workbench 返回按钮回跳。

### Data / Storage
- 不新增持久化，仅依赖现有页面状态；跳转为前端路由导航。

### Errors / Edge Cases
- 入口缺失关键 id 时禁止跳转并提示（或不渲染按钮）。
- Workbench 参数非法时由 BAI-4.1 的错误态提示处理。
- 返回路径统一走浏览器 back 或页面内返回按钮，不主动清空原页面草稿状态。

### Test Plan
- Workflow/Step/Agent/Asset 四入口点击后能打开 Workbench，header 显示正确 targetType/targetId。
- 从 Workbench 返回后，原页面编辑态不丢失（手工回归）。

## Technical Scope

- 在四入口添加 AI Workbench 跳转：
  - Workflow（项目/工作流页面）
  - Step（工作流编辑器内 step/节点）
  - Agent（Agent 列表/编辑页）
  - Asset（Assets 面板）
- 统一构造 Workbench URL：
  - `/builder/{projectId}/ai-workbench?targetType=...&targetId=...&mode=...`
  - 可选追加 `returnTo` 以保留入口页面返回路径
- 入口 UI 维持现有风格，新增 AI 按钮/菜单项即可。

## Tasks / Subtasks

- [x] Task 1: 明确四入口位置与 targetId 来源（AC: 1,2）
- [x] Task 2: 新增 AI Workbench 跳转入口（AC: 1）
- [x] Task 3: 处理返回路径与状态保持（AC: 3）
- [x] Task 4: 补充基础回归测试或手工验证说明（AC: 1,2,3）

## Test Plan

- 四入口都能打开 Workbench，且 header 中 targetType/targetId 正确
- 返回 Builder/Editor 不触发状态丢失
- 缺失参数时显示错误状态（由 BAI-4.1 处理）

## Planned File List

- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/app/builder/[projectId]/agents/[agentId]/page.tsx`
- `crewagent-builder-frontend/src/app/builder/[projectId]/agents/new/page.tsx`
- `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`
- `crewagent-builder-frontend/src/lib/ai-workbench-client.ts`

## File List

- `_bmad-output/implementation-artifacts/bai-4-5-connect-four-entrypoints.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/components/builder/agent-editor-page.tsx`
- `crewagent-builder-frontend/src/lib/ai-workbench-client.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- Not run (not requested).

### Completion Notes List

- 增加 Workflow/Agent 列表与 Assets 面板的 AI Workbench 入口，统一用 helper 构造 URL。
- 在工作流编辑器顶部新增 Step 入口，缺少选中节点时禁用。
- Agent 详情页新增 AI Workbench 入口，新建模式根据名称生成候选 id 后启用。
- 新增 `ai-workbench-client` 统一构造 workbench URL。

### File List

- `_bmad-output/implementation-artifacts/bai-4-5-connect-four-entrypoints.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/components/builder/agent-editor-page.tsx`
- `crewagent-builder-frontend/src/lib/ai-workbench-client.ts`

## Change Log

- Frontend：新增四入口的 AI workbench 跳转按钮、补齐 Assets 列表入口（仅文件行显示且移除顶部入口）、将 Step 入口放到节点卡片底部，统一 URL 构造 helper，并支持 `returnTo` 回跳。
