# Story BAI-2.1: Build Prompt Composer Layering Engine

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want to assemble system instructions by layers,  
So that entry/mode/domain variations are controlled and testable.

## Acceptance Criteria

### AC-1: 固定分层顺序
- **Given** different `targetType/mode` inputs
- **When** composing prompts
- **Then** output includes `Layer-0A` + `Layer-0B` + Guardrails + Contract + Entry + Mode + Context + ApplyPolicy

### AC-2: 分层可测试
- **Given** same input
- **When** composer runs repeatedly
- **Then** produced layer order and content signature are deterministic

### AC-3: 统一消息组装
- **Given** context envelope and user message
- **When** `compose_messages` is called
- **Then** request messages structure is stable and reusable by loop controller

### AC-4: 系统指令包含 schema-first 校验约束
- **Given** target contract requires structured output
- **When** composing system instruction
- **Then** guardrails include schema self-check rules (`required/type/enum/no extra fields`)
- **And** when schema confidence is low, model is instructed to read more context instead of guessing fields

## Technical Scope

- 新增 `app/services/prompt_composer_service.py`
- 定义 `ComposeInput/PromptLayer`
- 实现 `compose_system_instruction/compose_messages`
- 在 Guardrails 层固化 schema-first 约束文本
- 补充 snapshot/unit tests

## Design Decisions (Frozen)

1. Prompt 组装层顺序固定，不允许按入口动态重排：`0A -> 0B -> 1 -> 2 -> 3 -> 4 -> 5 -> 6`。
2. `compose_system_instruction(...)` 仅负责“组装与裁剪已给定输入”，不做数据库读取。
3. `compose_messages(...)` 输出固定为 `system + user` 基础结构，tool result message 由 loop controller 追加。
4. 每个 layer 具备稳定 `layerId`，便于测试与审计（而不是仅按自然语言匹配）。
5. 任一必需层缺失时返回 `AI_PROMPT_COMPOSE_ERROR`，不降级为静默空层。
6. Layer-1/Layer-2 必须显式声明 schema-first 自检，且要求不足信息时先 tool-read 再输出。

## Interface Freeze

### Frozen Service Contract

```python
compose_system_instruction(input: ComposeInput) -> list[PromptLayer]
compose_messages(
    *,
    input: ComposeInput,
    user_message: str,
    prior_messages: list[dict[str, str]] | None = None,
) -> list[dict[str, str]]
```

### Frozen `ComposeInput` Shape

```json
{
  "entrypoint": "workflow|step|agent|asset",
  "targetType": "workflow|step|agent|asset",
  "mode": "create|optimize",
  "domainProfile": {},
  "contextEnvelope": {},
  "writePolicy": {
    "allowApply": false,
    "requiresValidation": true
  }
}
```

### Frozen `PromptLayer` Shape

```json
{
  "layerId": "Layer-0A|Layer-0B|Layer-1|Layer-2|Layer-3|Layer-4|Layer-5|Layer-6",
  "title": "string",
  "content": "string",
  "required": true
}
```

## Tasks / Subtasks

- [x] Task 1: 定义 Prompt 层模型（AC: 1,2）
- [x] Task 2: 实现 8 层组装器（AC: 1,2）
- [x] Task 3: 接口输出标准化为 messages（AC: 3）
- [x] Task 4: 补测试（层顺序、幂等、异常分支）（AC: 2,3）
- [x] Task 5: 增加 schema guardrails 断言测试（AC: 4）

## Test Plan

- 相同输入多次执行输出一致
- `step/create` 与 `asset/optimize` 能命中不同 entry/mode 层
- 丢失关键输入时返回 `AI_PROMPT_COMPOSE_ERROR`
- 组装结果包含 schema-first 校验条款，且文案不被其他层覆盖

## File List

- `crewagent-builder-backend/app/services/prompt_composer_service.py`
- `crewagent-builder-backend/tests/test_prompt_composer_service.py`

## Dev Agent Record

### Review Closure (2026-02-11)

- 已完成 BAI-2 代码审查问题修复并闭环：
  - Step 主链路已统一使用 `ComposeInput + compose_messages(...)`。
  - schema-first guardrails 仍由 Layer-1 / Layer-2 固化并通过回归测试。
  - `writePolicy/contextEnvelope` 组装路径在运行链路稳定生效。
- 结论：本 Story 从 `review` 更新为 `done`。

### Validation

- `crewagent-builder-backend/.venv/bin/ruff check crewagent-builder-backend/app/services/prompt_composer_service.py crewagent-builder-backend/tests/test_prompt_composer_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_prompt_composer_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_ai_step_draft.py`
