# Code Review: Story BAI-2.1 - Build Prompt Composer Layering Engine

**Date:** 2026-02-09  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `_bmad-output/implementation-artifacts/bai-2-1-build-prompt-composer-layering-engine.md`

---

## Scope

- Review target is limited to Story BAI-2.1 deliverables:
- `crewagent-builder-backend/app/services/prompt_composer_service.py`
- `crewagent-builder-backend/tests/test_prompt_composer_service.py`

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 2 |
| LOW Issues | 1 |
| Unit Tests | ✅ `pytest -q tests/test_prompt_composer_service.py` (7 passed) |
| Regression Tests | ✅ `pytest -q tests/test_ai_step_draft.py` (2 passed) |
| Lint | ✅ `ruff check app/services/prompt_composer_service.py tests/test_prompt_composer_service.py` |

---

## Acceptance Criteria Verification

| AC | Status | Evidence |
|----|--------|----------|
| AC-1 固定分层顺序 | ✅ | 固定 8 层组装顺序：`crewagent-builder-backend/app/services/prompt_composer_service.py:203`，顺序断言：`crewagent-builder-backend/tests/test_prompt_composer_service.py:31` |
| AC-2 分层可测试/可重复 | ✅ | JSON 摘要按 key 排序保障稳定性：`crewagent-builder-backend/app/services/prompt_composer_service.py:76`；幂等测试：`crewagent-builder-backend/tests/test_prompt_composer_service.py:46` |
| AC-3 统一消息组装 | ✅ | `compose_messages` 统一输出 system + prior + user 结构：`crewagent-builder-backend/app/services/prompt_composer_service.py:231`；结构测试：`crewagent-builder-backend/tests/test_prompt_composer_service.py:67` |

---

## Issues

## 🟡 MEDIUM

### M1. `writePolicy` 字符串布尔值被错误解释为 `true`

**Evidence**
- 布尔转换使用 `bool(...)`：`crewagent-builder-backend/app/services/prompt_composer_service.py:187`
- 布尔转换使用 `bool(...)`：`crewagent-builder-backend/app/services/prompt_composer_service.py:188`
- 复现输出：`allowApply='false'` 时仍生成 `allowApply=true`

**Impact**
- `Layer-6 Apply Policy` 会产生与真实策略相反的提示，降低模型行为可控性与审计可信度。

**Recommendation**
- 引入严格布尔解析（仅接受 `True/False` 或 `"true"/"false"`），非法值抛 `PromptComposeError`。

### M2. `contextEnvelope` 不可 JSON 序列化时抛出 `TypeError`，未落入统一错误码

**Evidence**
- 直接 `json.dumps(...)`，无异常封装：`crewagent-builder-backend/app/services/prompt_composer_service.py:76`
- 调用路径：`crewagent-builder-backend/app/services/prompt_composer_service.py:178`
- 复现：`contextEnvelope={"x": object()}` 抛出 `TypeError: Object of type object is not JSON serializable`

**Impact**
- 与 Story 约定的 `AI_PROMPT_COMPOSE_ERROR` 错误模型不一致，可能导致上层返回 500 或错误码不统一。

**Recommendation**
- 在 `_to_compact_json` 或 `_build_layer_5` 中捕获序列化异常并转换为 `PromptComposeError`。

## 🟢 LOW

### L1. `prior_messages` 缺少元素类型防御，异常数据会触发 `AttributeError`

**Evidence**
- 直接调用 `message.get(...)`，未先校验元素类型：`crewagent-builder-backend/app/services/prompt_composer_service.py:246`
- 复现：`prior_messages=[None]` 抛 `AttributeError: 'NoneType' object has no attribute 'get'`

**Impact**
- 会话历史若出现污染数据，`compose_messages` 将非预期崩溃，影响恢复路径稳定性。

**Recommendation**
- 对每个 `prior_messages` 元素先做 `Mapping` 校验；非法元素可跳过或统一抛 `PromptComposeError`。

---

## Conclusion

- Story BAI-2.1 的主流程和 AC 已满足，测试覆盖了核心顺序与幂等行为。
- 建议在进入 Done 前修复上述 2 个 MEDIUM 问题，并补充对应回归测试。

---

## Post-Review Closure (2026-02-11)

- M1 已闭环：`writePolicy` 布尔解析改为严格校验，不再把 `"false"` 误判为 `true`。
- M2 已闭环：`contextEnvelope` 非 JSON 可序列化输入已统一映射为 `AI_PROMPT_COMPOSE_ERROR`。
- 结论更新：本 review 中遗留问题已清零，可进入 `done`。
