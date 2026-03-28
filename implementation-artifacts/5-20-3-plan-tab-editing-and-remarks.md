# Story 5-20-3: Plan Tab Editing and Remarks

## Overview
**Epic**: 5 – Runtime Chat Plan Mode
**Priority**: High
**Status**: `done`

## Goal
允许用户在顶部 Tab 中打开 `plan.md` 进行查看和编辑，并增加结构化备注，使计划文档成为真正可协作的工件，并能在返回 Chat 后继续优化。

## Business Value
- **文档化协作**：计划不再只是聊天内容，而是实际工件。
- **审阅可沉淀**：备注能独立保存，不污染主计划正文。
- **优化闭环**：编辑和备注能回流到 Chat 上下文，让 LLM 真正“接着改”。

## Acceptance Criteria

### 1. 顶部 Tab 打开 `plan.md`
- **Given** Plan 模式已开启
- **When** 用户点击 `查看/编辑 plan.md`
- **Then** Workspace 顶部打开一个 `plan.md` 文件 Tab
- **And** 若已打开，则切换到该 Tab，而不是重复创建。

### 1A. 顶部 Tab 审阅视图
- **Given** `plan.md` 已在顶部 Tab 打开
- **When** 用户查看计划
- **Then** 顶部 Tab 中提供当前计划的预览区域
- **And** 备注不再通过预览下方固定表单录入
- **And** 评论应通过预览悬浮入口触发弹窗完成录入。

### 1B. 预览悬浮评论入口
- **Given** 用户在 `plan.md` 预览区域中移动鼠标
- **When** 鼠标悬停到可评论的段落、列表项、标题或表格行附近
- **Then** `Add Comment` 图标应出现在当前内容块附近，而不是固定吸附在页面最左或最右
- **And** 点击后自动把备注目标定位到相关章节
- **And** 弹出评论对话框
- **And** 可将当前悬停内容摘要带入备注上下文。

### 2. Markdown 编辑
- **Given** `plan.md` 已在顶部 Tab 打开
- **When** 用户编辑并保存
- **Then** 内容写回：

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
```

### 3. 备注写入
- **Given** 用户在 Plan 编辑面板中添加备注
- **When** 用户保存备注
- **Then** 备注写入：

```text
@state/projects/<projectId>/plans/<conversationId>/remarks.json
```

### 3A. 备注删除
- **Given** 当前 plan 已存在 remark
- **When** 用户点击该 remark 的删除动作
- **Then** 对应 remark 从 `remarks.json` 中移除
- **And** 预览审阅列表即时更新。

### 4. Chat 回流优化
- **Given** 用户编辑或新增备注后回到 Chat
- **When** 发送下一条消息
- **Then** Chat 上下文自动包含：
  - 最新 `plan.md`
  - 最新 `remarks.json`
  - 当前 `plan.state.json.status`

### 4A. 不通过聊天命令触发状态变化
- Chat 文本输入用于计划内容协作，不用于触发 Plan 生命周期动作。
- 本 Story 不引入聊天内控制命令协议。
- `Regenerate Plan`、`Approve`、`Execute` 等动作必须由 UI 触发。

### 5. 备注模型限制
- **Given** 本次为 MVP
- **When** 用户添加备注
- **Then** 备注采用结构化列表模型
- **And** 不要求实现真正的行级锚点批注。

### 6. Regenerate Plan 的输入来源
- **Given** 当前 Plan 处于 `drafting` 或 `reviewing`
- **When** 用户先在聊天框中输入内容，再点击 `Regenerate Plan`
- **Then** 当前聊天框内容应作为“优化当前计划”的请求传递给 Runtime
- **And** 系统基于最新 `plan.md`、remarks 和该输入重写当前计划。

## Technical Components / Changes

1. **WorkspacePage**
   - 增加打开 alias path 文件 Tab 的能力
2. **ConversationView**
   - 调用打开 `plan.md` Tab 的回调
3. **Plan Remark Editor**
   - 基于悬浮目标的评论对话框
   - comment 卡片列表
4. **Chat Context Injection**
   - 在 `callAgentChat()` 中注入 Plan 与备注内容
5. **Regenerate Plan Input Binding**
   - 右侧按钮读取当前聊天框草稿并作为计划优化指令发送

## Delivery Scope by Layer

### Backend Scope

- 支持读取和写入 `plan.md`。
- 支持读取和写入 `remarks.json`。
- 在 Chat 请求链路中注入 Plan 和备注上下文。

### Frontend Scope

- 在右侧 Plan 面板中触发打开顶部 Tab。
- 顶部 Tab 中可编辑 `plan.md`。
- 顶部 Tab 中提供“预览 + 评论弹窗”审阅区域。
- 预览区域支持 hover 浮出 `Add Comment` 入口。
- 点击后通过评论对话框录入结构化备注。
- 在进入后续自动优化前，用户可以手动删除 remark。
- `Preview` 模式下不再重复渲染下方整页正文预览。
- 仅在 `Edit` 模式下显示编辑器。
- 已有备注以 comment 卡片显示，而不是纯文本列表。
- 返回 Chat 后不需要用户手工复制计划内容。

### Integration Scope

- 编辑后的 `plan.md` 不要求同步到右侧正文预览；右侧仅保持控制状态一致。
- 新增备注后，下一次 Chat 请求上下文包含备注。
- 顶部 Tab 打开的 alias 路径与真实工件路径一致。
- 点击 `Regenerate Plan` 时，使用当前聊天框草稿作为计划优化输入。

## Dependencies
- 依赖 Story 5-20-1 的工件路径和 metadata。
- 依赖 Story 5-20-2 的 Plan 面板入口。
- 为 Story 5-20-4 的执行前审阅提供基础。

## Related Artifacts
- PRD: `_bmad-output/prd-runtime-chat-plan-mode.md`
- Epic: `_bmad-output/epics-runtime-chat-plan-mode.md`
- Design: `_bmad-output/implementation-artifacts/design-5-20-chat-plan-mode.md`
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-20-chat-plan-mode.md`

## Verification Plan

### Manual Verification
1. 从 Chat 详情区打开 `plan.md`，确认顶部 Tab 出现。
2. 顶部 Tab 中能看到计划预览，并可通过 hover 打开评论对话框。
3. 编辑并保存文档，重新打开会话后内容仍存在。
4. 添加备注，检查 `remarks.json` 持久化。
5. 返回 Chat 继续提问，确认 LLM 回复体现新计划或备注。
6. 在聊天框输入优化要求后点击 `Regenerate Plan`，确认计划按该输入被重写。

### Automated Tests
- Workspace 顶部 Tab 打开逻辑测试。
- Plan 文档保存测试。
- 备注持久化测试。
- Chat context injection 测试。

## Risks / Mitigations

| Risk | Description | Mitigation |
|:---|:---|:---|
| R1 | 顶部 Tab 打开逻辑与现有文件 Tab 模型冲突 | 统一复用现有 file tab id/path 规则 |
| R2 | 备注模型过于复杂导致实现扩张 | v1 只支持结构化备注列表，不做行级锚点 |
| R3 | Chat 注入上下文过长影响 token 使用 | 在注入时只带必要摘要、备注和状态，而不是无限拼接历史版本 |
| R4 | 错把聊天文本当成控制命令，导致状态机漂移 | 明确规定所有状态动作只能来自 UI，`Regenerate Plan` 只消费当前聊天草稿 |

## Acceptance Checklist

- [x] 点击 `查看/编辑 plan.md` 后顶部出现 Plan Tab
- [x] 再次点击不会重复创建多个同路径 Tab
- [x] 顶部 Tab 中显示计划预览，并可通过 hover 打开评论对话框
- [x] `plan.md` 可编辑并保存
- [x] 备注成功写入 `remarks.json`
- [x] 备注列表以 comment 卡片展示目标章节、类型和时间
- [x] 返回 Chat 后发送消息，LLM 上下文包含最新计划和备注
- [x] Chat 回复能体现编辑或备注的影响
- [x] 不存在聊天命令协议触发状态迁移的实现
- [x] `Regenerate Plan` 使用当前聊天框输入作为优化计划的要求

## Sequencing Notes

- 本 Story 完成后，Plan 会从“可看”升级为“可协作”。
- 如果时间受限，备注 UI 可以先做最小表单，不必追求复杂交互。

## Tasks / Subtasks
- [x] Backend-1 支持 `plan.md` 文件读写
- [x] Backend-2 支持 `remarks.json` 读写
- [x] Backend-3 在 `callAgentChat()` 中增加 Plan/备注上下文注入
- [x] Backend-4 保证聊天输入只作为内容层输入，不承担状态控制命令
- [x] Frontend-1 在 WorkspacePage 中暴露 alias file tab 打开能力
- [x] Frontend-2 在 ConversationView 中接入 `查看/编辑 plan.md` 按钮
- [x] Frontend-3 复用 Markdown Editor 编辑 `plan.md`
- [x] Frontend-4 实现悬浮触发的评论对话框
- [x] Frontend-5 实现备注 comment 卡片列表
- [x] Frontend-6 将 `Regenerate Plan` 与当前聊天框草稿绑定
- [x] Integration-1 验证编辑保存后右侧控制状态保持一致
- [x] Integration-2 验证备注写入后 Chat 能继续优化 Plan
- [x] Integration-3 验证聊天自由文本不会直接触发 Plan 生命周期变化
- [x] Integration-4 验证 `Regenerate Plan` 会消费当前聊天框草稿作为优化输入
- [x] Test-1 补顶部 Tab 打开与复用测试
- [x] Test-2 补 remarks 持久化和 context injection 测试

## File List

- `_bmad-output/implementation-artifacts/5-20-3-plan-tab-editing-and-remarks.md`
- `_bmad-output/implementation-artifacts/validation-report-story-5-20-3.md`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.css`
- `crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.ts`
- `crewagent-runtime/src/pages/WorkspacePage/workspaceTabLogic.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`

## Dev Agent Record

### Agent Model Used

GPT-5 Codex

### Debug Log References

- `crewagent-runtime`: `./node_modules/.bin/tsc --noEmit --pretty false`
- `crewagent-runtime`: `./node_modules/.bin/eslint src/pages/WorkspacePage/WorkspacePage.tsx src/pages/WorkspacePage/workspaceTabLogic.ts src/pages/WorkspacePage/workspaceTabLogic.test.ts electron/stores/runtimeStore.test.ts`
- `crewagent-runtime`: `./node_modules/.bin/vitest run src/pages/WorkspacePage/workspaceTabLogic.test.ts electron/stores/runtimeStore.test.ts`

### Completion Notes List

- `View/Edit plan.md` 已复用顶部 alias file tab 机制，重复点击会复用已打开的 `plan.md` tab，而不会重复创建。
- `plan.md` 的 `Preview` 模式保留审阅视图与 hover comment 弹窗，`Edit` 模式仅显示 Markdown 编辑器。
- remark 写入和删除继续以 `remarks.json` 为唯一事实来源，不混入聊天消息。
- 手工编辑 `plan.md` 后的保存已接入原子化 plan draft 持久化路径，保证正文、任务和状态一起更新。
- 手工编辑保存默认保留现有 remarks，不会误清空待处理 comments。
- 新增了顶部 tab 复用和 structured remark 构造的纯逻辑测试。
- 新增了 RuntimeStore 级别的回归测试，验证手工保存 `plan.md` 时 remarks 可保留。
- `plan.md` tab 现在绑定自己的 `conversationId / planPath / remarksPath` 工件身份，不再依赖当前 `selectedConversation`。
- 切换 conversation 或切换 tab 时会关闭 comment dialog，避免把旧弹窗内容写到错误的 plan remark 文件。
- `View/Edit plan.md` 在复用已打开 tab 时会重新读取最新磁盘内容，避免聊天区 `Regenerate Plan` 之后 tab 仍停留在旧版草稿。
- 从 Files 面板直接打开匹配 conversation `planPath` 的 `plan.md` 时，也会自动识别为 plan 工件，而不是退化成普通文件保存路径。
- 删除 conversation 后，绑定到该 conversation 的 `plan.md` tab 会自动从顶部 tab 列表移除，避免留下悬空计划工件编辑页。

## Change Log

- 2026-03-26: 完成 `plan.md` 顶部 tab 编辑与审阅闭环，状态更新为 `review`。
- 2026-03-26: 将 `plan.md` 手工保存接入统一的原子化 draft 持久化路径，确保编辑后的状态、任务与 version 同步更新。
- 2026-03-26: 新增 `workspaceTabLogic` 逻辑测试和 `RuntimeStore` remarks 保留测试，补齐本 Story 的最小自动化证据。
- 2026-03-26: 修复 `plan` tab 对 `selectedConversation` 的隐式耦合，保存与备注改为基于 tab 自身绑定的 Plan 工件身份。
- 2026-03-26: 修复 comment dialog 打开后被立即关闭的问题，并补齐 tab 复用刷新与 Files 面板 plan 身份识别。
- 2026-03-26: 删除 conversation 时自动清理关联的 plan tab，避免已删除计划工件继续留在顶部 tab 中。
- 2026-03-28: 按 BMAD sprint closing 约定关闭 Story；结合 story-group code review 中的 `Approved with Residual Test Risk` 结论，将状态更新为 `done`，残余测试风险保留在 review 记录中。
