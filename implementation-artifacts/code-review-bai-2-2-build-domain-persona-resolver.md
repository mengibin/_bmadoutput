# Code Review: Story BAI-2.2 - Build Domain Persona Resolver

**Date:** 2026-02-09  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `_bmad-output/implementation-artifacts/bai-2-2-build-domain-persona-resolver.md`

---

## Scope

- Review target is limited to Story BAI-2.2 deliverables:
- `crewagent-builder-backend/app/services/domain_persona_service.py`
- `crewagent-builder-backend/app/services/prompt_models.py`
- `crewagent-builder-backend/app/services/prompt_composer_service.py`
- `crewagent-builder-backend/tests/test_domain_persona_service.py`
- Cross-check with BAI-2.3 integration point:
- `crewagent-builder-backend/app/services/context_envelope_service.py`

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 1 |
| LOW Issues | 0 |
| Lint | ✅ `ruff check app/services/prompt_models.py app/services/domain_persona_service.py app/services/prompt_composer_service.py app/services/context_envelope_service.py tests/test_domain_persona_service.py tests/test_prompt_composer_service.py tests/test_context_envelope_service.py` |
| Unit Tests | ✅ `pytest -q tests/test_domain_persona_service.py tests/test_prompt_composer_service.py tests/test_context_envelope_service.py` (21 passed) |
| Regression Tests | ✅ `pytest -q tests/test_ai_step_draft.py tests/test_ai_step_profile_resolution.py` (7 passed) |

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 动态 Persona 生效 | ⚠️ Partial | Resolver 已接入 context envelope 构建：`crewagent-builder-backend/app/services/context_envelope_service.py:653`；但会话创建链路尚未接入（`build_initial_context_envelope` 仅在自身与测试中被调用）。 |
| AC-2 无元数据可降级 | ✅ | 无信号时返回 fallback：`crewagent-builder-backend/app/services/domain_persona_service.py:389`；fallback 文案符合“行业应用架构师”语义：`crewagent-builder-backend/app/services/domain_persona_service.py:408` |
| AC-3 可追踪来源 | ✅ | `DomainProfile` 包含 `source/confidence`：`crewagent-builder-backend/app/services/domain_persona_service.py:77`；Layer-0B 注入 `Source hint`：`crewagent-builder-backend/app/services/domain_persona_service.py:413` |

---

## Issues

## 🟡 MEDIUM

### M1. 尚未形成“session 创建即生效”的端到端接线

**Evidence**
- 解析链路已实现：`resolve_domain_profile(...)`：`crewagent-builder-backend/app/services/domain_persona_service.py:362`
- 2.3 中已接入 resolver：`crewagent-builder-backend/app/services/context_envelope_service.py:653`
- 但 `build_initial_context_envelope(...)` 没有生产调用点（仅定义+测试调用）

**Impact**
- AC-1 的 “When creating a session” 仍缺最终一跳。
- 当前能力主要停留在服务层与测试层，未对真实会话请求生效。

**Recommendation**
- 在后续 Session API 的 create path 中统一调用 `build_initial_context_envelope(...)`，并将 `domain_profile` 显式传入 prompt composer 的 `ComposeInput.domainProfile`。

## Resolved During Review

- ✅ 空/半结构 metadata 来源失真问题已修复：通过 metadata 信号强度判定 `project_meta` 有效性，空 metadata 不再强制标记 `project_meta@0.95`。  
  `crewagent-builder-backend/app/services/domain_persona_service.py`
- ✅ 补充回归测试：空 metadata 走 `workflow_signal`；部分 metadata 走 `project_meta` 且置信度低于满配。  
  `crewagent-builder-backend/tests/test_domain_persona_service.py`

---

## Conclusion

- BAI-2.2 服务能力已可用，metadata 来源追踪问题已修复并有测试覆盖。
- 仍建议保持 `review`，直到 Session create 主链路完成接入。

---

## Post-Review Closure (2026-02-11)

- M1 已闭环：session/messages 主链路已接入 context envelope 与 persona 注入。
- 运行时 `ComposeInput.domainProfile` 已由 envelope 的 `domain_profile` 驱动，Layer-0B 不再依赖空值。
- 新增集成回归覆盖 domain profile 注入与运行链路接线。
- 结论更新：本 review 中遗留问题已清零，可进入 `done`。
