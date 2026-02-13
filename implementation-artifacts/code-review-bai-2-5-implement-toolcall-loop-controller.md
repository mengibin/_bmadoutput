# Code Review: Story BAI-2.5 - Implement ToolCall Loop Controller

**Date:** 2026-02-09  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `_bmad-output/implementation-artifacts/bai-2-5-implement-toolcall-loop-controller.md`

---

## Scope

- Review target is limited to Story BAI-2.5 deliverables:
- `crewagent-builder-backend/app/services/tool_loop_service.py`
- `crewagent-builder-backend/tests/test_tool_loop_service.py`
- Related integration cross-check:
- `crewagent-builder-backend/app/routers/packages.py`
- `_bmad-output/implementation-artifacts/builder-ai-delivery-roadmap.md`

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 1 |
| LOW Issues | 0 |
| Lint | ✅ `ruff check app/services/tool_loop_service.py tests/test_tool_loop_service.py` |
| Unit Tests | ✅ `pytest -q tests/test_tool_loop_service.py` (6 passed) |
| Related Regression | ✅ `pytest -q tests/test_semantic_tool_service.py tests/test_context_envelope_service.py tests/test_prompt_composer_service.py tests/test_domain_persona_service.py` (34 passed) |

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 ToolCall 循环执行 | ✅ | 有 `tool_calls` 时执行工具并继续下一轮，无 `tool_calls` 时解析 suggestion：`crewagent-builder-backend/app/services/tool_loop_service.py:269`、`crewagent-builder-backend/app/services/tool_loop_service.py:304`；收敛路径测试：`crewagent-builder-backend/tests/test_tool_loop_service.py:61` |
| AC-2 停止条件明确 | ✅ | 超过上限返回 `AI_TOOL_LOOP_LIMIT_EXCEEDED`：`crewagent-builder-backend/app/services/tool_loop_service.py:337`；测试覆盖：`crewagent-builder-backend/tests/test_tool_loop_service.py:101` |
| AC-3 修复回路可控 | ✅ | `AI_BAD_RESPONSE/AI_VALIDATION_FAILED` 触发 repair，超过阈值失败：`crewagent-builder-backend/app/services/tool_loop_service.py:315`；测试覆盖：`crewagent-builder-backend/tests/test_tool_loop_service.py:134` |

---

## Issues

## 🟡 MEDIUM

### M1. `run_tool_loop` 尚未接入生产调用链，当前仅服务层可测

**Evidence**
- 定义位置：`crewagent-builder-backend/app/services/tool_loop_service.py:242`
- 调用点仅在测试：`crewagent-builder-backend/tests/test_tool_loop_service.py:83`、`crewagent-builder-backend/tests/test_tool_loop_service.py:118`、`crewagent-builder-backend/tests/test_tool_loop_service.py:144`、`crewagent-builder-backend/tests/test_tool_loop_service.py:178`
- 当前路由仍为 legacy `ai/step-draft`：`crewagent-builder-backend/app/routers/packages.py:545`
- Roadmap 目标要求 `POST /ai/sessions + /messages`：`_bmad-output/implementation-artifacts/builder-ai-delivery-roadmap.md:39`

**Impact**
- 该 Story 能力尚未在真实会话链路生效，端到端价值未闭环。

**Recommendation**
- 在会话主链路 story 中显式接线：`session create/message -> prompt composer -> context envelope -> tool loop -> suggestion/validate`。
- 若会话 API story 尚未拆分，建议先补一条 backend story，避免 Epic BAI-2 在 `review` 长时间阻塞。

---

## Resolved During Review

- ✅ 已将 LLM/Tool host 原生异常封装为 `ToolLoopError(AI_TOOL_EXECUTION_ERROR)`，确保返回结构化 `LoopResult.error`。  
  `crewagent-builder-backend/app/services/tool_loop_service.py:186`  
  `crewagent-builder-backend/app/services/tool_loop_service.py:201`
- ✅ 新增“LLM 抛异常 / Tool host 抛异常”回归测试，验证 `status=failed` 与错误码稳定。  
  `crewagent-builder-backend/tests/test_tool_loop_service.py:193`  
  `crewagent-builder-backend/tests/test_tool_loop_service.py:216`

---

## Conclusion

- BAI-2.5 的核心循环与测试基线完成度较高，AC 的单元行为基本满足。
- 结构化异常返回问题（H1）已修复并补充测试覆盖。
- 若要进入 `done`，仍需完成会话主链路接线（M1）。

---

## Post-Review Closure (2026-02-11)

- M1 已闭环：`run_tool_loop(...)` 已接入会话消息主链路（`/ai/sessions/{sessionId}/messages`）。
- 运行时支持 function-call 多轮循环，工具结果回注后继续下一轮。
- 结论更新：本 review 中遗留问题已清零，可进入 `done`。
