# Code Review: Story Group 5-20 - Runtime Chat Plan Mode

**Date:** 2026-03-27  
**Reviewer:** AI Code Reviewer (BMAD Method)  
**Scope:** Stories `5-20-1` ~ `5-20-4`  
**Status:** 🟢 **APPROVED WITH RESIDUAL TEST RISK - Story 5-20-1 and Story 5-20-2 approved; Story 5-20-3 and Story 5-20-4 re-reviewed clean with residual test risk only**

---

## Scope

本代码评审文档用于在 `Runtime Chat Plan Mode` 完成实现后，作为统一的跨 Story 汇总评审入口。

覆盖范围：

- Story 5-20-1: 项目级 Plan 工件与会话元数据
- Story 5-20-2: Chat Plan 面板与预览
- Story 5-20-3: `plan.md` 顶部 Tab 编辑与备注
- Story 5-20-4: 同意执行与进度可视化

关联文档：

- [prd-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-chat-plan-mode.md)
- [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md)
- [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md)
- [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md)

---

## Review Goal

确认 `Chat Plan Mode` 的最终实现是否满足以下目标：

1. 不引入新的顶层 `ConversationType`。
2. Plan 工件必须按项目优先命名空间落盘。
3. `plan.md` 能在顶部 Tab 中打开和编辑。
4. 备注是独立工件而非混入聊天消息。
5. Chat 能自动读取 Plan 与备注上下文继续优化。
6. Plan 执行进度独立于现有 workflow progress。
7. conversation 删除与项目清理逻辑正确覆盖 Plan 数据。

---

## Summary

| Metric | Value |
|--------|-------|
| Stories Reviewed | 4 / 4 |
| HIGH Issues | 0 unresolved |
| MEDIUM Issues | 0 unresolved |
| LOW Issues | 0 unresolved |
| Type Check | ✅ `npx tsc --noEmit` |
| Unit / Integration Tests | ✅ `npm test -- --run src/pages/WorksPage/ConversationView.integration.test.tsx src/pages/WorksPage/planExecutionLogic.test.ts src/pages/WorksPage/PlanControlPanel.test.tsx src/pages/WorksPage/planPanelLogic.test.ts src/pages/WorksPage/planAuthoring.shared.test.ts src/pages/WorksPage/planDraftCapture.test.ts electron/services/fileSystemToolHost.test.ts electron/stores/runtimeStore.test.ts src/stores/appStore.test.ts` |
| Review Outcome | Approved with Residual Test Risk (`5-20-1`, `5-20-2` approved; `5-20-3`, `5-20-4` clean with residual test risk only) |

> 说明：Story `5-20-4` 已完成复审，前一轮 4 个实现级问题和本轮执行态状态同步 follow-up 均已关闭；当前剩余风险仅为 `WorksPage` 持久化失败路径的组件/集成级自动化覆盖偏薄。

---

## Review Checklist

### Architecture / Data Model

- [ ] `chatSubmode` 采用 `chat` 子模式实现，而不是新增顶层会话类型
- [ ] `ConversationMetadata` 与前端 `Conversation` 类型保持一致
- [ ] `planStatus / planPath / planStatePath / planRemarksPath` 字段持久化正确
- [ ] Plan 工件路径符合：

```text
@state/projects/<projectId>/plans/<conversationId>/...
```

### Storage / Cleanup

- [ ] 首次开启 Plan 时会自动创建 `plan.md`
- [ ] 首次开启 Plan 时会自动创建 `plan.state.json`
- [ ] 首次开启 Plan 时会自动创建 `remarks.json`
- [ ] 删除 conversation 时仅删除该 conversation 的 Plan 子目录
- [ ] 删除项目或孤立项目清理时能删除整个项目级 Plan 数据

### UI / UX

- [ ] Chat 详情区已替换为 Plan 面板
- [ ] 普通 Chat 与 Plan 模式切换正常
- [ ] 右侧不显示 `plan.md` 正文预览
- [ ] 状态标签与进度来自 `plan.state.json`
- [ ] `查看/编辑 plan.md` 能打开顶部 Tab
- [ ] 已打开的 Plan Tab 不会重复创建

### Remarks / Iteration

- [ ] 备注持久化到 `remarks.json`
- [ ] 备注没有混进 `messages.json` 作为唯一事实来源
- [ ] 返回 Chat 后，LLM 能读取最新计划和备注
- [ ] Chat 回复能体现计划修订后的上下文

### Execution / Progress

- [ ] `同意并执行 Plan` 能推进 `plan.state.json.status = executing`
- [ ] 任务状态和完成比例渲染正确
- [ ] `completed` / `blocked` 等终态可见
- [ ] 应用重启后计划进度可恢复
- [ ] 未错误复用现有 workflow progress 作为唯一来源

---

## Suggested Review Targets

实施后重点检查这些文件：

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/hooks/useConversationWorkspace.ts`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.css`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`

如果实现中新增了 Plan 专属 hook / component / service，也应一并纳入。

---

## Expected Test Evidence

### Required Automated Coverage

- [ ] RuntimeStore 单测：Plan 工件初始化/读取/删除
- [ ] Store/IPC 单测：Plan metadata 持久化与恢复
- [ ] 组件测试：Plan 面板状态渲染
- [ ] 集成测试：顶部 Tab 打开 `plan.md`
- [ ] 集成测试：remarks 持久化
- [ ] 集成测试：Chat context injection
- [ ] 恢复测试：重启后 Plan 状态和进度恢复

### Required Manual Coverage

- [ ] 开启 Plan -> 工件创建
- [ ] 编辑 `plan.md` -> 保存 -> 返回 Chat
- [ ] 添加备注 -> 再次聊天优化
- [ ] 同意执行 -> 进度显示
- [ ] 删除 conversation -> 子目录清理
- [ ] 删除项目 -> 项目级目录清理

---

## Findings

### Story 5-20-1 Resolved Findings

- ✅ `M1` `plan:ensureArtifacts` 现已校验 conversation 存在且 `activeType = 'chat'`，不再允许创建脱离 conversation index 的孤立 Plan 目录。证据：`crewagent-runtime/electron/stores/runtimeStore.ts`、`crewagent-runtime/electron/stores/runtimeStore.test.ts`
- ✅ `M2` 重新开启 Plan Mode 时，前端不再把已有 `plan.state.json.status` 覆盖成 `drafting`；IPC 现返回真实持久化状态。证据：`crewagent-runtime/electron/stores/runtimeStore.ts`、`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `M3` 已补齐 metadata 持久化/恢复、非法初始化拦截、orphan project 清理覆盖 Plan 命名空间的自动化测试。证据：`crewagent-runtime/electron/stores/runtimeStore.test.ts`、`crewagent-runtime/src/stores/appStore.test.ts`

### Story 5-20-2 Resolved Findings

- ✅ `H1` 切换 Plan conversation 时先清空旧的 `planMarkdown / planState`，并为异步 artifact 读取增加 request guard，避免上一会话状态污染当前面板。证据：`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `M1` `Regenerate Plan` 现优先使用当前聊天输入框草稿；只有在无草稿时才回退到默认修订 prompt。证据：`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`、`crewagent-runtime/src/pages/WorksPage/planPanelLogic.ts`、`crewagent-runtime/src/pages/WorksPage/planPanelLogic.test.ts`
- ✅ `M1.1` `Regenerate Plan` 在有输入时会继续结合 `PLAN_CONTEXT` 中的 remarks 共同优化；优化成功写回 `plan.md` 后会清空已消费的 `remarks.json`。证据：`crewagent-runtime/electron/main.ts`、`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `M2` `Approve and Execute` 现要求至少存在一条结构化任务，空草稿 / 无结构化任务不会再直接进入 `executing`。证据：`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`、`crewagent-runtime/src/pages/WorksPage/planPanelLogic.ts`、`crewagent-runtime/src/pages/WorksPage/planPanelLogic.test.ts`
- ✅ `M3` `Regenerate Plan` 失败或中止时不再清空用户输入草稿；输入仅在成功路径上清理。证据：`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `M4` `PLAN_CONTEXT` 的 remarks 摘要现保留 `section heading / excerpt`，可将 section-scoped comments 映射回原计划位置。证据：`crewagent-runtime/electron/main.ts`
- ✅ `M5` `Regenerate Plan` 现通过原子化的 plan draft 持久化入口统一写入 `plan.md` / `plan.state.json` / 已消费 remarks，且仅在全部成功后才清空输入，避免保存失败时丢失修订草稿或留下半成功状态。证据：`crewagent-runtime/electron/stores/runtimeStore.ts`、`crewagent-runtime/electron/stores/runtimeStore.test.ts`、`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `M6` `PLAN_CONTEXT` 现直接注入完整当前 `plan.md` 和全部 unresolved remarks，不再仅传摘要或截断前 6 条 comments。证据：`crewagent-runtime/electron/main.ts`
- ✅ `L2` `Regenerate Plan` 作为 UI 动作不再向聊天历史追加内部英文修订指令，也不会把生成出的 plan 正文当作普通 assistant 消息写入 transcript。证据：`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- ✅ `L1` 已补最小面板渲染测试，覆盖按钮可见性、`Resume Execution` 条件显示和 `Approve Plan` 移除回归。证据：`crewagent-runtime/src/pages/WorksPage/PlanControlPanel.test.tsx`

### Story 5-20-3 Applied Review Follow-Ups

- ✅ `M1` `plan.md` 顶部 tab 现绑定自身的 `conversationId / planPath / planRemarksPath`，保存与 remark 持久化不再依赖当前 `selectedConversation`。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`、`crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.ts`
- ✅ `M2` comment dialog 现会在切换 workspace tab 或切换 conversation 时自动关闭，避免旧弹窗内容写入错误的 `remarks.json`。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- ✅ `L1` 已补充 tab 身份升级的纯逻辑测试，覆盖已有 tab 被补绑定 plan 身份元数据的回归场景。证据：`crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.test.ts`
- ✅ `H1` comment dialog 的上下文切换 effect 已改为仅在 tab / conversation 真正变化时收口，不再在打开弹窗后立即自我关闭。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- ✅ `M3` `View/Edit plan.md` 在复用已打开 tab 时会重读最新磁盘内容，避免 `Regenerate Plan` 后旧 tab 保持过期草稿。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- ✅ `M4` 从 Files 面板直接打开匹配 `planPath` 的 `plan.md` 时，现会自动解析 plan 工件身份并沿用计划专用保存路径。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`、`crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.ts`
- ✅ `M5` 删除 conversation 后，`WorkspacePage` 现会自动清理与该 conversation 绑定的 plan tabs，避免已删除计划工件继续留在顶部 tab 中。证据：`crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`、`crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.ts`、`crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.test.ts`

---

## Acceptance Criteria Verification

| Area | Status | Evidence |
|------|--------|----------|
| Story 5-20-1 | Approved | `runtimeStore` / `appStore` 测试通过；TypeScript 类型检查通过；review findings 已清零 |
| Story 5-20-2 | Approved | `WorksPage.tsx` / `PlanControlPanel.tsx` 行为已对齐最新产品语义；`tsc` / `eslint` 通过；`planPanelLogic.test.ts` 与 `PlanControlPanel.test.tsx` 通过；review findings 已清零 |
| Story 5-20-3 | Approved with Residual Test Risk | `WorkspacePage.tsx` 已完成顶部 tab 复用、comment 弹窗、remark 持久化和手工保存同步；复审未发现新的实现级问题；剩余风险仅为组件/集成级测试覆盖偏薄 |
| Story 5-20-4 | Approved with Residual Test Risk | `planExecutionLogic.ts` / `WorksPage.tsx` / `runtimeStore.ts` / `fileSystemToolHost.ts` 已关闭前一轮 4 个实现级问题，并完成执行态 follow-up：引入受控 `plan.*` 状态迁移工具、执行态影子计划文档硬约束、`plan:state-changed` 后端推送与 `WorksPage` 集成回归；剩余风险仅为 `persistPlanState()` 失败路径缺少组件/集成级自动化覆盖 |

---

## Cross-Story Risk Focus

实施后重点关注以下跨 Story 问题：

1. **元数据漂移**
   前端 `Conversation` 与后端 `ConversationMetadata` 字段不一致。

2. **路径漂移**
   某些代码仍使用旧路径方案，如 `@state/plans/<conversationId>/...`。

3. **上下文污染**
   备注或计划内容被重复注入，导致 prompt 冗余。

4. **职责混乱**
   把 Plan 进度写入聊天消息、execution logs 或影子 `@project/*计划.md` 文档，而不是 live `plan.state.json`。

5. **清理不完整**
   conversation 删除或项目清理漏删 Plan 工件。

---

## Review Outcome

`Approved with Residual Test Risk`

简短结论：
- Story `5-20-1` 的 review follow-ups 已完成，项目级 Plan 工件、metadata 持久化/恢复、conversation 删除与 orphan project 清理已具备可验证证据。
- Story `5-20-2` 的 review follow-ups 已完成，右侧 Plan 控制面板与最新产品语义保持一致，stale state / regenerate semantics / empty execution guard 均已整改。
- Story `5-20-4` 的前一轮 4 个实现级问题和执行态 follow-up 已复核关闭；当前执行态通过受控 `plan.*` 工具推进 live `plan.state.json`，并由后端向前端推送 `plan:state-changed`，因此本 Story Group 当前可判定为通过，但仍保留 `persistPlanState()` 失败路径的残余测试风险说明。

---

## Related Validation Reports

- [validation-report-story-5-20-1.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/validation-report-story-5-20-1.md)
- [validation-report-story-5-20-2.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/validation-report-story-5-20-2.md)
- [validation-report-story-5-20-3.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/validation-report-story-5-20-3.md)
- [validation-report-story-5-20-4.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/validation-report-story-5-20-4.md)
