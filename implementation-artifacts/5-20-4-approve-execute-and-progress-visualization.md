# Story 5-20-4: Approve, Execute, and Progress Visualization

## Overview
**Epic**: 5 – Runtime Chat Plan Mode
**Priority**: High
**Status**: `done`

## Goal
让用户可以在 Chat 中批准并执行计划，并通过独立的 Plan 状态模型查看任务执行进度。

## Business Value
- **计划到执行闭环**：用户不是只生成计划，而是能把计划推进到执行态。
- **透明可跟踪**：执行过程有明确的任务状态和完成比例。
- **状态可恢复**：用户重启应用或切换会话后仍能看到当前进度。

## Acceptance Criteria

### 1. 同意并执行入口
- **Given** 当前 Plan 处于 `drafting` 或 `reviewing`
- **When** 用户点击 `同意并执行 Plan`
- **Then** `plan.state.json.status` 更新为 `executing`
- **And** 记录 `executingAt` 时间。

### 2. 任务进度模型
- **Given** `plan.state.json.tasks[]` 存在
- **When** 详情区渲染
- **Then** 显示：
  - 任务列表
  - 每个任务状态
  - 完成比例
  - 当前阶段说明

### 2A. 执行中等待用户决策
- **Given** 某个任务执行中需要用户补充决策
- **When** 当前任务被标记为 `waiting_user`
- **Then** Plan 详情区显示该等待状态和等待原因。

- **And** 详情区同时提供：
  - `Submit Decision and Continue`
  - `Revise Plan Instead`

- **Given** 当前存在 `waiting_user` 的任务
- **When** 用户在 Chat 中补充内容
- **Then** 默认将该内容解释为当前任务的决策输入
- **And** 执行继续当前任务，而不是重新编写整份计划。

- **Given** 当前存在 `waiting_user` 的任务
- **When** 用户在聊天框中输入内容并点击 `Regenerate Plan`
- **Then** 当前输入应被解释为计划优化要求
- **And** 系统进入计划修订，而不是继续当前任务。

### 2B. Resume Execution
- **Given** Plan 已经进入 `executing` 或 `blocked`
- **When** 当前不存在 `waiting_user` 任务且用户点击 `Resume Execution`
- **Then** 系统继续当前执行链路
- **And** 若没有活动任务，则优先激活下一个可执行 task。

### 3. 进度恢复
- **Given** Plan 正在执行
- **When** 用户关闭并重新打开应用
- **Then** 系统从 `plan.state.json` 恢复执行状态与任务进度。

### 4. 与 workflow progress 隔离
- **Given** 现有 workflow progress 仍存在
- **When** Plan 进入执行态
- **Then** Plan 进度不依赖现有 workflow progress hook
- **And** 使用独立的 Plan 状态模型。

### 5. 阻塞与完成
- **Given** 任务全部完成
- **When** 状态汇总
- **Then** `plan.state.json.status = completed`。

- **Given** 执行无法继续
- **When** 状态被标记为阻塞
- **Then** 详情区显示 `blocked` 状态和阻塞说明。

### 6. 执行完成后的继续聊天
- **Given** Plan 已完成
- **When** 用户继续普通聊天
- **Then** 默认语义是开启新的计划版本
- **And** 系统不覆盖当前已完成版本。

- **Given** 新版本被创建
- **When** 新一轮计划开始
- **Then** `version + 1`
- **And** 执行任务列表重新初始化。

## Technical Components / Changes

1. **Plan State Updater**
   - 更新执行状态、任务状态与时间戳
2. **Plan Progress UI**
   - 进度条
   - 任务清单
   - 状态卡片
3. **恢复逻辑**
   - 从 `plan.state.json` 重新加载

## Delivery Scope by Layer

### Backend Scope

- 支持将 `plan.state.json.status` 推进到 `executing`。
- 支持持久化任务数组、阶段、完成比例相关字段。
- 支持恢复读取执行状态。
- 支持 `waiting_user` 任务状态以及用户补充输入后的继续执行语义。
- 支持 `completed` 后根据下一次聊天输入开启新版本。
- 在执行态仅暴露受控的 `plan.*` 状态迁移工具，而不是允许 LLM 直接编辑 `plan.state.json`。
- 在执行态禁止创建 `@project/*计划.md` / `@project/*任务.md` / `@project/*plan.md` / `@project/*task.md` 这类影子文档来追踪进度。
- 后端在执行态状态变更后推送 `plan:state-changed`，使前端 UI 与 live `plan.state.json` 保持同步。

### Frontend Scope

- 在 Plan 面板中显示执行按钮、任务清单、进度条和阶段状态。
- 在 Plan 面板中提供显式 `Resume Execution` 按钮。
- 根据任务状态重新计算完成比例。
- 展示 `completed` / `blocked` 等终态。
- 在等待用户决策时展示明确的等待状态和决策动作。
- 去除 `Approve Plan`，保留 `Approve and Execute` 作为唯一执行入口。

### Integration Scope

- 点击执行后，按钮动作、状态文件和右侧 UI 同步变化。
- 重启应用后，恢复的进度与磁盘状态一致。
- 执行中等待用户输入后，Chat 输入返回到当前任务执行链路。
- `completed` 后下一次聊天输入触发新版本计划，而不是覆盖旧版本。

## Dependencies
- 依赖 Story 5-20-1 的 `plan.state.json`
- 依赖 Story 5-20-2 的 Plan 面板承载区域
- 依赖 Story 5-20-3 的计划文档与备注闭环

## Related Artifacts
- PRD: `_bmad-output/prd-runtime-chat-plan-mode.md`
- Epic: `_bmad-output/epics-runtime-chat-plan-mode.md`
- Design: `_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md`

## Verification Plan

### Manual Verification
1. 审阅态点击执行，状态切换到 `executing`。
2. 手工更新任务状态，确认 UI 进度同步变化。
3. 重启应用，确认状态恢复。
4. 任务全部完成后，状态变为 `completed`。
5. 将任务置为 `waiting_user`，确认等待卡片和两个动作按钮可见。
6. 在非等待状态点击 `Resume Execution`，确认执行链路继续。
7. 在 `waiting_user` 时点击 `Regenerate Plan`，确认当前聊天框输入进入计划修订流。
8. 在 `completed` 后发送新消息，确认创建 `version + 1`。

### Automated Tests
- Plan 状态更新单测。
- 进度计算与渲染测试。
- 重启恢复测试。
- 执行态 `plan.*` 工具可见性与状态推进测试。
- `plan:state-changed` 推送后的 `WorksPage` 集成测试。
- 执行态影子计划文档拦截测试。

## Risks / Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | 计划执行态语义与现有 workflow progress 混淆 | 明确使用独立的 `plan.state.json` 作为唯一进度来源 |
| R2 | 任务状态更新逻辑不一致导致百分比错误 | 在 Tech Spec 中固定聚合规则并加单测 |
| R3 | 执行中断后恢复信息不足 | 在状态文件中记录 `updatedAt`、`executingAt` 和任务状态 |
| R4 | 用户补充执行决策时被误判为重新规划 | 规定 `waiting_user` 状态下的默认语义是继续当前 task，只有点击 `Regenerate Plan` 才进入修订 |

## Acceptance Checklist

- [x] 点击 `同意并执行 Plan` 后状态进入 `executing`
- [x] 执行时间戳被记录
- [x] 任务列表和任务状态可见
- [x] 完成比例计算正确
- [x] `completed` / `blocked` 状态可见且有明确展示
- [x] 重启应用后仍可恢复执行进度
- [x] Plan 进度不依赖现有 workflow progress hook
- [x] 非等待状态存在显式 `Resume Execution` 按钮
- [x] `waiting_user` 任务收到用户输入后继续当前 task
- [x] `waiting_user` 任务存在 `Submit Decision and Continue` 与 `Revise Plan Instead`
- [x] `waiting_user` 时点击 `Regenerate Plan` 会把当前聊天框输入解释为计划优化要求
- [x] `completed` 后下一次聊天输入会开启 `version + 1`，而不是覆盖旧计划
- [x] 面板中不包含 `Approve Plan`

## Sequencing Notes

- 本 Story 最后实施，避免前面 Story 尚未稳定时过早引入执行态复杂度。
- 若执行引擎暂时不联动实际自动化动作，也应先把状态推进和 UI 闭环做好。

## Tasks / Subtasks
- [x] Backend-1 定义任务状态与百分比聚合规则
- [x] Backend-2 增加执行态推进与状态文件更新
- [x] Backend-3 增加恢复读取逻辑
- [x] Backend-4 定义 `waiting_user` 任务的输入回流与继续执行规则
- [x] Backend-5 定义 `completed` 后的新版本 rollover 规则
- [x] Backend-6 增加受控 `plan.*` 执行工具与状态迁移校验
- [x] Backend-7 增加执行态影子计划/任务文档硬约束与 deliverable 白名单
- [x] Backend-8 增加 `plan:state-changed` 推送链路
- [x] Frontend-1 接入 `同意并执行 Plan` 按钮
- [x] Frontend-2 实现任务清单、进度条、阶段卡片
- [x] Frontend-3 增加 `completed` / `blocked` 等终态 UI
- [x] Frontend-4 增加 `Resume Execution` 按钮
- [x] Frontend-5 增加 `waiting_user` 任务展示、决策输入和等待动作
- [x] Frontend-6 移除 `Approve Plan` 并让 `Regenerate Plan` 读取当前聊天框草稿
- [x] Frontend-7 订阅 `plan:state-changed` 并优先采用后端推送的 live plan state
- [x] Integration-1 验证点击执行后 UI 与状态文件同步
- [x] Integration-2 验证重启后进度恢复
- [x] Integration-3 验证 `waiting_user` 任务收到用户输入后继续执行
- [x] Integration-4 验证 `Resume Execution` 仅在允许状态下出现或可点击
- [x] Integration-5 验证 `waiting_user` 时点击 `Regenerate Plan` 会进入计划修订
- [x] Integration-6 验证 `completed` 后下一次聊天输入会开启新版本
- [x] Integration-7 验证后端 `plan.*` 工具已推进状态时，前端不会再做本地二次 fallback
- [x] Test-1 补状态推进与聚合规则单测
- [x] Test-2 补恢复场景与终态渲染测试
- [x] Test-3 补执行态工具、影子文档拦截与 `WorksPage` 集成回归

## File List

- `_bmad-output/implementation-artifacts/5-20-4-approve-execute-and-progress-visualization.md`
- `_bmad-output/implementation-artifacts/validation-report-story-5-20-4.md`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/PlanControlPanel.tsx`
- `crewagent-runtime/src/pages/WorksPage/planExecutionLogic.ts`
- `crewagent-runtime/src/pages/WorksPage/planExecutionLogic.test.ts`
- `crewagent-runtime/src/pages/WorksPage/ConversationView.integration.test.tsx`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/shared/planExecutionState.ts`

## Senior Developer Review (AI)

### Review Outcome

- Date: 2026-03-27
- Outcome: Approved with Residual Test Risk
- Issues: 0
- Acceptance Criteria: PASS

### Action Items

- [x] [AI-Review][RESOLVED] `Resume Execution` 现在在 `waiting_user` 场景下仅恢复当前 task 为 `in_progress` 并保留用户决策备注，不再在收到 assistant 文本后直接把当前 task 标记为 `completed`。（`crewagent-runtime/src/pages/WorksPage/planExecutionLogic.ts:119-143`, `crewagent-runtime/src/pages/WorksPage/planExecutionLogic.test.ts:55-98`）
- [x] [AI-Review][RESOLVED] `derivePlanStatusFromTasks()` 现在在无 active/waiting task 时优先返回 `blocked`，不会再被后续 `pending` task 错误掩盖。（`crewagent-runtime/src/pages/WorksPage/planExecutionLogic.ts:83-95`, `crewagent-runtime/src/pages/WorksPage/planExecutionLogic.test.ts:100-107`）
- [x] [AI-Review][RESOLVED] `handleResumeExecution()` 现在先请求 assistant 响应、再持久化 `plan.state.json`、最后才写入 user/assistant 消息，避免聊天记录先于 Plan 真源前进。（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx:1354-1402`）
- [x] [AI-Review][RESOLVED] `saveConversationPlanDraft()` 在 live plan 已完成时会先归档旧版本，再创建 `version + 1` 的新 draft，`plan.md` 顶部 Tab 保存路径不再覆盖 completed 版本。（`crewagent-runtime/electron/stores/runtimeStore.ts:3078-3145`, `crewagent-runtime/electron/stores/runtimeStore.test.ts:1457-1495`）
- [x] [AI-Review][RESOLVED] 执行态现在通过受控 `plan.*` 工具推进 live `plan.state.json`，不再依赖“最后一条 assistant 文本”作为唯一状态推进来源；状态更新后会通过 `plan:state-changed` 推送回前端，`WorksPage` 优先采用后端推送的 live state。（`crewagent-runtime/electron/services/fileSystemToolHost.ts`, `crewagent-runtime/electron/main.ts`, `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`, `crewagent-runtime/src/pages/WorksPage/ConversationView.integration.test.tsx`）
- [x] [AI-Review][RESOLVED] 执行态现在硬拦截 `@project/*计划.md` / `@project/*任务.md` 等影子计划文档；只有 `Deliverables` 已声明的目标文件允许继续写入，避免 Plan UI 与实际执行真源分叉。（`crewagent-runtime/electron/services/fileSystemToolHost.ts`, `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`）
- [ ] [AI-Review][Residual Risk] 目前仍缺少 `WorksPage` 级自动化测试去模拟“assistant 返回成功但 `persistPlanState()` 失败”的前端失败路径；本次复审确认实现逻辑正确，但该场景仍主要依赖代码审阅而非组件/集成级回归。

### Review History

- 2026-03-27: Approved with Residual Test Risk（执行态 follow-up 已补齐：引入受控 `plan.*` 工具、阻止影子计划文档、后端 `plan:state-changed` 推送和 `WorksPage` 级集成回归；剩余风险仍仅为 `persistPlanState()` 失败路径缺少组件/集成级自动化覆盖）
- 2026-03-27: Approved with Residual Test Risk（上轮 4 个实现级问题已复核关闭；剩余风险仅为 `WorksPage` 持久化失败路径缺少组件/集成级自动化覆盖）
- 2026-03-27: Changes Requested（H1: Resume Execution 会虚假完成当前任务；M1: blocked 状态推导优先级错误；M2: 聊天记录与 `plan.state.json` 持久化非原子；M3: completed 版本可被 plan tab 保存路径覆盖）
- 2026-03-28: 按 BMAD sprint closing 约定关闭 Story；状态更新为 `done`，残余测试风险继续保留在本评审记录中。
