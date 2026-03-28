# Validation Report: Story 5-20-4

**Story**: 5-20-4 Approve, Execute, and Progress Visualization  
**Validation Date**: 2026-03-27  
**Status**: ✅ **APPROVED WITH RESIDUAL TEST RISK** (story closed in sprint tracking)

---

## Story Structure Validation

| Criterion | Status | Notes |
|:---|:---:|:---|
| Overview / Goal | ✅ | 清晰聚焦批准执行与进度可视化 |
| Acceptance Criteria | ✅ | 覆盖执行态推进、任务进度、恢复、终态、`waiting_user` 的双路径处理和完成后新版本 |
| Delivery Scope by Layer | ✅ | 分层清晰，便于前后端并行 |
| Risks / Mitigations | ✅ | 覆盖语义混淆、聚合错误、恢复不足 |
| Acceptance Checklist | ✅ | 可直接作为交付检查表 |
| Task Breakdown | ✅ | 已拆成后端状态推进、前端视图、恢复与测试 |

---

## Alignment Check

### 1. Alignment with `prd.md` ✅
- FR-PLAN-06、FR-PLAN-07 直接对应本 Story。
- NFR-PLAN-03 和 NFR-PLAN-05 也在本 Story 中有明确约束。

### 2. Alignment with Sub PRD ✅
- 对应子 PRD 的 AC-6、AC-6A、AC-7、AC-7A 中的执行、恢复和版本 rollover 部分。
- 与“Plan 进度独立于 workflow progress”的要求一致。

### 3. Alignment with Design ✅
- 设计文档中已将执行态、完成态、阻塞态作为独立 UI 状态定义。
- 本 Story 没有把 Plan 执行错误地映射成 workflow graph progress。

### 4. Alignment with Tech Spec ✅
- 与 `plan.state.json.tasks[]` 派生进度的技术路线一致。
- 与恢复逻辑、执行状态推进、独立事件模型一致。

---

## Dependency Analysis

| Dependency | Status | Notes |
|:---|:---:|:---|
| Story 5-20-1 | ✅ | 需要 `plan.state.json` 工件基础 |
| Story 5-20-2 | ✅ | 需要右侧面板承载进度 UI |
| Story 5-20-3 | ✅ | 需要计划先完成编辑和备注闭环 |
| Existing workflow progress hook | ✅ | 本 Story 明确不复用该模型，只参考实现模式 |

---

## Implementation Evidence

- `WorksPage.tsx` 现已按 plan 状态分流普通发送语义：
  - `drafting / reviewing / approved`：聊天输入捕获为新的 `plan.md`
  - `executing + waiting_user`：聊天输入默认作为当前 task 的决策输入
  - `completed`：聊天输入先归档已完成版本，再生成 `version + 1` 的新计划
- `planExecutionLogic.ts` 新增执行推进与聊天语义辅助函数，并补齐单测。
- `shared/planExecutionState.ts` 抽离了执行态状态迁移纯逻辑，使前后端对 `waiting_user / blocked / completed` 的判定一致。
- `runtimeStore.ts` 新增 `archiveConversationPlanVersion()`，将已完成版本归档到项目级 Plan 命名空间下的 `versions/v<version>/...`。
- `runtimeStore.ts` 现新增 `saveConversationPlanState()` 与 `updateConversationPlanExecutionState()`，作为 live `plan.state.json` 的专用持久化入口。
- `main.ts` / `preload.ts` / `electron-env.d.ts` 已补 `plan:archiveVersion`、`plan:saveState` 与 `plan:state-changed` IPC 链路。
- `PlanControlPanel` 保持独立于 workflow progress 的 Plan 任务与进度展示。
- 本轮 follow-up 已修复：
  - `Resume Execution` 仅在拿到有效 assistant 响应后才推进 task
  - `Revise Plan Instead` 不再先落盘 `reviewing` 再发重生成请求
  - `blocked` 状态新增独立说明卡片，显示阻塞任务与阻塞原因
  - 执行态现在通过受控 `plan.*` 工具推进 live `plan.state.json`，而不是继续依赖“最后一条 assistant 文本”
  - 执行态禁止创建 `@project/*计划.md` / `@project/*任务.md` 等影子文档，避免 Plan UI 与执行真源分叉
  - `WorksPage` 已订阅 `plan:state-changed`，后端状态变更会主动刷新右侧 Plan UI
  - 已补 `ConversationView.integration.test.tsx`，覆盖“后端 tool 已推进状态时前端不再做本地二次 fallback”的链路

## Gap Analysis

| Gap ID | Description | Severity | Recommendation |
|:---|:---|:---:|:---|
| G1 | 当前执行推进仍是状态机驱动，不是真实任务执行引擎 | Medium | 作为 MVP 接受，后续若接实际执行器再细化 task lifecycle |
| G2 | 仍未覆盖 “assistant 返回成功但 `persistPlanState()` 本地 fallback 持久化失败” 的 `WorksPage` 失败路径 | Low | 后续补 `WorksPage` 级组件测试或端到端测试，专门模拟该失败场景 |

---

## Recommendations

1. 本 Story 的主闭环已经形成，且执行态 follow-up 已完成，可维持 `Approved with Residual Test Risk` 结论。
2. 不要在后续 review 中把 Plan 进度重新绑回 workflow progress。
3. 后续优先补 `WorksPage` 级失败路径测试，而不是继续扩执行引擎复杂度。

---

## Verdict

**✅ Story 5-20-4 已完成开发实现与执行态 follow-up，同步结论为 `Approved with Residual Test Risk`。**

当前 Story 已具备：
- 清晰的执行态入口
- 独立的进度模型
- `waiting_user` 与 `Resume Execution` 的发送语义闭环
- `completed` 后的新版本归档与 `version + 1` rollover
- 恢复与终态标准

## Review Follow-up Addendum (2026-03-26)

| Review Finding | Status | Resolution |
|:---|:---:|:---|
| `Resume Execution` 会在无有效响应时推进 task | ✅ | `sendMessageContent()` 增加执行型响应校验，未返回有效 assistant 内容时不推进任务 |
| `Revise Plan Instead` 先写 `reviewing` 导致失败后状态混合 | ✅ | 改为只有新 `plan.md` 成功生成并保存后，才通过 `savePlanDraft()` 进入 `reviewing` |
| `blocked` 状态缺少明确说明 | ✅ | 右侧 Plan 面板新增 `Execution Blocked` 卡片，显示阻塞任务与阻塞原因 |

## Execution-State Follow-up Addendum (2026-03-27)

| Follow-up | Status | Resolution |
|:---|:---:|:---|
| 执行态仍依赖 assistant 最终文本推进状态 | ✅ | 引入受控 `plan.get_state / complete_current_task / wait_for_user / block_current_task` 工具，由后端更新 live `plan.state.json` |
| 执行中可能创建影子计划文档导致 UI 与状态源分叉 | ✅ | 执行态硬拦截 `@project/*计划.md` / `@project/*任务.md` / `@project/*plan.md` / `@project/*task.md`，仅放行已声明 deliverable |
| 后端状态更新后前端 UI 不会自动同步 | ✅ | 增加 `plan:state-changed` 推送，`WorksPage` 订阅并优先采用后端 live state |
| 缺少执行态状态推送的集成级自动化验证 | ✅ | 新增 `ConversationView.integration.test.tsx` 覆盖执行态状态推送与本地 fallback 分流 |

---

## Cross-Reference Links

- Story: [5-20-4-approve-execute-and-progress-visualization.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-4-approve-execute-and-progress-visualization.md)
- Tech Spec: [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md)
- Design: [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md)
- Dependency Story 1: [5-20-1-project-scoped-plan-storage-and-session-metadata.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md)
- Dependency Story 2: [5-20-2-chat-plan-panel-and-preview.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-2-chat-plan-panel-and-preview.md)
- Dependency Story 3: [5-20-3-plan-tab-editing-and-remarks.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-3-plan-tab-editing-and-remarks.md)
- Epic: [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md)
