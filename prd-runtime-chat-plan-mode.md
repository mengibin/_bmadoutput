# PRD: Runtime Chat Plan Mode

> **Parent Document**: [Product Requirements Document (CrewAgent)](prd.md)
> **Traceability**: FR-PLAN-01 / FR-PLAN-02 / FR-PLAN-03 / FR-PLAN-04 / FR-PLAN-05 / FR-PLAN-06 / FR-PLAN-07 / FR-PLAN-08 / FR-PLAN-09 / FR-PLAN-10 / FR-PLAN-11 / FR-PLAN-12 / FR-PLAN-13 / FR-PLAN-14 in `prd.md`

## 1. 概述

本 PRD 定义 Runtime 在现有 `Chat` 会话内新增 `Plan` 子模式的产品能力，重点支持：

1. 在 `Chat` 详情区通过 `Plan` 开关进入计划工作流；
2. 为每个项目下的会话创建独立的计划文档与计划状态；
3. 支持在顶部 Tab 中查看和编辑 `plan.md`；
4. 支持增加结构化备注，并在返回 Chat 后继续优化计划；
5. 支持用户同意并执行计划，并在 Chat 页面查看计划进度；
6. 支持根据 Plan 当前状态切换聊天输入的默认语义。

本能力不新增新的顶层会话类型，仍保持 `workflow / agent / chat` 的现有结构，仅在 `chat` 内增加 `Plan` 子模式。
本能力不引入聊天内控制命令协议；所有会导致 Plan 生命周期状态变化的操作，均通过显式 UI 动作触发。

---

## 2. 用户故事 (User Stories)

- 作为项目执行者，我希望在 Chat 中先整理出一个临时工作计划，再决定是否执行，而不是直接进入不可逆的自动执行流程。
- 作为计划审阅者，我希望可以直接查看和编辑 `plan.md`，并对计划内容增加备注，便于多轮修订。
- 作为 Chat 用户，我希望返回聊天界面后，LLM 自动理解当前计划和备注，继续优化方案，而不是重复粘贴上下文。
- 作为项目维护者，我希望所有计划数据按项目归档，后续清理孤立项目数据时可以统一删除。
- 作为项目跟踪者，我希望计划一旦执行，能在 Chat 页面看到清晰的阶段状态和任务进度。

---

## 3. 问题定义 (Problem Definition)

当前 Runtime 的 Chat 模式适合自由对话，但缺少“计划协作 -> 审阅 -> 执行 -> 进度跟踪”的完整工作流，导致：

1. 用户和 LLM 生成的计划只存在于消息流中，不是独立工件，难以复用和编辑；
2. 用户无法在统一的文档界面中修改计划，也不能把备注与计划状态结构化保存；
3. 计划执行后没有独立的状态模型和可视化进度，只能依赖聊天消息自行理解；
4. 计划相关临时数据如果没有项目级隔离，后续项目清理和孤立数据回收会非常困难。

需要在保留 Chat 灵活性的前提下，增加一个项目级、会话级、可追溯的 Plan 协作能力。

---

## 4. 功能范围 (Scope)

### In Scope

- `Chat` 详情区显示 `Plan` 开关与 Plan 控制面板。
- 为开启 Plan 的会话自动创建计划工件：
  - `plan.md`
  - `plan.state.json`
  - `remarks.json`
- `plan.md` 初始内容为空白草稿，由聊天输入逐步编写。
- 支持 `plan.md` 在顶部 Tab 中打开、查看、编辑、保存。
- 支持结构化备注录入与持久化。
- Chat 请求时自动注入当前 `plan.md` 与备注上下文。
- 支持 `Regenerate Plan` 与“同意并执行 Plan”入口。
- 支持执行过程中因用户决策而暂停，并在用户补充内容后继续当前任务执行。
- 支持在 `waiting_user` 状态下，通过显式点击 `Regenerate Plan` 将当前聊天框输入解释为“优化计划”而不是“继续任务”。
- 支持当前 Plan 完成后，在下一次聊天输入时开启新版本 Plan。
- 支持在 Chat 页面查看计划执行状态与任务进度。
- 支持 conversation 删除与 project 清理时的计划数据联动清理。

### Out of Scope (MVP)

- 真正的行级批注锚点系统。
- 多人实时协同编辑计划。
- 将 Plan 自动转换为 workflow graph 并接入现有 Run Engine。
- 复杂的计划版本差异比较 UI。
- 聊天消息中的控制命令协议（如 `PLAN_ACTION` envelope）。

---

## 5. 工件与存储原则 (Artifacts & Storage Rules)

所有 Plan 工件必须遵循“项目优先”目录规范：

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
@state/projects/<projectId>/plans/<conversationId>/plan.state.json
@state/projects/<projectId>/plans/<conversationId>/remarks.json
```

规则：

1. `projectId` 是一级命名空间，用于项目级隔离与统一清理。
2. `conversationId` 是二级命名空间，用于区分同一项目内的多个计划会话。
3. conversation 删除只清理自身目录。
4. 项目删除或孤立项目清理应删除整个 `@state/projects/<projectId>/...` 命名空间。

---

## 6. 验收标准 (Acceptance Criteria)

### AC-1: Chat 下的 Plan 子模式

- 在现有 `Chat` 会话中提供 `Plan` 开关。
- 打开后进入 `Chat Plan Mode`，关闭后回退到普通 Chat。
- 不新增新的顶层 `ConversationType`。

### AC-2: 项目级 Plan 工件初始化

- 用户首次在某个 Chat 会话中打开 Plan 开关时，系统自动创建：
  - `plan.md`
  - `plan.state.json`
  - `remarks.json`
- 文件必须创建在：

```text
@state/projects/<projectId>/plans/<conversationId>/
```

- 会话刷新或重启后，系统能恢复这些路径与状态。
- `plan.md` 初始为空文件或空白草稿。
- `remarks.json` 初始为空数组。
- `plan.state.json` 初始为 `version = 1`、`status = drafting`、`tasks = []`。

### AC-3: Plan 面板与控制区

- Chat 详情区在 Plan 模式下显示：
  - Plan 开关
  - `查看/编辑 plan.md` 按钮
- `Regenerate Plan` 按钮
  - `同意并执行 Plan` 按钮
  - `Resume Execution` 按钮（仅在允许时显示或可点击）
  - 计划状态与进度概览

### AC-4: 顶部 Tab 编辑 `plan.md`

- 点击 `查看/编辑 plan.md` 后，在顶部 Tab 打开 Plan 编辑面板。
- 用户可以查看、编辑、保存 `plan.md`。
- 保存后，回到 Chat 页面不要求显示正文预览，但相关控制状态必须保持一致。
- 顶部 Tab 中应支持“计划预览 + 添加备注”的审阅方式。

### AC-5: 备注录入与回流优化

- 在 Plan 编辑面板中，用户可以增加结构化备注。
- 备注持久化到 `remarks.json`。
- 返回 Chat 页面继续发送消息时，LLM 自动获得：
  - 当前 `plan.md`
  - 当前备注列表
  - 当前计划状态
- LLM 可以基于这些信息继续优化计划。

### AC-5B: 聊天输入的状态化语义

- 当 Plan 处于 `drafting / reviewing` 时：
  - 用户在聊天框输入并发送内容，默认语义为“编写或修改当前计划”。
- 当 Plan 处于 `executing` 且存在 `waiting_user` 任务时：
  - 用户在聊天框输入并发送内容，默认语义为“该等待任务的决策输入”，系统应继续当前任务执行。
- 当 Plan 处于 `executing` 且存在 `waiting_user` 任务时：
  - 若用户点击 `Regenerate Plan`，则系统应将当前聊天框输入解释为“优化当前计划的要求”，进入计划修订，而不是继续当前任务。

### AC-5A: 状态变化由 UI 触发

- Chat 文本输入仅用于计划讨论、修订说明、执行中补充决策和结果追问。
- 所有会导致 Plan 生命周期状态变化的操作，必须由显式 UI 控件触发。
- 系统不通过聊天消息协议触发 `execute`、`resume`、`rollover` 等状态动作。

### AC-6: 同意并执行

- 用户点击 `同意并执行 Plan` 后，计划状态至少从 `drafting/reviewing` 进入 `executing`。
- 执行状态写入 `plan.state.json`。
- Chat 详情区显示执行态信息，而不是仅显示静态计划。

### AC-6A: 执行中等待用户决策

- 当某个任务执行到一半需要用户补充决策时，系统可将当前任务标记为 `waiting_user`。
- 用户在该状态下补充内容后，默认语义应为“继续当前任务执行”，而不是“重新编写整份计划”。
- 若用户要重写或重开计划，必须通过显式 UI 动作触发。

### AC-7: 计划进度可视化

- 系统在 Chat 页面展示计划任务列表、当前阶段、完成比例。
- 重新进入会话或重启应用后，计划进度可以恢复显示。

### AC-7A: 执行完成后的继续对话

- Plan 执行完成后，用户下一次在聊天框输入并发送内容时，系统应创建 `version + 1` 的新计划草稿，而不是覆盖旧版本。
- 创建新版本时，系统应清空上一版本的结构化执行任务，并将生命周期状态重置为 `drafting` 或新的草稿修订态。
- 若用户需要继续未完成执行流程，应通过显式 UI 动作继续执行，而不是靠普通聊天输入触发。

### AC-8: 项目与会话清理

- 删除 conversation 时，仅清理该 conversation 对应的 Plan 工件目录。
- 删除项目或清理孤立项目时，应删除该项目对应的全部 Plan 工件。

---

## 7. 功能需求编号 (Functional Requirements)

- **FR-PLAN-01**: Chat 会话支持 `Plan` 子模式开关。
- **FR-PLAN-02**: 开启 Plan 后自动初始化项目级 Plan 工件。
- **FR-PLAN-03**: 支持从 Chat 详情区打开并编辑 `plan.md`。
- **FR-PLAN-04**: 支持在 Plan 编辑面板中增加结构化备注。
- **FR-PLAN-05**: 返回 Chat 后，LLM 自动读取计划与备注上下文继续优化。
- **FR-PLAN-06**: 支持用户同意并执行 Plan。
- **FR-PLAN-07**: 支持计划执行进度可视化。
- **FR-PLAN-08**: 支持会话恢复与项目级数据清理。
- **FR-PLAN-09**: Plan 生命周期状态变化必须由显式 UI 动作触发，不通过聊天命令协议触发。
- **FR-PLAN-10**: 执行中需要用户决策时，用户补充内容后默认继续当前任务执行。
- **FR-PLAN-11**: 打开 Plan Mode 后，`plan.md` 初始必须为空草稿，不应自动生成默认正文模板。
- **FR-PLAN-12**: Chat 输入的默认语义必须随 Plan 生命周期状态变化而变化，在 `drafting/reviewing` 时用于编写计划，在 `waiting_user` 时用于继续当前任务。
- **FR-PLAN-13**: 当前 Plan 全部执行完成后，下一次聊天输入必须开启新的计划版本，而不是覆盖当前版本。
- **FR-PLAN-14**: 当存在 `waiting_user` 任务时，显式点击 `Regenerate Plan` 必须将当前聊天框输入解释为计划优化请求，而不是执行决策输入。

---

## 8. 非功能需求 (Non-Functional Requirements)

- **NFR-PLAN-01 项目级隔离**：所有 Plan 数据必须按项目命名空间隔离。
- **NFR-PLAN-02 可追溯性**：计划、备注、状态必须可通过 `projectId + conversationId` 追溯。
- **NFR-PLAN-03 可恢复性**：应用重启后可恢复 Plan 状态与进度。
- **NFR-PLAN-04 可测试性**：Story 必须按前后端最小闭环拆分，并具备可独立验收条件。
- **NFR-PLAN-05 向后兼容**：不得破坏现有 `workflow / agent / chat` 行为。
- **NFR-PLAN-06 上下文可控**：Plan 注入 Chat 时必须避免无界全文堆叠，优先注入当前版本的必要上下文。

---

## 9. 技术方向 (Technical Direction)

### 9.1 会话元数据扩展

建议在 Conversation Metadata 上增加：

- `chatSubmode?: 'default' | 'plan'`
- `planStatus?: 'drafting' | 'reviewing' | 'executing' | 'completed' | 'blocked'`
- `planPath?: string`
- `planStatePath?: string`
- `planRemarksPath?: string`

### 9.2 Plan 状态模型

`plan.state.json` 建议至少包含：

```json
{
  "conversationId": "conv_xxx",
  "projectId": "project_xxx",
  "version": 1,
  "status": "drafting",
  "tasks": [],
  "summary": "",
  "approvedAt": null,
  "executingAt": null,
  "completedAt": null,
  "updatedAt": "2026-01-01T00:00:00.000Z"
}
```

补充要求：

- `tasks[]` 是执行阶段的系统真源。
- 执行阶段不得依赖对自由格式 `plan.md` 的即时解析来决定任务列表。
- 若 `tasks[]` 缺失或与当前计划版本不一致，应在批准执行前进行一次规范化重建。
- `plan.md` 的默认生成时机不是“打开 Plan 开关”，而是用户开始在 Chat 中输入计划内容之后。

### 9.3 备注模型

`remarks.json` 建议采用结构化数组：

```json
[
  {
    "id": "remark_1",
    "target": {
      "kind": "document",
      "heading": "范围与交付物"
    },
    "content": "补充风险控制与依赖条件",
    "createdAt": "2026-01-01T00:00:00.000Z"
  }
]
```

### 9.4 Chat Context 注入原则

- `plan.md` 不应被硬编码进固定系统模板中。
- Runtime 应在每轮 Chat 请求前，基于当前 `plan.md`、`remarks.json`、`plan.state.json` 动态构造 `PLAN_CONTEXT` 注入消息。
- 注入内容应优先包含：
- 当前计划版本
  - 当前计划状态
  - 当前计划摘要或必要全文
  - 当前备注列表
  - 当前执行阶段信息

### 9.5 执行与继续对话原则

- 当 `planStatus = executing` 且当前任务 `status = waiting_user` 时，用户补充内容默认用于继续当前任务。
- 当 `planStatus = completed` 且仍处于 Plan Mode 时，用户下一次发送内容默认用于创建 `version + 1` 的新计划草稿，而不是覆盖已完成版本。
- 若计划内容发生实质变更，则必须进入新版本或重新审批流程，而不是直接沿用既有批准状态。

---

## 10. 验证计划 (Validation Plan)

### 10.1 核心回归场景

1. 新建 Chat 会话 -> 打开 Plan -> 创建项目级 Plan 工件。
2. 编辑 `plan.md` -> 返回 Chat -> 继续优化。
3. 增加备注 -> 返回 Chat -> LLM 响应中体现备注影响。
4. 同意并执行 -> 进度开始显示。
5. 重启应用 -> 恢复 Plan 状态。
6. 删除 conversation -> 仅删除对应 Plan 子目录。
7. 清理项目 -> 删除整个项目下所有 Plan 工件。

### 10.2 交付完成标志

- 子 PRD、Epic、Design、Tech Spec、Story 文档全部建立交叉引用。
- Story 可按前后端一体方式逐条实现与验收。
- 所有路径、状态与命名规则在文档中保持一致。
