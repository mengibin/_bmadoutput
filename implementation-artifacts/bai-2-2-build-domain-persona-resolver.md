# Story BAI-2.2: Build Domain Persona Resolver

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-bai-2-prompt-composer-context-tool-loop.md`

## Story

As a **System**,  
I want domain persona to be dynamic per project/workflow context,  
So that AI role is not a generic builder assistant.

## Acceptance Criteria

### AC-1: 动态 Persona 生效
- **Given** project has domain metadata
- **When** creating a session
- **Then** `Layer-0B` reflects industry/target users/pain points

### AC-2: 无元数据时可降级
- **Given** no explicit domain metadata
- **When** resolver runs
- **Then** returns safe fallback persona instead of generic “Builder helper”

### AC-3: 可追踪来源
- **Given** generated domain profile
- **Then** result includes source hints (`project_meta/workflow_signal/fallback`)

## Technical Scope

- 新增 `app/services/domain_persona_service.py`
- 定义 `DomainProfile` 数据结构
- 实现 `resolve_domain_profile/build_domain_persona_layer`
- 接入 prompt composer `Layer-0B`

## Design Decisions (Frozen)

1. Persona 来源优先级固定：`project metadata > workflow signal > fallback template`。
2. `fallback` 必须是“行业应用架构师”语义，不允许返回通用 `Builder assistant` 文案。
3. Resolver 输出必须带 `source` 与 `confidence`，便于后续调试和灰度观察。
4. `build_domain_persona_layer(...)` 使用固定模板字段，避免不同入口文风漂移。
5. 当元数据不完整时不报错中断，降级到 fallback；仅结构损坏时返回 `AI_DOMAIN_PROFILE_INCOMPLETE`。

## Interface Freeze

### Frozen Service Contract

```python
resolve_domain_profile(
    db,
    *,
    project_id: int,
    workflow_id: int | None,
) -> DomainProfile

build_domain_persona_layer(profile: DomainProfile) -> PromptLayer
```

### Frozen `DomainProfile` Shape

```json
{
  "industry_domain": "string",
  "target_users": ["string"],
  "core_pain_points": ["string"],
  "expected_outcomes": ["string"],
  "quality_style": ["string"],
  "source": "project_meta|workflow_signal|fallback",
  "confidence": 0.0
}
```

### Frozen Layer Binding

- Persona 输出仅用于 `Layer-0B`。
- `Layer-0B` 由 prompt composer 统一拼接，不允许业务入口自行拼接 persona 文本。

## Tasks / Subtasks

- [x] Task 1: 定义 `DomainProfile` schema（AC: 1,2,3）
- [x] Task 2: 实现 resolver 优先级（AC: 1,2）
- [x] Task 3: 生成 persona layer 文本（AC: 1,2）
- [x] Task 4: 增加来源追踪字段与测试（AC: 3）

## Test Plan

- 有 domain 元数据时输出命中对应行业特征
- 无元数据时输出 fallback persona 且不为空
- 输出带 `source` 信息

## File List

- `crewagent-builder-backend/app/services/domain_persona_service.py`
- `crewagent-builder-backend/app/services/prompt_models.py`
- `crewagent-builder-backend/app/services/prompt_composer_service.py`
- `crewagent-builder-backend/tests/test_domain_persona_service.py`

## Review Closure (2026-02-11)

- 原阻塞项已闭环：
  - Session/messages 主链路已接入 `build_initial_context_envelope(...)`。
  - `domain_profile` 已注入 `ComposeInput.domainProfile`，由 prompt composer 的 `Layer-0B` 统一消费。
  - 新增集成回归覆盖运行链路中的 persona 注入（见 `tests/test_ai_step_draft.py`）。
- 保留 2026-02-09 的质量修复结论：
  - 空/半结构 metadata 来源与置信度失真问题已修复。
  - 空 metadata 与部分 metadata 回归测试已补齐。
- 结论：本 Story 从 `review` 更新为 `done`。

## Dev Agent Record

### Validation

- `crewagent-builder-backend/.venv/bin/ruff check crewagent-builder-backend/app/services/prompt_models.py crewagent-builder-backend/app/services/domain_persona_service.py crewagent-builder-backend/app/services/prompt_composer_service.py crewagent-builder-backend/tests/test_domain_persona_service.py crewagent-builder-backend/tests/test_prompt_composer_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_domain_persona_service.py crewagent-builder-backend/tests/test_prompt_composer_service.py`
- `crewagent-builder-backend/.venv/bin/pytest -q crewagent-builder-backend/tests/test_ai_step_draft.py crewagent-builder-backend/tests/test_ai_step_profile_resolution.py`
