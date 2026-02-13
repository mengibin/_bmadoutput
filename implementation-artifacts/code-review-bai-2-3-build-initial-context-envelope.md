# Code Review: Story BAI-2.3 - Build Initial Context Envelope

**Date:** 2026-02-09  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `_bmad-output/implementation-artifacts/bai-2-3-build-initial-context-envelope.md`

---

## Scope

- Review target is limited to Story BAI-2.3 deliverables:
- `crewagent-builder-backend/app/services/context_envelope_service.py`
- `crewagent-builder-backend/tests/test_context_envelope_service.py`
- Related integration cross-check:
- `crewagent-builder-backend/app/services/domain_persona_service.py`
- `crewagent-builder-backend/app/routers/packages.py`

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 1 |
| LOW Issues | 0 |
| Lint | ✅ `ruff check app/services/context_envelope_service.py tests/test_context_envelope_service.py` *(included in full suite check)* |
| Unit Tests | ✅ `pytest -q tests/test_context_envelope_service.py` *(included in full suite run)* |
| Related Regression | ✅ `pytest -q tests/test_domain_persona_service.py tests/test_prompt_composer_service.py tests/test_ai_step_draft.py tests/test_ai_step_profile_resolution.py` |

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 标准 Envelope 结构 | ⚠️ Partial | 7 个顶层键与 schema 已实现：`crewagent-builder-backend/app/services/context_envelope_service.py:21`、`crewagent-builder-backend/tests/test_context_envelope_service.py:186`；但“session starts”主链路尚未接入该服务。 |
| AC-2 四入口都可构建 | ✅ | workflow/step/agent/asset 分支均实现并有参数化测试：`crewagent-builder-backend/app/services/context_envelope_service.py:634`、`crewagent-builder-backend/tests/test_context_envelope_service.py:152`；workflow 目标一致性已补校验：`crewagent-builder-backend/app/services/context_envelope_service.py:312` |
| AC-3 与预算裁剪兼容 | ✅ | 大依赖场景仍保持固定结构（`items/total/truncated` + 7 顶层键）：`crewagent-builder-backend/tests/test_context_envelope_service.py:232` |

---

## Issues

## 🟡 MEDIUM

### M1. 服务尚未接入会话创建路径，AC 的 “When session starts” 未端到端闭环

**Evidence**
- `build_initial_context_envelope(...)` 定义：`crewagent-builder-backend/app/services/context_envelope_service.py:594`
- 调用点仅在测试：`crewagent-builder-backend/tests/test_context_envelope_service.py:176`
- 当前路由仍只有 legacy step-draft AI 路径，无 session create 路由：`crewagent-builder-backend/app/routers/packages.py:545`

**Impact**
- Envelope 目前是“可调用能力”而非“会话启动默认行为”。
- Story 叙述与真实运行路径存在落差。

**Recommendation**
- 在 Session API `POST /projects/{projectId}/ai/sessions` 实现时，强制调用本服务并持久化到 session context。

## Resolved During Review

- ✅ 已补 workflow 目标一致性校验：`target_type=workflow` 时强校验 `target_id` 与 resolved workflow 一致。  
  `crewagent-builder-backend/app/services/context_envelope_service.py`
- ✅ 已补 workflow target mismatch 回归测试。  
  `crewagent-builder-backend/tests/test_context_envelope_service.py`

---

## Conclusion

- BAI-2.3 的服务实现质量整体良好，结构化输出与多目标分支完成度高。
- workflow target 一致性问题已修复并有测试覆盖。
- 建议在进入 `done` 前补齐 session 接线。

---

## Post-Review Closure (2026-02-11)

- M1 已闭环：session/messages 运行链路已接入 `build_initial_context_envelope(...)`。
- envelope 已与 runtime prompt 组装和 budget trimming 串联，不再是“仅测试调用”状态。
- 结论更新：本 review 中遗留问题已清零，可进入 `done`。
