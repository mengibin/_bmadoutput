# Story 4.3: Parse Frontmatter to Determine Current Node

Status: done

## Story

As a **Runtime Execution Engine**,
I want to **parse the Frontmatter of the active `workflow.md` file**,
so that I can determine the **Current Node ID** and the current state of execution (variables, history).

## Acceptance Criteria

1. **Given** a reference to the active `workflow.md` file (entry point)
   **When** the execution session initializes or resumes
   **Then** the system reads the file content.

2. **When** parsing the content
   **Then** it extracts the YAML Frontmatter (between `---` delimiters).
   **And** identifies the `currentNodeId` property.

3. **Given** a missing or empty `currentNodeId`
   **When** interpreting state
   **Then** it defaults to the `entryNodeId` defined in the Graph (loaded in Story 4.2).

4. **When** parsing is complete
   **Then** the `RuntimeStore` (or `Run` object) updates its state with:
   - `currentNodeId`
   - `variables` map (if present)
   - `stepsCompleted` list (if present)

## Technical Context

- **File**: The `workflow.md` file is the "living" document of the execution.
- **Library**: Use `gray-matter` or similar frontmatter parser (or regex if simple).
- **Fallback**: If `currentNodeId` is missing in `workflow.md` (e.g. fresh start), use Graph's entry node.

- [x] Verify unit tests for extracting `currentNodeId`.

## References

- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-3-parse-frontmatter-to-determine-current-node.md

## Design

### Summary
- `RuntimeStore.getWorkflowState(runId, graph)` 读取运行态 `@state/workflow.md` 并用 `gray-matter` 解析 frontmatter。
- 解析结果校验 `workflow-frontmatter.schema.json`（v1.1），并做逻辑校验（`currentNodeId` 存在于 graph）。
- `currentNodeId` 为空时使用 `graph.entryNodeId`；`variables/stepsCompleted/decisionLog/artifacts` 缺失时填默认值。
- Tech Spec: _bmad-output/implementation-artifacts/tech-spec-4-3-parse-frontmatter-to-determine-current-node.md

### UX / UI
- N/A（后端逻辑）

### API / Contracts
- IPC: `workflow:getState`
  - 请求: `{ packageId: string, workflowId: string, runId: string, projectRoot: string }`
  - 响应:
    - 成功: `{ success: true, state: WorkflowState }`
    - 失败: `{ success: false, error: { code: string, message: string, details?: unknown } }`
- 失败时返回结构化错误（`code/message/details`），用于 UI 阻断与修复提示。

### Data / Storage
- 只读运行态 `runs/<runId>/state/workflow.md`（优先），否则回退到 `state/workflow.md`。
- 不写回文件（写入/更新由后续 Story 4.7 负责）。

### Errors / Edge Cases
- `workflow.md` 缺失/不可读 → 失败并阻断恢复（提示重新初始化 run）。
- YAML 解析失败或 schema 校验失败 → 失败并提示修复 frontmatter。
- `currentNodeId` 为空 → fallback 到 `graph.entryNodeId`。
- `currentNodeId` 不在 graph 中 → 失败并提示合法节点列表。

**Error Codes（示例）**：
- `PROJECT_ROOT_NOT_SET` / `RUN_ID_INVALID`
- `PACKAGE_NOT_FOUND` / `WORKFLOW_NOT_FOUND`
- `GRAPH_PATH_INVALID` / `GRAPH_NOT_FOUND` / `GRAPH_JSON_INVALID`
- `GRAPH_SCHEMA_INVALID` / `GRAPH_INTEGRITY_INVALID`
- `STATE_FILE_NOT_FOUND` / `STATE_FILE_READ_FAILED`
- `FRONTMATTER_PARSE_FAILED` / `FRONTMATTER_SCHEMA_INVALID`
- `CURRENT_NODE_INVALID`

### Test Plan
- **Automated**:
  - 使用 `create-story-micro` 作为 fixture。
  - `RuntimeStore.getWorkflowState` 解析 `workflow.md`，断言 `currentNodeId == step-01-select-story`。
  - 缺失/非法 frontmatter、`currentNodeId` 不在 graph → 断言报错。
- **Manual**:
  - 启动 Runtime，加载示例包，读取 `@state/workflow.md` 并确认 UI 识别当前节点。
