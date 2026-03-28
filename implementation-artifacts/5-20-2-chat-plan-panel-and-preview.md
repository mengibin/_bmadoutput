# Story 5-20-2: Chat Plan Panel and Controls

## Overview
**Epic**: 5 – Runtime Chat Plan Mode
**Priority**: High
**Status**: `done`

## Goal
将当前 Chat 详情区从静态说明替换为可操作的 Plan 面板，在不离开 Chat 的情况下完成状态判断和执行入口控制。

## Business Value
- **更强的可见性**：用户立即知道当前会话是否处于 Plan 阶段。
- **降低切换成本**：不用先打开文件页才能判断计划情况。
- **为执行做好准备**：在同一位置集中放置查看、优化、执行入口。

## Acceptance Criteria

### 1. Chat 详情区切换
- **Given** 当前会话 `activeType = 'chat'`
- **When** `chatSubmode = 'default'`
- **Then** 右侧详情区显示普通 Chat 信息。

- **Given** 当前会话 `activeType = 'chat'`
- **When** `chatSubmode = 'plan'`
- **Then** 右侧详情区显示 Plan 控制面板。

### 2. Plan 面板内容
- **Given** Plan 模式已开启
- **When** 面板渲染
- **Then** 至少显示：
  - `Plan` 开关（switch，而不是 checkbox）
  - 当前计划状态标签
  - `查看/编辑 plan.md`
  - `Regenerate Plan`
  - `同意并执行 Plan`
  - `Resume Execution`（在允许时显示或可点击）
  - 当前进度概览区

### 2B. Regenerate Plan 语义
- **Given** Plan 模式已开启且存在 `remarks.json` comments / remarks
- **When** 用户在聊天输入框中输入新的优化要求并点击 `Regenerate Plan`
- **Then** 本次优化必须同时考虑：
  - 当前聊天输入内容
  - 当前 `plan.md`
  - 尚未消费的 comments / remarks

- **Given** 聊天输入框为空但存在未消费 comments / remarks
- **When** 用户点击 `Regenerate Plan`
- **Then** 系统默认基于当前 `plan.md` 的 comments / remarks 进行优化

- **Given** 本次 Plan 优化成功写回新的 `plan.md`
- **When** 优化流程完成
- **Then** 当前已消费的 comments / remarks 从 `remarks.json` 中清空

### 2A. 初始空白草稿
- **Given** 用户首次打开 Plan 开关
- **When** Plan 面板首次可见
- **Then** `plan.md` 对应为空草稿
- **And** 面板不依赖预填默认模板正文
- **And** 用户接下来的聊天输入默认用于编写或修改当前计划。

### 3. 状态数据来源
- **Given** `plan.state.json` 已存在
- **When** 面板读取数据
- **Then** 状态与阶段来自 `plan.state.json`。

### 4. 状态区分
- **Given** `plan.state.json.status` 不同
- **When** 面板渲染
- **Then** `drafting/reviewing/executing/completed/blocked/default` 有明确视觉区分。

## Technical Components / Changes

1. **WorksPage / ConversationView**
   - 替换现有 `Chat mode is ready` 详情区块
2. **Plan Panel UI**
   - 状态徽标
   - 操作按钮区
   - 进度概览区
   - switch 式 Plan 开关
3. **数据加载**
   - 读取 `plan.state.json`

## Delivery Scope by Layer

### Backend Scope

- 暴露读取 `plan.md` 和 `plan.state.json` 的能力。
- 保证缺省文件或损坏文件时返回可展示错误，而不是直接崩溃。

### Frontend Scope

- 替换 Chat 详情区静态说明。
- 渲染 Plan 开关、状态标签、按钮区、进度概览区。
- Plan 开关采用 switch 交互，不使用 checkbox。
- 根据 `plan.state.json.status` 渲染不同视觉状态。

### Integration Scope

- Plan 开关切换后，详情区 UI 与会话 metadata 同步。
- 状态数据与右侧控制区内容一致。

## Dependencies
- 依赖 Story 5-20-1 提供 Plan 工件与 metadata。
- 为 Story 5-20-3 打开编辑入口，为 Story 5-20-4 提供进度展示容器。

## Related Artifacts
- PRD: `_bmad-output/prd-runtime-chat-plan-mode.md`
- Epic: `_bmad-output/epics-runtime-chat-plan-mode.md`
- Design: `_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md`

## Verification Plan

### Manual Verification
1. Chat 普通模式下显示普通详情区。
2. 打开 Plan 开关后显示 Plan 面板。
3. 修改 `plan.state.json.status` 后状态标签和颜色正确变化。
4. 右侧不显示计划正文预览。
5. 首次开启时 `plan.md` 为空草稿。
6. 面板动作不包含 `Approve Plan`。
7. `Regenerate Plan` 在有输入时同时考虑输入和 remarks；在无输入但有 remarks 时仍可优化。
8. Plan 优化成功后会清空已消费 remarks。

### Automated Tests
- 组件测试：Plan 面板不同状态渲染。
- 集成测试：Plan 开关切换与状态读取。

## Risks / Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | 右侧详情区信息过载，影响可读性 | 只展示控制信息和状态，不展示计划正文或多余说明文案 |
| R2 | 用户误以为右侧内容就是可编辑文档 | 明确通过 `查看/编辑 plan.md` 进入文档编辑 |
| R3 | `plan.state.json` 缺失时 UI 空白 | 提供降级文案和 `Reinitialize Plan` 入口 |

## Acceptance Checklist

- [x] `chatSubmode = default` 时显示普通 Chat 详情区
- [x] `chatSubmode = plan` 时显示 Plan 面板
- [x] 面板包含 Plan 开关、状态标签、编辑按钮、`Regenerate Plan`、执行按钮、进度概览
- [x] 状态来自 `plan.state.json`
- [x] 右侧不显示 `plan.md` 正文预览
- [x] 不同生命周期状态有清晰视觉区分
- [x] Plan 开关使用 switch 交互
- [x] 初次启用时 `plan.md` 为空草稿
- [x] 面板中不包含 `Approve Plan`
- [x] `Regenerate Plan` 在有输入时同时考虑输入和 remarks
- [x] 仅当有输入或有 remarks 时允许触发 `Regenerate Plan`
- [x] Plan 优化成功后会清空已消费 remarks

## Sequencing Notes

- 本 Story 完成后，用户就能看见 Plan 工作台，但还不能完成真正的编辑闭环。
- `查看/编辑 plan.md` 按钮可以先连到占位逻辑，再由 5-20-3 完整实现。

## Tasks / Subtasks
- [x] Backend-1 提供 `plan.md` 和 `plan.state.json` 读取接口
- [x] Backend-2 定义缺省/错误响应格式
- [x] Frontend-1 在 ConversationView 中加入 Plan 模式判断
- [x] Frontend-2 实现 Plan 面板组件、switch 与状态卡片
- [x] Frontend-3 实现生命周期状态样式
- [x] Integration-1 验证 Plan 开关与详情区联动
- [x] Test-1 补状态渲染组件测试
- [x] Test-2 补右侧无预览展示的集成测试

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] 切换 Plan conversation 时先清空旧的 `planMarkdown / planState`，并为异步 artifact 读取增加 request guard，避免上一会话的 plan 状态短暂污染当前会话面板（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`）
- [x] [AI-Review][MEDIUM] `Regenerate Plan` 必须优先使用当前聊天输入框草稿，而不是固定发送通用 prompt，确保按钮语义与最新产品规则一致（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`, `crewagent-runtime/src/pages/WorksPage/planPanelLogic.ts`）
- [x] [AI-Review][MEDIUM] `Approve and Execute` 必须禁止空草稿 / 无结构化任务时直接进入 `executing`，并给出明确错误提示（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`, `crewagent-runtime/src/pages/WorksPage/planPanelLogic.ts`）
- [x] [AI-Review][MEDIUM] `Regenerate Plan` 请求失败或中止时必须保留当前输入草稿；当前实现会在 `finally` 中无条件清空输入，导致用户丢失尚未生效的修订指令（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`）
- [x] [AI-Review][MEDIUM] `PLAN_CONTEXT` 中的 remarks 不能只保留纯文本内容；当前注入丢失了 section heading / excerpt，导致针对特定段落的 comment 无法可靠映射到原计划位置（`crewagent-runtime/electron/main.ts`）
- [x] [AI-Review][LOW] 仍缺少右侧 Plan 控制面板的组件 / 集成自动化测试，当前只有纯逻辑 helper 测试，尚不足以覆盖按钮可见性和状态渲染（`crewagent-runtime/src/pages/WorksPage/PlanControlPanel.test.tsx`）
- [x] [AI-Review][MEDIUM] `Regenerate Plan` 在 `plan.md` 持久化失败时仍会清空输入框；修复后仅在计划写回和 remark 清理都成功时才清空输入（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`）
- [x] [AI-Review][MEDIUM] `PLAN_CONTEXT` 只注入计划摘要会导致长计划后半部分缺失；修复后改为注入完整当前 plan 内容（`crewagent-runtime/electron/main.ts`）
- [x] [AI-Review][LOW] `Regenerate Plan` 作为 UI 动作不应把内部英文指令写成用户消息；修复后该按钮走隐藏系统动作路径，不再污染聊天记录（`crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`）

## File List

- `_bmad-output/implementation-artifacts/5-20-2-chat-plan-panel-and-preview.md`
- `_bmad-output/implementation-artifacts/validation-report-story-5-20-2.md`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`
- `crewagent-runtime/src/pages/WorksPage/WorksPage.css`
- `crewagent-runtime/src/pages/WorksPage/PlanControlPanel.tsx`
- `crewagent-runtime/src/pages/WorksPage/PlanControlPanel.test.tsx`
- `crewagent-runtime/src/pages/WorksPage/planPanelLogic.ts`
- `crewagent-runtime/src/pages/WorksPage/planPanelLogic.test.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/vite-env.d.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `crewagent-runtime`: `./node_modules/.bin/tsc --noEmit --pretty false`
- `crewagent-runtime`: `./node_modules/.bin/eslint src/pages/WorksPage/WorksPage.tsx`

### Completion Notes List

- Chat 右侧详情区已替换为 Plan 控制面板，支持 switch 式 `Plan` 开关和状态徽标。
- 面板已移除右侧计划正文预览，仅保留 `View/Edit plan.md`、`Regenerate Plan`、`Approve and Execute` 和按状态可见的 `Resume Execution`。
- `Approve Plan` 已从右侧动作区移除。
- 右侧说明性标题和文案已压缩，面板聚焦控制动作、等待决策区和进度概览。
- 本 Story 依赖 Story `5-20-1` 的空白 `plan.md` 初始化语义，首次开启时不会写入默认模板正文。
- 已补最小逻辑单测，覆盖 `Regenerate Plan` 输入优先级和 `Approve and Execute` 可用性判断。
- `Regenerate Plan` 现会复用当前输入框内容，并依赖 chat 上下文中的 `remarks.json` 一并优化当前计划。
- 计划优化成功并写回 `plan.md` 后，会清空当前已消费的 remarks。
- `Regenerate Plan` 请求失败或中止时，不再丢失用户尚未生效的输入草稿。
- `PLAN_CONTEXT` 现会带上 remark 的 section heading / excerpt，保证 comment 锚点在优化时仍可定位。
- 已补最小面板渲染测试，覆盖按钮可见性和移除 `Approve Plan` 的回归。
- `Regenerate Plan` 改为通过原子化的 draft 持久化路径统一写入 `plan.md`、`plan.state.json` 和已消费 remarks，避免出现正文与状态半成功半失败的不一致。
- `Regenerate Plan` 只有在 `plan.md` 写回和 remark 清理都成功后才会清空输入框。
- `PLAN_CONTEXT` 现会注入完整当前 plan，并携带全部 unresolved remarks，而不再只传前 12 行摘要或截断前 6 条 comment。
- `Regenerate Plan` 触发的内部英文修订指令和生成出的 plan 正文都不再写入聊天记录。
- 已完成类型检查、针对性 lint 校验、纯逻辑单测和面板渲染测试。

## Change Log

- 2026-03-25: 完成右侧 Chat Plan 面板改造，移除 `Approve Plan` 与正文预览，统一为 switch 开关、状态徽标、动作区和进度概览；状态更新为 `review`。
- 2026-03-25: 根据 code review 结果完成 3 项整改：修复切会话时的 stale plan 状态、让 `Regenerate Plan` 使用当前输入草稿、阻止空草稿直接执行；新增 `planPanelLogic` 单测。
- 2026-03-25: 收敛 `Regenerate Plan` 语义：输入和 remarks 会同时参与优化；无输入但存在 remarks 时仍可优化；优化成功后自动清空已消费 remarks。
- 2026-03-25: 第二轮 code review 发现 2 个 MEDIUM 和 1 个 LOW 残留问题（输入失败丢失、remark 锚点丢失、UI 自动化缺口）；状态短暂回退为 `in-progress`。
- 2026-03-25: 完成第二轮 review follow-ups：请求失败保留输入、`PLAN_CONTEXT` remark 锚点补全、最小面板渲染测试补齐；状态恢复为 `review`。
- 2026-03-25: 第三轮 code review 发现 3 个残留问题（计划写回失败仍清空输入、`PLAN_CONTEXT` 仅注入摘要 / 截断 remarks、`Regenerate Plan` 污染聊天记录）；已通过原子化 plan draft 持久化、完整上下文注入和隐藏 UI 动作消息完成整改，并保持 `review` 状态。
- 2026-03-28: 按 BMAD sprint closing 约定关闭 Story；结合 story-group code review 中的 `Approved` 结论，将状态更新为 `done`。
