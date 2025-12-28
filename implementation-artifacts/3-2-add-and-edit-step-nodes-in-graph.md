# Story 3.2: Add and Edit Step Nodes in Graph

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to add and edit Step nodes on the React Flow canvas,  
so that I can visually define workflow steps.

## Acceptance Criteria

1. **Given** I am in the Workflow Editor  
   **When** I drag a "Step" from the palette onto the canvas  
   **Then** a new Step node appears with default properties  
   **And** I can double-click to edit the step's name, instructions, and agent assignment

## Design

### Summary

- Editor 页面（`/editor/[workflowId]`）引入 React Flow 画布。
- 左侧提供 Palette（至少一个可拖拽项：Step）。
- 拖拽到画布后创建 Step Node（默认 name/instructions/agent）。
- 双击节点打开编辑弹窗：支持修改 name / instructions / agent（MVP：agent 用字符串占位）。

### UX / UI

- 左侧：Palette（可拖拽 Step）
- 中间：Canvas（React Flow）
- 弹窗：Edit Step
  - Step Name（必填）
  - Instructions（可多行）
  - Agent（占位，后续 Story 3.4/3.5 接入真实 Agent 列表）
  - Save / Cancel

### Data / Storage

- 本 Story 先以内存状态保存 graph（不落库）；持久化与 Markdown 生成在 Story 3.6 实现。

### Errors / Edge Cases

- 未登录访问 Editor：仍应重定向 `/login`（已由 auth guard 处理）。
- 缺少 `NEXT_PUBLIC_API_BASE_URL`：显示明确提示（已实现）。

### Test Plan

- 手工冒烟（前端）：
  - 拖拽 Step 到画布 → 出现节点
  - 双击节点 → 打开编辑弹窗 → 修改并保存 → 节点展示更新
- 工程校验：
  - Frontend：`npm run lint`、`npm run build`

## Tasks / Subtasks

- [x] 1) 引入 React Flow（AC: 1）
  - [x] 安装依赖并引入样式
  - [x] Editor 页面渲染画布与基础节点

- [x] 2) Palette 拖拽创建 Step Node（AC: 1）
  - [x] Palette draggable item（Step）
  - [x] Canvas drop handler：创建节点并定位

- [x] 3) 双击节点编辑（AC: 1）
  - [x] 编辑弹窗（name/instructions/agent）
  - [x] 保存后更新节点 data

- [x] 4) 校验
  - [x] `npm run lint`
  - [x] `npm run build`

## Dev Notes

- 画布选型：React Flow（`reactflow`）。
- 本 Story 不做边（edge）连接（Story 3.3 再实现）。

### Project Structure Notes

- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/app/globals.css`

### References

- `_bmad-output/architecture.md`（Editor 路由 & React Flow 约定）
- `_bmad-output/epics.md`（Epic 3 / Story 3.2）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- Editor 页面引入 React Flow 画布与 Step 节点渲染（本 Story 不做 edge）。
- 支持从左侧 Palette 拖拽 Step 到画布创建节点（默认 name/instructions/agent）。
- 支持双击节点打开弹窗编辑并保存回写到节点 data。
- 手工验证通过：拖拽创建节点、双击编辑保存均正常。

### File List

- `_bmad-output/implementation-artifacts/3-2-add-and-edit-step-nodes-in-graph.md`
- `crewagent-builder-frontend/package-lock.json`
- `crewagent-builder-frontend/package.json`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/app/globals.css`
