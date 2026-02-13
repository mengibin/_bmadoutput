# Story BAI-2.3: Build Initial Context Envelope

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want a standard initial context envelope,  
So that all entrypoints share deterministic context structure.

## Acceptance Criteria

### AC-1: 标准 Envelope 结构
- **Given** targetType in `{workflow, step, agent, asset}`
- **When** session starts
- **Then** envelope includes `session_meta/domain_profile/normative_baseline/target_snapshot/dependency_digest/tool_capabilities/revision_info`

### AC-2: 四入口都可构建
- **Given** each targetType
- **When** builder creates context
- **Then** target snapshot and dependency digest are populated for that target

### AC-3: 与预算裁剪兼容
- **Given** oversized dependency data
- **When** context is passed to budget service
- **Then** envelope fields remain schema-compatible

## Technical Scope

- 新增 `app/services/context_envelope_service.py`
- 定义 `ContextEnvelope` schema 与 builder 接口
- 连接 project/workflow/agent/asset 服务构建快照
- 预留与 budget service 的接口对接点

## Design Decisions (Frozen)

1. Envelope 顶层键固定为 7 个，不允许按入口增删顶层结构。
2. `target_snapshot` 必须包含目标对象完整内容；`dependency_digest` 仅包含摘要与引用索引。
3. `tool_capabilities` 仅暴露当前会话允许的语义工具，不透出未授权工具。
4. `revision_info` 统一来自后端当前 revision 快照，供后续 apply 并发校验复用。
5. 构建失败统一返回 `AI_CONTEXT_BUILD_ERROR`，并附带缺失对象定位信息。

## Interface Freeze

### Frozen Service Contract

```python
build_initial_context_envelope(
    db,
    *,
    user_id: int,
    project_id: int,
    workflow_id: int | None,
    target_type: str,
    target_id: str,
    mode: str,
) -> ContextEnvelope
```

### Frozen Envelope Shape

```json
{
  "session_meta": {},
  "domain_profile": {},
  "normative_baseline": {},
  "target_snapshot": {},
  "dependency_digest": {},
  "tool_capabilities": {},
  "revision_info": {}
}
```

### Frozen Target Contract

- `target_type=workflow`：`target_snapshot` 至少含 `workflow_md/graph_json/step_index`
- `target_type=step`：至少含 `step_markdown/node_meta/adjacent_edges`
- `target_type=agent`：至少含 `agent_definition/referenced_steps`
- `target_type=asset`：至少含 `path/content/file_type/referenced_by`

## Tasks / Subtasks

- [x] Task 1: 定义 envelope schema（AC: 1）
- [x] Task 2: 实现四类 targetType 构建分支（AC: 1,2）
- [x] Task 3: 增加 revision 与 tool capabilities 信息（AC: 1,2）
- [x] Task 4: 补 schema 兼容测试（AC: 3）

## Test Plan

- workflow/step/agent/asset 四入口输出字段齐全
- 目标对象缺失时返回 `AI_CONTEXT_BUILD_ERROR`
- 输出可被后续 trim 处理且不破 schema

## File List

- `crewagent-builder-backend/app/services/context_envelope_service.py`
- `crewagent-builder-backend/tests/test_context_envelope_service.py`

## Review Closure (2026-02-11)

- 原阻塞项已闭环：
  - Session/messages 主链路已落地并调用 `build_initial_context_envelope(...)`（Step 场景运行链路生效）。
  - 上下文构建结果已进入 prompt composer，并与 budget trim 形成串联。
- 已完成质量修复（2026-02-09）持续有效：
  - workflow `target_id` 与 resolved workflow 一致性校验。
  - workflow target mismatch 回归测试。
- 结论：本 Story 从 `review` 更新为 `done`。

## Dev Agent Record

### Validation

- `crewagent-builder-backend/.venv/bin/ruff check crewagent-builder-backend/app/services/context_envelope_service.py crewagent-builder-backend/tests/test_context_envelope_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_context_envelope_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_prompt_composer_service.py crewagent-builder-backend/tests/test_domain_persona_service.py`
