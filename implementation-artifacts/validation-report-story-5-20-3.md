# Validation Report: Story 5-20-3 (Revalidated)

**Story**: 5-20-3 Plan Tab Editing and Remarks  
**Validation Date**: 2026-03-26  
**Status**: ✅ **APPROVED WITH RESIDUAL TEST RISK** (story closed in sprint tracking)

---

## Story Structure Validation

| Criterion | Status | Notes |
|:---|:---:|:---|
| Overview / Goal | ✅ | 明确聚焦 `plan.md` 顶部 Tab 编辑和备注回流优化 |
| Acceptance Criteria | ✅ | 覆盖 Tab 打开、预览审阅、文档保存、备注写入、Chat 注入，以及 `Regenerate Plan` 输入绑定 |
| Delivery Scope by Layer | ✅ | Backend / Frontend / Integration 职责完整 |
| Risks / Mitigations | ✅ | 覆盖 Tab 冲突、备注膨胀、上下文过长 |
| Acceptance Checklist | ✅ | 适合作为开发完成核查表 |
| Task Breakdown | ✅ | 已可直接进入拆工实施 |

---

## Alignment Check

### 1. Alignment with `prd.md` ✅
- FR-PLAN-03、FR-PLAN-04、FR-PLAN-05 直接对应本 Story。
- NFR-PLAN-04 要求前后端一体可测试，本 Story 已具备清晰的集成边界。

### 2. Alignment with Sub PRD ✅
- 对应子 PRD 的 AC-4、AC-5。
- 与“备注先做结构化列表，不做行级锚点”完全一致。

### 3. Alignment with Design ✅
- 设计里明确了 `plan.md` 通过顶部 Tab 打开，不新增页面。
- 备注使用结构化 remark list，而不是内嵌富评论系统。
- 顶部 Tab 中采用“预览 + 评论弹窗”审阅区，符合最新交互方向。

### 4. Alignment with Tech Spec ✅
- 与 Workspace 顶部 alias tab 打开能力一致。
- 与 Chat context injection 的技术方向一致。

---

## Dependency Analysis

| Dependency | Status | Notes |
|:---|:---:|:---|
| Story 5-20-1 | ✅ | 需要稳定的 Plan 工件路径 |
| Story 5-20-2 | ✅ | 需要右侧 Plan 面板提供入口 |
| Existing MarkdownEditor | ✅ | 已有成熟编辑器，可直接复用 |
| Chat context builder chain | ✅ | 已有上下文注入机制，可扩展 |

---

## Gap Analysis

| Gap ID | Description | Severity | Recommendation |
|:---|:---|:---:|:---|
| G1 | 备注录入 UI 交互细节尚未完全定型 | Low | 用最小表单先落地，不阻塞 Story |
| G2 | 注入 Chat 的 Plan/备注内容量未精确定义 | Medium | 开发时限制为摘要 + 当前备注列表 |
| G4 | `Regenerate Plan` 与聊天框草稿绑定后，需防止误消费空输入 | Low | 开发时定义空输入下的禁用或降级策略 |
| G3 | `plan.md` 是否展示 remarks 镜像区块未固定 | Low | 保持 `remarks.json` 为唯一事实来源即可 |

---

## Recommendations

1. 不要在本 Story 引入复杂评论系统。
2. Tab 打开逻辑必须完全复用现有文件 Tab 机制，避免造第二套编辑器状态。
3. 预览 + 备注输入应尽量轻量，不要演变成独立审批系统。
4. 上下文注入要做摘要控制，避免把整个历史版本堆进 prompt。

---

## Verdict

**✅ Story 5-20-3 已完成当前范围实现，并可进入评审。**

该 Story 当前已具备：
- 顶部 `plan.md` tab 打开与复用闭环
- 预览悬浮 comment 弹窗与 `remarks.json` 持久化闭环
- `plan.md` 手工编辑保存与状态同步闭环
- 返回 Chat 后继续优化的上下文闭环

---

## Post-Implementation Addendum (2026-03-26)

| Check | Status | Evidence |
|:---|:---:|:---|
| `View/Edit plan.md` opens top tab | ✅ | `WorksPage.tsx` 继续通过 `openAliasFileTab()` 打开 `plan.md`，`WorkspacePage.tsx` 负责顶部 file tab 管理 |
| Existing `plan.md` tab is reused | ✅ | `workspaceTabLogic.ts` 中的 `upsertWorkspaceFileTab()` 负责同路径 tab 复用；`workspaceTabLogic.test.ts` 已覆盖 |
| Review surface + hover comment dialog | ✅ | `WorkspacePage.tsx` 的 `Plan Review` 区已支持 hover 浮出 `Add Comment` 并弹出评论对话框 |
| Structured remark persistence | ✅ | `addPlanRemark()` / `deletePlanRemark()` 持久化 `remarks.json`，并通过 comment 卡片即时刷新 |
| Manual `plan.md` edit and save | ✅ | `saveFileTab()` 在保存 plan tab 时改走 `savePlanDraft()`，不再只做裸 `writeFile()` |
| Manual save preserves existing remarks | ✅ | `savePlanDraft(clearRemarks: false)` 场景已由 `runtimeStore.test.ts` 覆盖 |
| Chat context can consume latest plan + remarks | ✅ | `sendChat` 的 `PLAN_CONTEXT` 已读取最新 `plan.md` 与 `remarks.json`；Story 5-20-2 / 5-20-3 集成语义一致 |
| No chat command protocol for state transitions | ✅ | 当前实现未引入聊天内控制命令；状态动作继续由 UI 触发 |
| Type safety | ✅ | `./node_modules/.bin/tsc --noEmit --pretty false` passed |
| Targeted lint | ✅ | `./node_modules/.bin/eslint src/pages/WorkspacePage/WorkspacePage.tsx src/pages/WorkspacePage/workspaceTabLogic.ts src/pages/WorkspacePage/workspaceTabLogic.test.ts electron/stores/runtimeStore.test.ts` passed |
| Workspace tab logic tests | ✅ | `./node_modules/.bin/vitest run src/pages/WorkspacePage/workspaceTabLogic.test.ts` passed |
| RuntimeStore remark preservation tests | ✅ | `./node_modules/.bin/vitest run electron/stores/runtimeStore.test.ts` passed |

### Addendum Notes

1. 本 Story 保持了“顶部 Tab 是唯一文档工件编辑入口，右侧面板不承载正文”的产品约束。
2. `plan.md` 的手工编辑保存现与 Runtime 里的 plan draft 持久化模型对齐，避免正文、状态、任务之间漂移。
3. 备注仍然独立保存在 `remarks.json`，只有 `Regenerate Plan` 成功应用后才在 Story `5-20-2` 的路径中清空。
4. `plan.md` tab 已绑定自身的 `conversationId / planPath / remarksPath`，切换左侧 conversation 后再返回旧 tab 保存，仍会写回原 conversation 的 plan 工件。
5. comment dialog 会在切换 tab 或切换 conversation 时自动关闭，避免旧弹窗内容保存到错误的 `remarks.json`。
6. `View/Edit plan.md` 在 tab 已打开时会重新读取最新 `plan.md`，避免聊天区 `Regenerate Plan` 后旧 tab 长时间停留在过期内容。
7. 从 Files 面板直接打开的 `plan.md` 现会自动绑定 plan 工件身份，保存与备注行为与右侧入口保持一致。
8. 删除 conversation 后，与其绑定的 `plan.md` tab 会自动从顶部 tab 列表移除，不再保留悬空计划工件编辑页。
9. 当前仍缺少 `WorkspacePage` 级别的组件/集成测试，这一项保留为评审阶段的残余测试缺口，但不阻塞本轮代码修复验证。

---

## Cross-Reference Links

- Story: [5-20-3-plan-tab-editing-and-remarks.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-3-plan-tab-editing-and-remarks.md)
- Tech Spec: [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md)
- Design: [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md)
- Dependency Story 1: [5-20-1-project-scoped-plan-storage-and-session-metadata.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md)
- Dependency Story 2: [5-20-2-chat-plan-panel-and-preview.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-2-chat-plan-panel-and-preview.md)
- Epic: [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md)
