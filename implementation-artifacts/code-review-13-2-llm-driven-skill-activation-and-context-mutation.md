# Code Review: Story 13.2 - LLM-Driven Skill Activation & Context Mutation

**Date:** 2026-03-19  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `13-2-llm-driven-skill-activation-and-context-mutation.md`

---

## 范围说明

- 本次为 re-review，聚焦 Story 13.2 在 H1/M1 follow-up 修复之后的最终状态。
- 复审范围包括 `skill.activate` 的 user-global alias 解析、chat/run mixed tool-call persistence 语义，以及新增回归测试覆盖。

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "user-global skills"` passed; `cd crewagent-runtime && npx vitest run electron/services/chatToolLoop.test.ts -t "persists non-activation tool protocol when skill.activate shares a turn with ordinary tools"` passed; `cd crewagent-runtime && npx vitest run electron/services/executionEngine.test.ts -t "persists non-activation tool protocol in run mode when activation shares a turn with ordinary tools"` passed |
| Type Check | ✅ `cd crewagent-runtime && npx tsc --noEmit --pretty false` passed |
| Review Outcome | Approved |

---

## Resolved Findings

### R1 - user-global skill 现在可以通过 `skill.activate` 成功激活

**What changed**  
- `FileSystemToolHost.activateSkill()` 不再直接把所有 skill path 都交给 run-state 语义的 `resolveExistingPath()`。
- 当前实现新增 `resolveReadableSkillPath()`，对 `@state/skills/global/...` 复用 `RuntimeStore.resolveAbsolutePath()` 的全局 skill 根解析，再校验解析结果仍然位于 global root 内。

**Evidence**  
- 激活入口： [fileSystemToolHost.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/fileSystemToolHost.ts#L1216)
- 全局 alias 解析 helper： [fileSystemToolHost.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/fileSystemToolHost.ts#L1268)
- 回归测试： [fileSystemToolHost.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/fileSystemToolHost.test.ts#L433)

**Assessment**  
- Story 13.7 暴露出来的 user-global skill 现在既能进入 registry，也能被 `skill.activate` 正确读取并注入 skill body。
- AC-1 的 user-global 激活阻塞已解除。

---

### R2 - mixed `skill.activate + ordinary tool` 回合现在只隐藏 activation 协议，不再吞掉业务工具历史

**What changed**  
- chat loop 和 run loop 都改成按 `activationCallIds` 粒度过滤持久化，而不是整轮布尔量。
- assistant message 现在只去掉 activation tool call，自带的普通工具调用仍然落盘；tool result 也只跳过 activation 自己的 tool message。

**Evidence**  
- chat loop assistant/tool persistence： [chatToolLoop.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/chatToolLoop.ts#L703)
- run loop assistant persistence： [executionEngine.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/executionEngine.ts#L1096)
- run loop tool-result persistence： [executionEngine.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/executionEngine.ts#L1326)
- chat mixed-turn regression： [chatToolLoop.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/chatToolLoop.test.ts#L347)
- run mixed-turn regression： [executionEngine.test.ts](/Users/mengbin/code/GPT/CrewAgent/crewagent-runtime/electron/services/executionEngine.test.ts#L2203)

**Assessment**  
- `skill.activate + fs.read` 这类组合调用现在会保留普通工具的 assistant/tool history，同时继续保证 activation 协议不写入持久化历史。
- AC-2 / AC-3 的持久化语义回归已修复。

---

## Acceptance Criteria 状态

| AC | Status | Notes |
|----|--------|-------|
| AC-1 LLM-triggered activation | ✅ | session-local 与 user-global skill 均可激活，`disable-model-invocation` 仍被正确拒绝 |
| AC-2 Context mutation after activation | ✅ | activation 后 system blocks 重建、tools 重算、activation 协议不持久化且 mixed-turn 历史不再回归 |
| AC-3 Multi-loop parity | ✅ | chat/run 两条 loop 对 activation mutation 和 mixed-turn persistence 的行为一致 |

---

## Residual Risk / Test Gap

- 本次 re-review 已补齐 user-global activation 和 mixed-turn persistence 的回归覆盖。
- 仍有一个与 Story 13.2 无关的现存测试波动：`fileSystemToolHost.test.ts` 里的 `terminal.run` timeout 断言在全量跑 3 个文件时偶发拿不到 partial stdout。本次定向回归和类型检查均通过，因此不阻塞 Story 13.2 关闭。

---

## Conclusion

- No blocking findings remain for Story 13.2.
- Story 13.2 can be closed as `done`.
