# Validation Report: Story 5-20-1 (Revalidated)

**Story**: 5-20-1 Project-Scoped Plan Storage and Session Metadata  
**Validation Date**: 2026-03-25  
**Status**: ✅ **APPROVED** (story closed in sprint tracking)

---

## Story Structure Validation

| Criterion | Status | Notes |
|:---|:---:|:---|
| Overview / Goal | ✅ | 目标明确，聚焦项目级 Plan 工件与会话元数据基础 |
| Acceptance Criteria | ✅ | 覆盖初始化、空白草稿创建、恢复、conversation 清理、项目级清理 |
| Delivery Scope by Layer | ✅ | 已拆分 Backend / Frontend / Integration |
| Risks / Mitigations | ✅ | 覆盖 projectId、一致性与清理遗漏风险 |
| Acceptance Checklist | ✅ | 可直接作为开发完成检查表 |
| Task Breakdown | ✅ | 已拆到 Backend / Frontend / Integration / Test |

---

## Alignment Check

### 1. Alignment with `prd.md` ✅
- FR-PLAN-01 / FR-PLAN-02 / FR-PLAN-08 已在总 PRD 中登记。
- NFR-PLAN-01 / NFR-PLAN-02 / NFR-PLAN-03 / NFR-PLAN-04 已覆盖本 Story 的核心目标。

### 2. Alignment with Sub PRD ✅
- 与 [prd-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-chat-plan-mode.md) 中“项目优先”路径规范一致。
- 与子 PRD 的 AC-2、AC-8 完整对应。
- 已补充“Plan 开启后 `plan.md` 为空草稿”的初始化语义。

### 3. Alignment with Epic ✅
- 已在 [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md) 中定义为 PLAN-1.1。
- 属于所有后续 Plan Story 的前置基础。

### 4. Alignment with Tech Spec ✅
- 与 [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md) 中的目录结构、metadata 扩展、清理规则一致。

---

## Dependency Analysis

| Dependency | Status | Notes |
|:---|:---:|:---|
| Story 5-8 Conversation Persistence | ✅ | 现有 conversation metadata/messages 持久化已存在，可扩展字段 |
| RuntimeStore project runtime root | ✅ | 现有项目级 runtime root 能承载 Plan 工件命名空间 |
| Project cleanup flow | ⚠️ Partial | 需要确认孤立项目清理逻辑确实覆盖新的 `state/projects/<projectId>` 子树 |

---

## Gap Analysis

| Gap ID | Description | Severity | Recommendation |
|:---|:---|:---:|:---|
| G1 | `projectId` 的来源在文档中已说明，但与 runtime 实际 ID 解析方式尚未显式绑定 | Medium | 实施时必须复用现有 runtime `getProjectIdForRoot()` 逻辑 |
| G2 | Plan 初始化默认内容尚未在 Story 中完全固定 | Low | 以 Tech Spec 中的最小 `plan.state.json` 模型为默认模板 |
| G3 | 项目级清理联动目前还是要求，未写成单独测试场景模板 | Medium | 在对应 validation report 或后续测试计划中加项目清理场景 |

---

## Recommendations

1. 先实施本 Story，再进行右侧 Plan 面板开发。
2. 不要在本 Story 中引入 UI 复杂行为，只完成 metadata 和工件生命周期闭环。
3. 将 `projectId` 统一绑定到现有 runtime 稳定键，避免后续路径迁移。

---

## Verdict

**✅ Story 5-20-1 已完成实现更新，并可进入评审。**

当前已经具备：
- 上游需求依据
- 明确的目录规范
- 空白草稿初始化行为
- 可恢复与可清理边界
- 已通过针对性自动化验证

---

## Post-Implementation Addendum (2026-03-24)

> 原始 pre-implementation 校验结论保留；以下为根据 2026-03-24 review 整改后的实现复核结果。

| Check | Status | Evidence |
|:---|:---:|:---|
| `plan:ensureArtifacts` conversation/type guard | ✅ | 仅允许真实存在且当前 `activeType = 'chat'` 的 conversation 初始化 Plan 工件 |
| Metadata persistence / recovery | ✅ | `ConversationMetadata` / 前端 `Conversation` 均已持久化 `chatSubmode` 与 Plan 路径字段，并补齐 `appStore` 恢复测试 |
| Conversation cleanup | ✅ | 删除 conversation 时会清理对应 `plans/<conversationId>/` 子目录 |
| Project-level cleanup compatibility | ✅ | orphan project 清理删除整个项目 runtime 根目录，显式测试覆盖 Plan 命名空间一并删除 |
| Empty draft initialization | ✅ | `plan.md` 初始化为空字符串，`remarks.json`/`plan.state.json` 初始内容已通过单测锁定 |
| Regression coverage | ✅ | `npm test -- electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts` passed |
| Type safety | ✅ | `npx tsc --noEmit` passed |

### Addendum Notes

1. 首轮 review 指出的 3 个 MEDIUM 问题已全部关闭：
   - conversation/type 校验缺失
   - 重新开启 Plan 时错误重置 `planStatus`
   - metadata/recovery/cleanup 自动化证据不足
2. 2026-03-25 进一步按最新产品语义完成“空白 plan 初始化”整改，并新增工件默认内容断言。
3. Story 文档已同步补充 `Review Follow-ups (AI)`、`File List`、`Change Log` 与 `Senior Developer Review (AI)`。
4. 当前 Story `5-20-1` 可进入复审结论；Story `5-20-2` ~ `5-20-4` 仍需分别验证。

---

## Cross-Reference Links

- Story: [5-20-1-project-scoped-plan-storage-and-session-metadata.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md)
- Tech Spec: [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md)
- Design: [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md)
- Sub PRD: [prd-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-chat-plan-mode.md)
- Epic: [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md)
