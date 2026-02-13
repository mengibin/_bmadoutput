# Story BAI-2.4: Implement Semantic Read Tools (`builder.*`)

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want DB-semantic read tools instead of raw fs writes,  
So that tool calls are aligned with Builder storage model.

## Acceptance Criteria

### AC-1: 语义读工具可用
- **Given** tool call `builder.step.read`
- **When** node exists and user authorized
- **Then** tool returns step snapshot with metadata and revision

### AC-2: 鉴权与隔离
- **Given** unauthorized project access
- **When** any `builder.*.read` is called
- **Then** tool returns permission error without leaking data

### AC-3: 统一返回格式
- **Given** any semantic read tool
- **When** execution completes
- **Then** response shape follows `ok/data/error/meta`

## Technical Scope

- 新增 `app/services/semantic_tool_service.py`
- 实现 `builder.context.get/workflow.read/step.read/agent.read/asset.read/refs.find`
- 封装工具注册与调度接口
- 接入审计字段（sessionId/userId/toolName）

## Design Decisions (Frozen)

1. 工具宿主采用单一分发入口：所有 `builder.*` 工具必须经 `execute_semantic_read_tool(...)` 调度，不允许业务代码直接调用内部读函数。
2. `builder.context.get` 复用 `build_initial_context_envelope(...)`，禁止再定义第二套上下文拼装逻辑。
3. 鉴权在工具层统一执行：必须先校验 `user_id + project_id` 归属，再执行业务读取，避免跨租户数据泄露。
4. 语义读工具全部只读：`allowWrite=false` 固定写入 `meta`，并在工具名白名单层阻断任何写工具调用。
5. 返回结构固定为 `ok/data/error/meta`；`error.code` 与 HTTP 状态解耦，供 loop controller 统一处理。

## Interface Freeze

### Frozen Dispatcher Contract

```python
execute_semantic_read_tool(
    db,
    *,
    user_id: int,
    project_id: int,
    workflow_id: int | None,
    session_id: str | None,
    tool_name: str,
    arguments: Mapping[str, Any],
) -> dict[str, Any]
```

### Frozen Tool Name Set

- `builder.context.get`
- `builder.workflow.read`
- `builder.step.read`
- `builder.agent.read`
- `builder.asset.read`
- `builder.refs.find`

### Frozen Arguments Contract

- `builder.context.get`：`targetType`, `targetId`, `mode`, `workflowId?`
- `builder.workflow.read`：`workflowId`
- `builder.step.read`：`workflowId`, `nodeId|stepPath`
- `builder.agent.read`：`agentId`, `workflowId?`
- `builder.asset.read`：`path`, `workflowId?`
- `builder.refs.find`：`locator`（`type=id|path` + `value`）

### Frozen Unified Response Shape

```json
{
  "ok": true,
  "data": {},
  "error": null,
  "meta": {
    "tool": "builder.step.read",
    "sessionId": "string | null",
    "userId": 1,
    "projectId": 10,
    "workflowId": 2,
    "revision": {"workflow": "sha256...", "target": "sha256..."},
    "allowWrite": false
  }
}
```

### Frozen Error Codes

- `AI_TOOL_NOT_ALLOWED`
- `AI_TOOL_EXECUTION_ERROR`
- `AI_CONTEXT_BUILD_ERROR`

## Tasks / Subtasks

- [x] Task 1: 定义工具注册与调用协议（AC: 3）
- [x] Task 2: 实现六个只读工具（AC: 1,2）
- [x] Task 3: 接入权限检查与错误码映射（AC: 2,3）
- [x] Task 4: 补充工具级单元测试（AC: 1,2,3）

## Test Plan

- 每个工具在合法输入下返回 `ok=true`
- 跨用户访问返回权限错误
- 不存在对象返回明确 not-found 错误

## File List

- `crewagent-builder-backend/app/services/semantic_tool_service.py`
- `crewagent-builder-backend/tests/test_semantic_tool_service.py`

## Dev Agent Record

### Review Closure (2026-02-11)

- 代码审查结论为 0 个待修复问题，统一响应/鉴权/错误码符合 AC。
- 语义读工具已在 session/tool-loop 主链路被调用，非“仅单测可用”状态。
- 结论：本 Story 从 `review` 更新为 `done`。

### Validation

- `crewagent-builder-backend/.venv/bin/ruff check crewagent-builder-backend/app/services/semantic_tool_service.py crewagent-builder-backend/tests/test_semantic_tool_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_semantic_tool_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_context_envelope_service.py crewagent-builder-backend/tests/test_prompt_composer_service.py crewagent-builder-backend/tests/test_domain_persona_service.py`
