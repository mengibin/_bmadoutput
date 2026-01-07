# Story 4.2: Load Workflow Graph and Step Files

Status: done

## Story

As a **Runtime System**,
I want to **load and parse the `workflow.graph.json` and referenced step files** for a specific workflow,
so that I have all necessary data in memory to start an execution session.

## Acceptance Criteria

1. **Given** a loaded package and a target `workflowId`
   **When** `RuntimeStore.loadWorkflow(packageId, workflowId)` is called
   **Then** the system locates the `workflow.graph.json` corresponding to that workflow ID.
   **And** parses the JSON content.

2. **Given** the parsed graph JSON
   **When** validating the structure
   **Then** it MUST verify:
   - `schemaVersion` is supported (v1.1).
   - `entryNodeId` exists in the `nodes` list.
   - All `nodes` have `id`, `type`, and `file` properties.
   - All `edges` connect valid `from`/`to` node IDs.

3. **Given** a valid graph structure
   **When** loading step content
   **Then** the system iterates through all `nodes` (regardless of type).
   **And** resolves the `file` path relative to the package root.
   **And** reads the content of each `.md` file.
   **And** fails with a clear error if any file is missing or unreadable.

4. **Given** successful loading
   **When** returning the result
   **Then** the method returns a `WorkflowDefinition` object containing:
   - The original graph structure.
   - A map of `nodeId` -> `StepContent` (raw markdown string).

## Technical Context

- **Schema**: `workflow-graph.schema.json` (v1.1) defines the structure.
- **Path Resolution**: Step files are relative paths in the zip/package structure. `RuntimeStore` knows the package root.
- **Memory**: The loaded definition is ephemeral (per session). It doesn't need to be persisted to disk again, but loaded into the active `Run` state in memory.

## Design

### Summary
- `RuntimeStore.loadWorkflow(packageId, workflowId)` 异步定位并解析 `workflow.graph.json`，返回 `WorkflowDefinition`（`graph` + `steps` 映射）。
- 使用 `ajv`(2020) 校验 v1.1 schema，并做逻辑校验（`entryNodeId` 存在、`nodes/edges` 引用有效、`node.file` 必填）。
- 遍历所有 `nodes`，通过 `resolveInsideDir(packageRoot, node.file)` 解析路径并并行读取 `.md`，组装 `steps: Record<nodeId, string>`。
- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-2-load-workflow-graph-and-step-files.md

### UX / UI
- N/A（纯后端逻辑）

### API / Contracts
- IPC: `workflow:load`
  - 请求: `{ packageId: string, workflowId: string }`
  - 响应:
    - 成功: `{ success: true, definition: { graph: WorkflowGraph, steps: Record<nodeId, string> } }`
    - 失败: `{ success: false, error: { code: string, message: string, details?: unknown } }`
- 失败时返回结构化错误（`code/message/details`），用于 UI 提示与可编程处理。

### Data / Storage
- 读取 RuntimeStore 的包缓存目录（`packages/<packageId>/...`），不写磁盘。
- `WorkflowDefinition` 仅存内存（Run 会话级）。

### Errors / Edge Cases
- `workflow.graph.json` 缺失或 JSON 解析失败 → 报错并阻止启动。
- Schema 不支持/校验失败（含 `entryNodeId`、`nodes/edges` 引用无效）→ 返回明确错误详情。
- `node.file` 路径越界或文件缺失/不可读 → 报错 `"File not found: <path>"`。
- 任一步骤文件读取失败 → 整体失败（避免半加载）。

**Error Codes（示例）**：
- `PACKAGE_NOT_FOUND` / `WORKFLOW_NOT_FOUND`
- `GRAPH_PATH_INVALID` / `GRAPH_NOT_FOUND` / `GRAPH_JSON_INVALID`
- `GRAPH_SCHEMA_INVALID` / `GRAPH_INTEGRITY_INVALID`
- `STEP_FILE_UNSUPPORTED` / `STEP_FILE_PATH_INVALID` / `STEP_FILE_NOT_FOUND` / `STEP_FILE_READ_FAILED`

### Test Plan
- 单元测试：使用 `create-story-micro` fixture，验证 `entryNodeId` 与 `steps` map。
- 负向测试：缺失 graph、schema 不匹配、edge 指向不存在节点、step 文件缺失。
- 手动：通过 IPC 调用 `workflow:load`，返回 graph + steps。

## Tasks / Subtasks

- [x] Define `WorkflowDefinition` and `StepContent` interfaces in `RuntimeStore`.
- [x] Implement `loadWorkflow(packageId, workflowId)` in `RuntimeStore`.
- [x] Implement `validateGraphIntegrity` helper (nodes vs edges vs entry).
- [x] Implement `loadStepFiles` helper (parallel read of .md files).
- [x] Add error handling for "Graph not found", "Step file missing", "Invalid Schema".
- [x] Expose via IPC `workflow:load`.

## References

- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-2-load-workflow-graph-and-step-files.md
