# Story BAI-4.1: Create AI Workbench Route and Shell

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Builder User**,  
I want a dedicated AI workbench route,  
So that all AI operations happen in one consistent page.

## Acceptance Criteria

### AC-1: Workbench Shell 可见
- **Given** entry query params (`targetType/targetId/mode`)
- **When** opening workbench
- **Then** page shows resolved target header and mode badge

## Design

### Summary
- 新增 `/builder/[projectId]/ai-workbench` 路由与基础 Shell 组件，作为统一 AI Workbench 容器。
- 通过 query params 解析 `targetType/targetId/mode`，渲染目标 header 与模式 badge。
- 参数非法时展示清晰的错误状态与返回入口（回到 Builder/Project）。
- Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

### UX / UI
- 顶部延续 Builder 现有视觉（logo + 标题 + 返回/登出按钮），保持一致性。
- Header 区展示：
  - 目标类型（Workflow/Step/Agent/Asset）
  - 目标标识（targetId 或 fallback 文案）
  - Mode badge：`create` 或 `optimize`
- 主体区域仅渲染空状态占位（后续接入 Conversation/Preview Pane）。
- 错误状态：使用醒目提示卡，说明缺失/非法参数，并提供返回按钮（回到 `/builder/{projectId}`）。

### API / Contracts
- N/A（仅前端路由 + query params 解析）。
- Query params 约定：
  - `targetType`: `workflow|step|agent|asset`
  - `targetId`: string（非空）
  - `mode`: `create|optimize`

### Data / Storage
- N/A（不引入持久化或状态存储）。

### Errors / Edge Cases
- 缺少 `targetType/targetId/mode` → 展示错误卡与返回按钮。
- `targetType`/`mode` 不在允许值集合 → 展示错误卡与返回按钮。
- `targetId` 为空或全空白 → 视为非法参数处理。

### Test Plan
- 访问带完整 params 的路由，header 显示目标类型与 targetId，mode badge 正确。
- 传入非法 `targetType` 或 `mode` 时显示错误卡且可返回 Builder。
- 缺少任一参数时显示错误状态，不崩溃。

## Technical Scope

- 新增 Workbench 路由页：
  - `crewagent-builder-frontend/src/app/builder/[projectId]/ai-workbench/page.tsx`
- 新增基础 Shell 组件：
  - `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`
- 解析 query params：
  - `targetType` / `targetId` / `mode`
- 展示基础 UI：
  - 目标对象标题
  - mode badge（create/optimize）
  - 空状态占位区（后续对话/预览面板接入）

## Tasks / Subtasks

- [x] Task 1: 建立 Workbench 路由与基础布局（AC: 1）
- [x] Task 2: 解析 query params 并渲染 header（AC: 1）
- [x] Task 3: 空状态与错误状态占位（AC: 1）

## Test Plan

- 访问 `/builder/{projectId}/ai-workbench?targetType=workflow&targetId=...&mode=create` 能看到 header 与 badge
- 缺失或非法 params 显示可理解的错误提示

## File List

- `_bmad-output/implementation-artifacts/bai-4-1-create-ai-workbench-route-shell.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/ai-workbench/page.tsx`
- `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`

## Dev Agent Record

### Agent Model Used

GPT-5 (Codex CLI)

### Debug Log References

- Not run (not requested).

### Completion Notes List

- 新增 AI Workbench 路由页，解析 query params 并渲染目标 header 与 mode badge。
- 新增 Workbench Shell 组件，提供 header/空状态/错误状态占位区。

### File List

- `_bmad-output/implementation-artifacts/bai-4-1-create-ai-workbench-route-shell.md`
- `_bmad-output/implementation-artifacts/sprint-status-builder-ai.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/ai-workbench/page.tsx`
- `crewagent-builder-frontend/src/components/ai/workbench-shell.tsx`

## Change Log

- Frontend：新增 AI Workbench 路由与基础 Shell，支持 query params 解析与空/错误状态展示。
