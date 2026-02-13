# Story BAI-2.6: Context Budget + Trimming Policy

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want predictable context budgeting,  
So that request size remains stable and controllable.

## Acceptance Criteria

### AC-1: 预算可配置
- **Given** model and token limits
- **When** composing request
- **Then** system uses configured `context_budget` and reserved response budget

### AC-2: 裁剪策略稳定
- **Given** oversized context
- **When** composing request
- **Then** non-target content is reduced to path + summary while target content remains intact

### AC-3: 可观察裁剪结果
- **Given** trim is applied
- **When** request is sent
- **Then** trim report contains removed/summarized sections and token estimate

## Technical Scope

- 新增 `app/services/context_budget_service.py`
- 实现 token estimate 与 trim pipeline
- 输出 `TrimmedEnvelope + TrimReport`
- 与 prompt composer/tool loop 集成

## Design Decisions (Frozen)

1. 预算输入采用显式参数优先：优先使用调用方传入 `context_token_budget/reserve_for_response`；缺失时按模型窗口推导默认比例（70/30）。
2. `target_snapshot` 永不裁剪（字段完整性强约束）；超预算时仅允许对 `dependency_digest` 及非核心扩展字段做降级。
3. 裁剪策略固定顺序：先摘要化依赖列表，再截断低优先级长文本，最后仅保留 `path + summary + hash`。
4. token 估算采用本地启发式估算器（不依赖外部 provider token API），保证可离线与可预测。
5. 每次裁剪必须产出 `trim_report`，记录 before/after token 估算与逐字段动作，便于审计与调参。

## Interface Freeze

### Frozen Core Contract

```python
trim_context_with_budget(
    *,
    envelope: Mapping[str, Any],
    model: str,
    context_token_budget: int,
    reserve_for_response: int,
) -> dict[str, Any]
```

### Frozen Output Shape

```json
{
  "trimmedEnvelope": {},
  "trimReport": {
    "applied": true,
    "estimateBefore": 4200,
    "estimateAfter": 2600,
    "contextBudget": 2800,
    "reserveForResponse": 1200,
    "actions": [
      {
        "section": "dependency_digest.references.asset_paths",
        "action": "summarize|truncate|drop",
        "beforeChars": 12000,
        "afterChars": 1600
      }
    ]
  }
}
```

### Frozen Invariants

- `trimmedEnvelope.target_snapshot == envelope.target_snapshot`
- 顶层键集合不变：`session_meta/domain_profile/normative_baseline/target_snapshot/dependency_digest/tool_capabilities/revision_info`
- `estimateAfter <= contextBudget`，否则返回 `AI_CONTEXT_BUDGET_EXCEEDED`

### Frozen Error Codes

- `AI_CONTEXT_BUDGET_EXCEEDED`
- `AI_PROMPT_COMPOSE_ERROR`

## Tasks / Subtasks

- [x] Task 1: 定义 budget config 与估算器（AC: 1）
- [x] Task 2: 实现 target-first trimming 规则（AC: 2）
- [x] Task 3: 实现 trim report（AC: 3）
- [x] Task 4: 增加回归测试（AC: 1,2,3）

## Test Plan

- 小上下文不触发裁剪
- 大上下文触发裁剪且保留 `target_snapshot`
- trim report 包含裁剪统计和条目列表

## File List

- `crewagent-builder-backend/app/services/context_budget_service.py`
- `crewagent-builder-backend/tests/test_context_budget_service.py`

## Dev Agent Record

### Review Closure (2026-02-11)

- 原 review 问题已闭环：
  - 预算服务已接入 Step 运行主链路（非孤立实现）。
  - 上下文裁剪在运行时生效并输出 trim report 日志。
  - 顶层键校验修复为“键集合一致”而非“顺序一致”，避免误报。
  - 增加回归测试覆盖 key 顺序无关场景。
- 结论：本 Story 从 `review` 更新为 `done`。

### Validation

- `crewagent-builder-backend/.venv/bin/ruff check crewagent-builder-backend/app/services/context_budget_service.py crewagent-builder-backend/tests/test_context_budget_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_context_budget_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_context_budget_service.py crewagent-builder-backend/tests/test_tool_loop_service.py crewagent-builder-backend/tests/test_semantic_tool_service.py crewagent-builder-backend/tests/test_context_envelope_service.py crewagent-builder-backend/tests/test_prompt_composer_service.py crewagent-builder-backend/tests/test_domain_persona_service.py`
