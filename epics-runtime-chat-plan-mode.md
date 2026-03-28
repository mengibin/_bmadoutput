---
stepsCompleted: [1, 2, 3, 9]
inputDocuments:
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd-runtime-chat-plan-mode.md'
  - '/Users/mengbin/code/GPT/CrewAgent/_bmad-output/prd.md'
workflowType: 'epics'
lastStep: 9
project_name: 'CrewAgent Runtime Chat Plan Mode'
user_name: 'Mengbin'
date: '2026-02-28'
---

# CrewAgent Runtime Chat Plan Mode - Epic Breakdown

## Overview

本文件将 `Runtime Chat Plan Mode` 子需求拆解为 **单 Epic + 四条前后端一体可测试 Story**，覆盖：

- Chat 下的 Plan 子模式开关
- 项目级 Plan 工件存储
- Plan 详情区控制面板
- 顶部 Tab 的 `plan.md` 查看和编辑
- 备注录入与回流优化
- 同意并执行 Plan
- 执行进度可视化与恢复

来源文档：

- `_bmad-output/prd-runtime-chat-plan-mode.md`
- `_bmad-output/prd.md`

---

## Requirements Inventory

### Functional Requirements

- FR-PLAN-01: Chat 会话支持 `Plan` 子模式开关。
- FR-PLAN-02: 开启 Plan 后自动初始化项目级 Plan 工件。
- FR-PLAN-03: 支持从 Chat 详情区打开并编辑 `plan.md`。
- FR-PLAN-04: 支持在 Plan 编辑面板中增加结构化备注。
- FR-PLAN-05: 返回 Chat 后，LLM 自动读取计划与备注上下文继续优化。
- FR-PLAN-06: 支持用户同意并执行 Plan。
- FR-PLAN-07: 支持计划执行进度可视化。
- FR-PLAN-08: 支持会话恢复与项目级数据清理。
- FR-PLAN-09: Plan 生命周期状态变化必须由显式 UI 动作触发，不通过聊天命令协议触发。
- FR-PLAN-10: 执行中需要用户决策时，用户补充内容后默认继续当前任务执行。
- FR-PLAN-11: 打开 Plan Mode 后，`plan.md` 初始必须为空草稿。
- FR-PLAN-12: Chat 输入默认语义必须随 Plan 状态变化，在 `drafting/reviewing` 时编辑计划，在 `waiting_user` 时继续当前任务。
- FR-PLAN-13: 当前 Plan 完成后，下一次聊天输入必须开启新的计划版本。
- FR-PLAN-14: `waiting_user` 期间显式点击 `Regenerate Plan` 时，当前聊天框输入必须被解释为计划优化请求。

### Non-Functional Requirements

- NFR-PLAN-01: 项目级隔离。
- NFR-PLAN-02: 可追溯性。
- NFR-PLAN-03: 可恢复性。
- NFR-PLAN-04: 前后端一体可测试。
- NFR-PLAN-05: 向后兼容。
- NFR-PLAN-06: 上下文可控。

---

## FR Coverage Map

| FR/NFR | Epic |
|:---|:---|
| FR-PLAN-01 ~ FR-PLAN-14 | Epic PLAN-1 |
| NFR-PLAN-01 ~ NFR-PLAN-06 | Epic PLAN-1 |

---

## Epic List

### Epic PLAN-1: Runtime Chat Plan Mode

**Goal**: 在现有 Chat 会话中一次性交付 Plan 子模式，使用户可以创建、编辑、备注、优化、批准和执行计划，并按项目查看计划进度。  
**FRs covered**: FR-PLAN-01 ~ FR-PLAN-14, NFR-PLAN-01 ~ NFR-PLAN-06

**Deliverables**:

- Chat 子模式开关与会话元数据扩展。
- 项目级 `plan.md / plan.state.json / remarks.json` 工件。
- Chat 详情区的 Plan 控制面板。
- 顶部 Tab 打开与编辑 `plan.md`。
- 备注录入与 Chat 优化回流。
- 同意并执行入口与计划进度卡片。
- UI 驱动的状态迁移。
- 执行中等待用户决策并继续当前任务。
- Plan 开启后空白草稿初始化。
- 基于 Plan 状态切换聊天输入语义。
- 完成后基于下一次聊天输入开启新版本 Plan。
- conversation 删除与项目清理联动。

---

## Epic PLAN-1 Stories

## Recommended Implementation Order

建议严格按以下顺序实施，避免后续 Story 反向依赖前置能力：

1. **5-20-1 Project-Scoped Plan Storage and Session Metadata**
   建立路径、工件、会话元数据与清理能力。
2. **5-20-2 Chat Plan Panel and Preview**
   先让用户看得到、切得动、能读取状态。
3. **5-20-3 Plan Tab Editing and Remarks**
   再补真实文档编辑和备注回流优化。
4. **5-20-4 Approve, Execute, and Visualize Progress**
   最后进入执行态和进度渲染。

原因：

- 5-20-2 依赖 5-20-1 的工件路径与 metadata
- 5-20-3 依赖 5-20-1 和 5-20-2 的入口和数据基础
- 5-20-4 依赖前 3 条 Story 已形成完整的 Plan 工件闭环

---

## Story Testability Notes

所有 Story 均按“前后端一体可测试”的方式拆分，要求每条 Story 都同时具备：

| Story | Backend Result | Frontend Result | Integration Result |
|:---|:---|:---|:---|
| 5-20-1 | 成功创建/恢复/清理 Plan 工件与 metadata | Chat 可见 Plan 开关且状态持久 | 重开会话后状态恢复 |
| 5-20-2 | 可读取 `plan.md` / `plan.state.json` | 右侧面板正确显示 | 切换 Plan 开关后 UI 与文件状态一致 |
| 5-20-3 | `plan.md` 与 `remarks.json` 可写入 | 顶部 Tab 打开并可编辑 | 返回 Chat 后上下文能体现编辑与备注 |
| 5-20-4 | `plan.state.json` 可推进执行态与任务状态 | 进度卡片、阶段、百分比可见 | 重启后仍可恢复执行进度 |

每条 Story 的验证报告都应至少覆盖：

- 文件落盘结果
- 会话恢复结果
- UI 行为结果
- 关键异常路径

---

### Story PLAN-1.1 / 5-20-1: Project-Scoped Plan Storage and Session Metadata

Story Artifact: `_bmad-output/implementation-artifacts/5-20-1-project-scoped-plan-storage-and-session-metadata.md`

As a **Chat User**,  
I want Plan artifacts and session metadata created under a project-scoped state tree,  
So that Plan data can be restored per conversation and cleaned up per project.

**Acceptance Criteria:**

**Given** I open the `Plan` toggle in a Chat conversation  
**When** Plan mode is initialized for the first time  
**Then** the system creates:

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
@state/projects/<projectId>/plans/<conversationId>/plan.state.json
@state/projects/<projectId>/plans/<conversationId>/remarks.json
```

**And** conversation metadata stores `chatSubmode`, `planStatus`, and artifact paths.
**And** `plan.md` starts empty rather than with a default template body.

**Given** I reload the project or restart the app  
**When** the conversation is reopened  
**Then** the previously created Plan metadata and file paths are restored.

**Given** I delete the conversation  
**When** cleanup runs  
**Then** only the corresponding `plans/<conversationId>/` subtree is deleted  
**And** other conversations under the same project remain intact.

### Story PLAN-1.2 / 5-20-2: Chat Plan Panel and Controls

Story Artifact: `_bmad-output/implementation-artifacts/5-20-2-chat-plan-panel-and-preview.md`

As a **Chat User**,  
I want the Chat details panel to become a Plan control panel when Plan mode is enabled,  
So that I can understand and control the current plan without leaving the conversation.

**Acceptance Criteria:**

**Given** a Chat conversation with Plan mode enabled  
**When** the right-side details panel is rendered  
**Then** it shows:
- Plan toggle
- `查看/编辑 plan.md` button
- `Regenerate Plan` button
- `同意并执行 Plan` button
- `Resume Execution` button when execution can continue
- plan status card
- progress overview placeholder or active progress block

**Given** Plan mode is turned off  
**When** the details panel is rendered  
**Then** it falls back to the standard Chat details view.

### Story PLAN-1.3 / 5-20-3: Plan Tab Editing and Remarks

Story Artifact: `_bmad-output/implementation-artifacts/5-20-3-plan-tab-editing-and-remarks.md`

As a **Plan Reviewer**,  
I want to open `plan.md` in a top tab, edit it, and add remarks,  
So that I can refine the plan as a real document and continue optimization in chat.

**Acceptance Criteria:**

**Given** I click `查看/编辑 plan.md`  
**When** the file opens  
**Then** a top Workspace tab is created for the Plan document  
**And** I can edit and save the markdown content.

**Given** I add a remark in the Plan editing area  
**When** I save the remark  
**Then** the remark is written to `remarks.json`.

**Given** I return to Chat after editing or adding remarks  
**When** I send the next message  
**Then** the runtime injects `plan.md`, remarks, and current plan status into chat context.

**Given** Plan is in `drafting` or `reviewing`
**When** the user sends a chat message
**Then** that message is interpreted as authoring or modifying the current plan.

**Given** Plan is in `waiting_user`
**When** the user sends a normal chat message
**Then** that message is interpreted as the decision input for the current waiting task.

### Story PLAN-1.4 / 5-20-4: Approve, Execute, and Visualize Progress

Story Artifact: `_bmad-output/implementation-artifacts/5-20-4-approve-execute-and-progress-visualization.md`

As a **Project Executor**,  
I want to approve a plan and track its execution progress from Chat,  
So that planning and execution stay connected in a single workspace.

**Acceptance Criteria:**

**Given** a plan is ready for execution  
**When** I click `同意并执行 Plan`  
**Then** `plan.state.json.status` transitions into `executing`  
**And** execution metadata is recorded.

**Given** Plan is in `waiting_user`
**When** the user clicks `Regenerate Plan`
**Then** the current chat draft is treated as a plan refinement request rather than an execution decision.

**Given** execution progresses  
**When** the Chat details panel is refreshed  
**Then** it shows task-level status, current phase, and completion percentage.

**Given** I leave the page or restart the app  
**When** I reopen the conversation  
**Then** the latest Plan progress is restored from `plan.state.json`.

**Given** the current Plan is completed
**When** the user sends the next chat message
**Then** the system starts `version + 1` of the plan rather than overwriting the completed plan.
