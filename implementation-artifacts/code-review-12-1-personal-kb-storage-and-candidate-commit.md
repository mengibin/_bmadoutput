# Code Review: Story 12-1 - Personal KB Storage & Candidate Commit

**Date:** 2026-03-11  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Story File:** `12-1-personal-kb-storage-and-candidate-commit.md`

---

## Scope

- This re-review audits the latest Story 12-1 implementation after Task 7 follow-up fixes and the later `IDENTITY.md` contract removal.
- Focus areas are fallback routing precision, `UPDATE/DELETE` mutation target resolution, and personal-KB structure consistency after removing `IDENTITY.md`.

---

## Summary

| Metric | Value |
|--------|-------|
| HIGH Issues | 0 |
| MEDIUM Issues | 0 |
| LOW Issues | 0 |
| Unit Tests | ✅ `npm --prefix crewagent-runtime test -- electron/services/personalKbService.test.ts electron/stores/runtimeStore.test.ts electron/services/chatContextBuilder.test.ts` passed (`84` tests) |
| Type Check | ✅ `cd crewagent-runtime && npx tsc --noEmit` passed |
| Review Outcome | Approved |

---

## Resolved Findings

### R1 - fallback 不再把场景性“偏好”误写到 `USER.md`

**What changed**  
- `USER.md` fallback 现在通过 `isStableUserPreference()` 收窄到稳定身份、称呼、语言和长期默认风格。  
- 不再仅因命中 `偏好/preference` 就把内容直接归入固定层。

**Evidence**  
- 分类收窄逻辑：`crewagent-runtime/electron/services/personalKbService.ts:234-256`
- fallback 路由使用新判定：`crewagent-runtime/electron/services/personalKbService.ts:279-288`
- 回归测试：`crewagent-runtime/electron/services/personalKbService.test.ts:222-236`

**Assessment**  
- 原先的 H1 已关闭。
- 当前实现与 Story 12-1 / 12-2 的 `USER vs MEMORY#General` 分层契约一致。

---

### R2 - `UPDATE/DELETE` 目标解析已优先使用 `candidateText`

**What changed**  
- 新增 `resolveMutationTarget()`，先用路由器抽取的 `candidateText` 查找目标，再把整句用户输入作为辅助 fallback。  
- `maybeCreateCandidate()` 已改为调用这一优先级链路。

**Evidence**  
- 查询优先级封装：`crewagent-runtime/electron/services/personalKbService.ts:736-765`
- `maybeCreateCandidate()` 改为 `primaryQuery=candidateText`：`crewagent-runtime/electron/services/personalKbService.ts:876-901`
- 回归测试：`crewagent-runtime/electron/services/personalKbService.test.ts:409-459`

**Assessment**  
- 原先的 M1 已关闭。
- 当前实现满足“旧值 + 新值同句”时优先按旧值定位目标的设计约束。

---

### R3 - `IDENTITY.md` 已从 personal KB 契约中移除

**What changed**  
- `IDENTITY.md` 不再属于 personal KB 初始化结构、路由 allowlist 或清空语义的一部分。  
- Runtime 仍兼容旧版本遗留文件：初始化时如果发现 `IDENTITY.md`，会自动删除并重建索引。

**Evidence**  
- personal KB 核心文件收敛：`crewagent-runtime/electron/stores/runtimeStore.ts:591-592`, `crewagent-runtime/electron/stores/runtimeStore.ts:1053-1054`
- legacy 清理：`crewagent-runtime/electron/stores/runtimeStore.ts:1312-1328`
- allowlist 移除：`crewagent-runtime/electron/services/personalKbService.ts:195-201`
- 回归测试：`crewagent-runtime/electron/stores/runtimeStore.test.ts:1646-1673`, `crewagent-runtime/electron/stores/runtimeStore.test.ts:1919-1964`

**Assessment**  
- 清空语义、索引统计与注入契约现在一致。
- 之前围绕 `IDENTITY.md` 的“可写但不注入 / clear 保留”歧义已关闭。

---

## Acceptance Criteria Status

| AC | Status | Notes |
|----|--------|-------|
| AC-1 初始化结构 | ✅ | personal KB 结构已收敛为 `USER/SOUL/MEMORY/daily/index/manifest` |
| AC-2 显式确认与安全路由 | ✅ | fallback 分类精度已补齐 |
| AC-3 提交写入与元数据 | ✅ | `targetSection` 与 mutation target resolve 均满足当前设计 |
| AC-4 索引重建 | ✅ | targeted tests pass |
| AC-5 极简治理 | ✅ | clear/rebuild 语义与最新 story 对齐 |
| AC-6 System confirmation 前置短路 | ✅ | 已有实现与验证 |
| AC-7 防重放与幂等 | ✅ | 已有实现与验证 |

---

## Conclusion

- No blocking findings remain for Story 12-1.
- Story 12-1 can be closed as `done`.
