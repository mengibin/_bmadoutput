# Validation Report: Story 5-20-2 (Revalidated)

**Story**: 5-20-2 Chat Plan Panel and Controls  
**Validation Date**: 2026-03-25  
**Status**: ✅ **APPROVED** (story closed in sprint tracking)

---

## Story Structure Validation

| Criterion | Status | Notes |
|:---|:---:|:---|
| Overview / Goal | ✅ | 聚焦右侧 Chat 详情区改造成 Plan 控制面板 |
| Acceptance Criteria | ✅ | 已覆盖 switch 式 Plan 开关、空白草稿初始化和收敛后的按钮集合 |
| Delivery Scope by Layer | ✅ | 前后端与集成责任边界清晰 |
| Risks / Mitigations | ✅ | 覆盖信息过载、正文展示误解和缺失文件降级 |
| Acceptance Checklist | ✅ | 开发完成可直接核对 |
| Task Breakdown | ✅ | 已拆到可执行粒度 |

---

## Alignment Check

### 1. Alignment with `prd.md` ✅
- FR-PLAN-01、FR-PLAN-03、FR-PLAN-07 与本 Story 直接相关。
- NFR-PLAN-05 明确要求不能破坏现有 Chat 行为，本 Story 保持顶层 `chat` 不变，符合要求。

### 2. Alignment with Sub PRD ✅
- 对应子 PRD 的 AC-1、AC-2、AC-3、AC-7。
- 保持“Plan 是 Chat 子模式”的产品定位一致，没有偏移成第四种会话类型。

### 3. Alignment with Design ✅
- 与 [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md) 中定义的 4 种 Plan 面板状态一致。
- 明确使用右侧详情区承载，而不是另起页面。

### 4. Alignment with Tech Spec ✅
- 与 `WorksPage / ConversationView` 改造范围一致。
- 与 `plan.state.json` 状态读取和右侧控制区技术方向一致。

---

## Dependency Analysis

| Dependency | Status | Notes |
|:---|:---:|:---|
| Story 5-20-1 | ✅ | 需要先有 Plan 工件和会话 metadata |
| Existing Chat details panel | ✅ | 现有静态 Chat 详情区可直接替换 |
| plan.md read capability | ⚠️ Pending | 需在实施时接入读取接口或复用文件读取能力 |

---

## Gap Analysis

| Gap ID | Description | Severity | Recommendation |
|:---|:---|:---:|:---|
| G1 | `Reinitialize Plan` 降级入口在 Design 提到，但 Story AC 未强制 | Low | 可作为实现增强项，不阻塞开发 |
| G2 | 状态标签视觉方案未写到 UI 规格级别 | Low | 开发时按 Design 中的状态卡片方案执行即可 |
| G3 | `Resume Execution` 的可见性策略需和执行态规则保持一致 | Low | 开发时以 Tech Spec 的允许状态作为唯一依据 |

---

## Recommendations

1. 本 Story 先实现“切换 + 读 + 展示”，不要抢先做编辑。
2. 右侧仅展示控制和状态，不展示 `plan.md` 正文。
3. 缺失文件时保持 Chat 可用，先降级提示，不要阻断整个会话。

---

## Verdict

**✅ Story 5-20-2 已完成当前范围实现，并可进入评审。**

该 Story 当前已经具备：
- 右侧 Chat Plan 控制面板实现
- switch 式 `Plan` 开关和状态徽标
- `View/Edit plan.md`、`Regenerate Plan`、`Approve and Execute`、按条件显示的 `Resume Execution`
- 无 `Approve Plan`、无右侧正文预览
- 基于 Story `5-20-1` 的空白草稿初始化

---

## Post-Implementation Addendum (2026-03-25)

| Check | Status | Evidence |
|:---|:---:|:---|
| Switch-style Plan toggle | ✅ | `WorksPage.tsx` 右侧控制区已采用 `role=\"switch\"` 的 `Plan` 开关 |
| Status badge rendering | ✅ | `plan.state.json.status` 对应 `plan-status-badge` 样式 |
| Right-side controls only | ✅ | 右侧不再展示 `plan.md` 正文，仅保留控制按钮和执行信息 |
| `Approve Plan` removed | ✅ | 右侧动作区只保留 `View/Edit plan.md`、`Regenerate Plan`、`Approve and Execute`、条件性 `Resume Execution` |
| Empty draft compatibility | ✅ | Story `5-20-1` 已将 `plan.md` 初始化为空字符串，本 Story UI 不依赖预填模板正文 |
| Stale state guard on conversation switch | ✅ | `WorksPage.tsx` 为 artifact 读取增加 request guard，并在每次切换时先清空旧的 `planMarkdown / planState` |
| `Regenerate Plan` input semantics | ✅ | 当前输入框有草稿时，`Regenerate Plan` 会直接发送该草稿；无草稿时才回退到默认修订 prompt |
| `Regenerate Plan` remark semantics | ✅ | `sendChat` 路径会继续注入 `PLAN_CONTEXT` 中的 remarks；优化成功写回 `plan.md` 后会清空已消费的 `remarks.json` |
| Input preservation on failed regenerate | ✅ | `sendMessageContent()` 仅在成功路径清空输入，失败或中止时保留当前草稿 |
| Remark anchor propagation | ✅ | `PLAN_CONTEXT` 中的 remark 摘要现包含 `section heading / excerpt`，不再只保留纯文本内容 |
| Atomic plan draft persistence | ✅ | `Regenerate Plan` 现通过统一的 draft 持久化入口一次性写入 `plan.md`、`plan.state.json` 和已消费 remarks，失败时回滚已写入文件 |
| Input preservation on plan-save failure | ✅ | 仅当 `plan.md` 写回成功且已消费 remarks 清理成功时才清空输入；保存失败时保留当前草稿 |
| Full plan injection in `PLAN_CONTEXT` | ✅ | `PLAN_CONTEXT` 现直接注入完整当前 `plan.md`，不再只传前 12 行摘要 |
| Full unresolved remarks injection | ✅ | `PLAN_CONTEXT` 不再截断前 6 条 comments，当前所有 unresolved remarks 都会参与本轮优化 |
| Synthetic regenerate prompt suppression | ✅ | `Regenerate Plan` 作为 UI 动作不再向聊天记录追加内部英文修订指令或生成出的 plan 正文 |
| Empty draft execution guard | ✅ | `Approve and Execute` 仅在存在结构化任务时可用；否则给出错误提示并阻止进入 `executing` |
| Type safety | ✅ | `./node_modules/.bin/tsc --noEmit --pretty false` passed |
| Targeted lint | ✅ | `./node_modules/.bin/eslint electron/stores/runtimeStore.ts electron/stores/runtimeStore.test.ts electron/main.ts electron/preload.ts electron/electron-env.d.ts src/pages/WorksPage/WorksPage.tsx` passed |
| RuntimeStore rollback tests | ✅ | `./node_modules/.bin/vitest run electron/stores/runtimeStore.test.ts` passed |
| Logic-level regression tests | ✅ | `./node_modules/.bin/vitest run src/pages/WorksPage/planPanelLogic.test.ts` previously passed and remains applicable |
| Component-level panel tests | ✅ | `./node_modules/.bin/vitest run src/pages/WorksPage/PlanControlPanel.test.tsx` passed |

### Addendum Notes

1. 本 Story 的实现范围已与最新产品语义对齐：
   - `Plan Mode` 打开后默认进入当前 conversation 的 Plan 控制面板
   - 右侧不承担正文预览职责
   - `Approve Plan` 已收敛出产品范围
2. 2026-03-25 code review 提出的 3 个实现问题已完成整改：
   - 会话切换时的 stale plan 状态
   - `Regenerate Plan` 未使用输入草稿
   - 空草稿也能进入执行
3. 本 Story 已进一步收敛 `Regenerate Plan` 语义：
   - 输入存在时，输入与 remarks 一起参与优化
   - 输入为空但 remarks 存在时，仍可基于 remarks 优化
   - 优化成功后自动清空已消费 remarks
4. 第二轮 review follow-ups 已全部关闭：
   - 失败请求不再清空输入草稿
   - `PLAN_CONTEXT` remark 摘要保留 section / excerpt 锚点
   - 面板按钮可见性与 `Approve Plan` 移除已由组件测试覆盖
5. 第三轮 review follow-ups 已全部关闭：
   - `Regenerate Plan` 改走原子化 plan draft 持久化路径，失败时回滚已写入的 plan 工件
   - `plan.md` 写回失败时不再错误清空输入草稿
   - `PLAN_CONTEXT` 现注入完整当前计划和全部 unresolved remarks，长计划后半部分与后续 comments 不会被截断
   - `Regenerate Plan` 不再把内部英文动作或生成出的 plan 正文写入用户聊天记录
6. 当前未发现阻塞 `review` 的残留问题。

---

## Cross-Reference Links

- Story: [5-20-2-chat-plan-panel-and-preview.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-2-chat-plan-panel-and-preview.md)
- Tech Spec: [tech-spec-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md)
- Design: [design-5-20-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md)
- Dependency Story: [5-20-1-project-scoped-plan-storage-and-session-metadata.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md)
- Epic: [epics-runtime-chat-plan-mode.md](/Users/mengbin/code/GPT/CrewAgent/_bmad-output/epics-runtime-chat-plan-mode.md)
