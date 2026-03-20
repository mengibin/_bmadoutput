# Code Review: Story 13.1 - Runtime Skill Discovery & Registry Injection

**Date:** 2026-03-19  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `13-1-runtime-skill-discovery-and-registry-injection.md`

---

## 范围说明

- 本次为 re-review，聚焦 Story 13.1 在 H1/M1 follow-up 修复之后的最终状态。
- 复审范围包括 `SkillRegistryService` 的 effective-agent 去重顺序、`attachedSkills` 主进程入口规范化，以及新增回归测试覆盖。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `npm --prefix crewagent-runtime test -- electron/services/attachedSkillSource.test.ts electron/services/skillRegistryService.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/executionEngine.test.ts` passed |
| Type Check | ✅ `cd crewagent-runtime && npx tsc --noEmit` passed |
| Review Outcome | Approved |

---

## Resolved Findings

### R1 - duplicate source 现在不会再吞掉其它 agent 的可见 skill 绑定

**What changed**  
- `SkillRegistryService.discover()` 不再在解析前按 `skillMdAbsPath` 提前去重。  
- 当前实现先完成 skill summary 解析，再做 effective-agent 可见性判断，最后只对“对当前 agent 可见”的 skill 做去重。

**Evidence**  
- 实现： [skillRegistryService.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/skillRegistryService.ts#L129)  
- 回归测试： [skillRegistryService.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/skillRegistryService.test.ts#L169)

**Assessment**  
- 同一 `SKILL.md` 被多条 attachment 绑定给不同 agent 时，当前 effective agent 仍能拿到自己的可见 skill。
- Story 13.1 的 agent-scoped registry 过滤现在与 AC-2 一致。

---

### R2 - `agent:dispatch` 现在与 chat/run 共用同一套 `attachedSkills` 规范化逻辑

**What changed**  
- 新增共享 helper [attachedSkillSource.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/attachedSkillSource.ts)。  
- `agent:dispatch`、`chat:send`、`runs:continue` 现在都先调用 `parseAttachedSkillsPayload()`，再把规范化结果下传。

**Evidence**  
- 共享解析实现： [attachedSkillSource.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/attachedSkillSource.ts#L1)  
- `agent:dispatch` 接入： [main.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/main.ts#L3240)  
- chat/run 接入仍保持一致： [main.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/main.ts#L3550) [main.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/main.ts#L3698)  
- 解析回归测试： [attachedSkillSource.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/attachedSkillSource.test.ts#L1)

**Assessment**  
- `attachedSkills` 的 trimming、空值过滤和 `agentIds` sanitation 现在在三条入口链路上一致。
- Story 13.1 的 AC-3 不再存在 agent-mode 特殊路径差异。

---

## Acceptance Criteria 状态

| AC | Status | Notes |
|----|--------|-------|
| AC-1 Runtime skill discovery | ✅ | discovery、结构化错误、supporting file index 和 scoped dedupe 均满足 |
| AC-2 Registry injection | ✅ | registry 注入保持轻量，且按当前 effective agent 正确过滤 |
| AC-3 Multi-mode consistency | ✅ | chat / agent / run 三条入口都复用统一的 `attachedSkills` 规范化链路 |

---

## Residual Risk / Test Gap

- 目前的回归测试覆盖了 service 层和 run-mode 注入链路，但仍没有 renderer-to-main 的 UI 集成测试去验证某个具体界面何时传入 `attachedSkills`。这不阻塞 Story 13.1，因为该 story 的约束是“host attaches runtime-accessible skill sources”，不是“当前 UI 已提供 skill attachment UX”。

---

## Conclusion

- No blocking findings remain for Story 13.1.
- Story 13.1 can be closed as `done`.
