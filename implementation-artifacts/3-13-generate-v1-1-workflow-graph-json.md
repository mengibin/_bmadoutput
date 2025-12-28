# Story 3.13: Generate v1.1 `workflow.graph.json`

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Builder to generate `workflow.graph.json` following **Package Spec v1.1**,  
so that Runtime can load/validate the workflow structure deterministically.

## Acceptance Criteria

1. **Given** the Builder has a saved graph (nodes/edges)  
   **When** the Builder generates `workflow.graph.json`  
   **Then** it includes `schemaVersion/entryNodeId/nodes/edges`（v1.1 shape）  
   **And** each node has `id/type/file` where `file` points to the step file in the package（路径以 zip root 为基准；single-workflow: `steps/<nodeId>.md`；multi-workflow: `workflows/<workflowId>/steps/<nodeId>.md`）  
   **And** each edge has `from/to/label` (and sets `isDefault: true` when a node has multiple outgoing edges)

## Design

### Summary

 - Tech Spec: `_bmad-output/tech-spec.md`
 - 从 Builder 保存的 ReactFlow graph（nodes/edges）派生生成 v1.1 `workflow.graph.json`（对齐 `workflow-graph.schema.json`），作为 Runtime 的权威图真源。
 - nodes 映射：`id=node.id`（稳定 nodeId）+ `type=node.type` + `file=steps/<nodeId>.md`，并可携带 `title/agentId/inputs/outputs`（来自 node.data）。
 - edges 映射：`from/to/label`（优先用户填写，否则按确定性默认规则生成）+ `isDefault`（多出边时保证唯一 default）+ 可选 `conditionText`。
 - 在 WorkflowEditor 增加 `workflow.graph.json (preview)`（便于验收）；写入 ZIP 结构在 Story 3.16 处理。

### UX / UI

- 在 WorkflowEditor 的 Debug Previews（折叠区域）新增 `workflow.graph.json (preview)`：
  - 展示 pretty JSON（或 schema 校验错误列表）。
  - 若检测到多起点/孤岛/环形依赖，给出 warning/error 提示（不影响画布编辑）。
- entryNodeId 选择策略：
  - 优先：入度为 0 的节点且仅 1 个 → 作为 entry。
  - 否则：从所有入度为 0 的节点中按 `nodeId` 字典序选择最小的作为 entry，并显示 warning（“多起点，已选择 entryNodeId=...”）。

### API / Contracts

- N/A（本 story 不新增后端接口；生成逻辑在 Builder 内完成）。
- Contract（必须满足，`additionalProperties: false`）：
  - 文件：`crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`
  - 关键字段：
    - `schemaVersion: "1.1"`
    - `entryNodeId: <nodeId>`
    - `nodes[]: [{id,type,file,...}]`（至少 1 个）
    - `edges[]: [{id?,from,to,label,isDefault?,conditionText?,conditionExpr?}]`

### Data / Storage

- 不新增 DB 字段；`workflow.graph.json` 为派生文件：
  - 输入：后端保存的 `graph_json`（ReactFlow nodes/edges）+ node.data（title/agentId/inputs/outputs）。
  - 输出：前端生成 JSON（用于 preview）；导出落盘（zip 内路径）在 Story 3.16 实现。
- node.file 规则：
  - single-workflow 兼容模式：`steps/<nodeId>.md`（与 Story 3.12 的 step files key 保持一致）
  - 多 workflow 推荐模式：导出时写为 `workflows/<workflowId>/steps/<nodeId>.md`（以 zip root 为基准，便于 Runtime 直接 `@pkg/<node.file>` 读取）

### Errors / Edge Cases

- 空图：`nodes.length == 0` → 生成失败（schema 要求 nodes 至少 1 个）。
- nodeId 不合法（不匹配 `^[A-Za-z0-9][A-Za-z0-9._:-]*$`）→ 阻断生成并提示。
- edge 引用不存在的 node（source/target 不在 nodes 中）→ 阻断生成并提示。
- 环形依赖：阻断生成并提示（避免 Runtime 无法推进）。
- edge.label 为空：按确定性规则补齐，保证 schema `label` 非空：
  - 单出边：`next`
  - 多出边：`branch-1..n`（按 edge.id 排序后编号）
- default edge：
  - 同一 `from` 存在多条出边时，保证仅 1 条 `isDefault: true`；
  - 若用户未选择 default，则按 edge.id 排序后的第一条设为 default。

### Test Plan

- Unit：对映射函数做覆盖（线性/分叉/多起点/缺 label/缺 default/无效 nodeId/断边/环）。
- Integration：在 WorkflowEditor 打开 preview，AJV 校验通过（与 schema 文件一致）。
- Regression：确认 `file: steps/<nodeId>.md` 与实际 `stepFilesJson` key 一致且稳定（改连线不改文件名）。

## Tasks / Subtasks

- [x] 1) 实现 graph→v1.1 workflow.graph.json 的映射函数
- [x] 2) 为 edge 生成 label/isDefault 的确定性策略
- [x] 3) 在 Builder UI 增加 `workflow.graph.json (preview)`（便于验收）

## References

- `_bmad-output/epics.md`（Epic 3 / Story 3.13）
- Tech Spec: `_bmad-output/tech-spec.md`（Package Spec v1.1）
- `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/workflow-graph.schema.json`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- 新增 v1.1 `workflow.graph.json` 生成：从 ReactFlow graph（nodes/edges）映射到 schema 结构，补齐 `entryNodeId/nodes/edges`，并为多出边提供确定性的 `label/isDefault` 默认规则。
- WorkflowEditor 增加 `workflow.graph.json (preview)` + AJV schema 校验错误展示，便于验收与排错。
- 整包导出（写入 ZIP 文件结构）在 Story 3.16 统一处理（本 story 只负责生成与预览）。

### File List

- `_bmad-output/implementation-artifacts/3-13-generate-v1-1-workflow-graph-json.md`
- `crewagent-builder-frontend/src/app/editor/[projectId]/[workflowId]/page.tsx`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/workflow-graph.schema.json`
- `crewagent-builder-frontend/src/lib/workflow-graph-v11.ts`

### Change Log

- Builder 生成并预览 v1.1 `workflow.graph.json`（含 schema 校验），供后续整包导出（Story 3.16）写入 ZIP。
- 修复 AJV 对 draft-2020-12 `$schema` 的运行时报错：改用 `ajv/dist/2020` 编译 v1.1 schemas。
