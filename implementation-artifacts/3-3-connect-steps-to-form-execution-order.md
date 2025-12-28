# Story 3.3: Connect Steps to Form Execution Order

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want to connect Steps with edges to define execution order,  
so that the workflow has a clear execution path.

## Acceptance Criteria

1. **Given** multiple Step nodes exist on the canvas  
   **When** I drag from one node's output port to another's input port  
   **Then** an edge (connection) is created between them  
   **And** the execution order is reflected in the generated Markdown (preview)

## Design

### Summary

- 在 Editor（`/editor/[workflowId]`）开启节点连线能力：
  - source → target 创建 edge（React Flow `onConnect` + `addEdge`）。
- 基于 edges 计算 “执行顺序”（MVP：拓扑排序；若存在环或分叉则提示）。
- 提供 `workflow.md (preview)`：根据排序结果生成 Steps 列表（不落库，Story 3.6 再持久化/写入文件）。

### UX / UI

- Canvas：
  - 从节点底部 handle 拖到另一个节点顶部 handle → 自动出现连线。
- 侧边栏新增：
  - Execution Order：显示步骤顺序（按排序后的节点 name）。
  - workflow.md (preview)：展示基于 graph 的 Markdown（前端即时生成）。

### Errors / Edge Cases

- 无边：按节点创建顺序作为默认预览顺序，并提示“可通过连线定义顺序”。
- 存在环：显示错误提示“检测到循环依赖”，并在预览区域停止生成。
- 存在分叉（一个节点多个后继或多个起点）：提示“顺序可能不唯一”，预览按拓扑排序的一种结果生成（MVP）。

### Test Plan

- 手工冒烟：
  - 创建 2 个 Step 节点 → 连线 step-1 → step-2 → 显示 edge
  - Execution Order 与 `workflow.md (preview)` 反映连线顺序
  - 制造环（A→B，B→A）→ 显示循环错误提示
- 工程校验：
  - Frontend：`npm run lint`、`npm run build`

## Tasks / Subtasks

- [x] 1) 开启连线并保存 edges（AC: 1）
  - [x] 实现 `onConnect`（`addEdge`）
  - [x] 启用 connect（默认开启；移除 `nodesConnectable={false}`）

- [x] 2) 计算执行顺序并展示（AC: 1）
  - [x] 基于 edges 的拓扑排序
  - [x] 处理循环/分叉提示

- [x] 3) workflow.md 预览生成（AC: 1）
  - [x] 根据排序结果生成 `## Steps` 列表（`- step-01.md`…）

- [x] 4) 校验
  - [x] `npm run lint`
  - [x] `npm run build`

## Dev Notes

- 本 Story 只做前端内存态与预览；持久化与文件生成在 Story 3.6。

### References

- `_bmad-output/epics.md`（Epic 3 / Story 3.3）
- `_bmad-output/architecture.md`（Builder: React Flow + Markdown Generator）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- Editor 画布支持 Step 节点连线（source → target 创建 edge）。
- 基于 edges 做拓扑排序并在侧边栏展示 Execution Order（循环/分叉提示）。
- 增加 `workflow.md (preview)`：按执行顺序生成 `## Steps` 列表（本 Story 不落库）。
- 手工验证通过：连线可创建 edge；顺序与 preview 能更新；循环依赖有提示。

### File List

- `_bmad-output/implementation-artifacts/3-3-connect-steps-to-form-execution-order.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/src/app/editor/[workflowId]/page.tsx`
