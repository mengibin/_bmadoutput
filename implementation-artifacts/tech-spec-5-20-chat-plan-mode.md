# Tech Spec: Story Group 5-20 – Chat Plan Mode

## Summary

在现有 Chat 会话基础上增加 `Plan` 子模式，使用项目级 Plan 工件目录保存：

- `plan.md`
- `plan.state.json`
- `remarks.json`

并通过 Works 详情区、Workspace 顶部 Tab、Chat 上下文注入与独立进度模型，完成“空白草稿初始化 -> Chat 编写计划 -> 编辑备注 -> Chat 优化 -> 批准执行 -> 进度展示 -> 完成后新版本”的闭环。

---

## 1. Backend (RuntimeStore + IPC)

### 1.1 Directory Structure

```text
$RUNTIME_DATA/projects/<projectId>/
├── conversations/
│   ├── index.json
│   └── <conversationId>/
│       └── messages.json
├── state/
│   └── projects/
│       └── <projectId>/
│           └── plans/
│               └── <conversationId>/
│                   ├── plan.md
│                   ├── plan.state.json
│                   └── remarks.json
└── runs/
```

说明：

- `conversations/` 继续保存聊天消息
- `state/projects/<projectId>/plans/` 专门保存 Plan 工件

### 1.2 Conversation Metadata Extension

扩展 `ConversationMetadata`：

```typescript
type ChatSubmode = 'default' | 'plan'
type PlanLifecycleStatus =
  | 'drafting'
  | 'reviewing'
  | 'executing'
  | 'completed'
  | 'blocked'

interface ConversationMetadata {
  id: string
  title: string
  entryType: 'workflow' | 'agent' | 'chat'
  activeType: 'workflow' | 'agent' | 'chat'
  createdAt: string
  lastActiveAt: string
  runs: ConversationRunMetadata[]
  selectedWorkflowId?: string
  selectedAgentId?: string
  generatedTitle?: string
  customTitle?: string

  chatSubmode?: ChatSubmode
  planStatus?: PlanLifecycleStatus
  planPath?: string
  planStatePath?: string
  planRemarksPath?: string
}
```

### 1.3 Plan State Types

```typescript
type PlanTaskStatus = 'pending' | 'in_progress' | 'waiting_user' | 'completed' | 'blocked' | 'cancelled'

interface PlanTaskRecord {
  id: string
  title: string
  description?: string
  status: PlanTaskStatus
  progress: number
  notes?: string
  dependsOn?: string[]
  waitingReason?: string
}

interface PlanStateRecord {
  conversationId: string
  projectId: string
  version: number
  status: PlanLifecycleStatus
  summary: string
  tasks: PlanTaskRecord[]
  approvedAt?: string | null
  executingAt?: string | null
  completedAt?: string | null
  updatedAt: string
}

interface PlanRemarkRecord {
  id: string
  target: {
    kind: 'document' | 'section'
    heading?: string
  }
  content: string
  createdAt: string
}
```

### 1.4 RuntimeStore Methods

建议新增：

```typescript
ensureConversationPlanArtifacts(projectRoot: string, conversationId: string): {
  success: boolean
  planPath: string
  statePath: string
  remarksPath: string
}

loadConversationPlanState(projectRoot: string, conversationId: string): PlanStateRecord | null

saveConversationPlanState(projectRoot: string, conversationId: string, state: PlanStateRecord): { success: boolean }

loadConversationPlanRemarks(projectRoot: string, conversationId: string): PlanRemarkRecord[]

saveConversationPlanRemarks(projectRoot: string, conversationId: string, remarks: PlanRemarkRecord[]): { success: boolean }

deleteConversationPlanArtifacts(projectRoot: string, conversationId: string): { success: boolean }
```

### 1.5 IPC Layer

建议新增：

```typescript
plan:ensureArtifacts
plan:loadState
plan:saveState
plan:loadRemarks
plan:saveRemarks
plan:deleteArtifacts
```

也可以保持精简，复用现有 `files:read/write` 完成文件读写，只额外新增：

```typescript
plan:ensureArtifacts
```

但为保证类型与业务语义清晰，推荐增加专用 IPC。

### 1.6 Command Model

本特性不引入聊天内控制命令协议。

例如以下协议不采用：

```text
PLAN_ACTION
{"action":"resume_execution"}
```

以下动作必须由 UI 触发：

- `Approve and Execute`
- `Resume Execution`
- `Regenerate Plan`

---

## 2. Frontend (Store / Works / Workspace)

### 2.1 appStore Changes

扩展 `Conversation`：

```typescript
interface Conversation {
  id: string
  title: string
  entryType: ConversationType
  activeType: ConversationType
  createdAt: string
  lastActiveAt: string
  runs: Run[]
  messages: ConversationMessage[]
  messagesLoaded?: boolean
  selectedWorkflowId?: string
  selectedAgentId?: string
  generatedTitle?: string
  customTitle?: string

  chatSubmode?: 'default' | 'plan'
  planStatus?: PlanLifecycleStatus
  planPath?: string
  planStatePath?: string
  planRemarksPath?: string
}
```

新增 store actions：

```typescript
setConversationChatSubmode(conversationId: string, submode: 'default' | 'plan'): void
updateConversationPlanMeta(conversationId: string, payload: Partial<Conversation>): void
```

### 2.2 WorksPage Changes

在 `selectedConversation.activeType === 'chat'` 的详情区中：

- 用 Plan 控制面板替代静态文案
- 读取并展示 `plan.state.json` 状态与进度
- 提供：
  - `Plan` 开关
  - `查看/编辑 plan.md`
  - `Regenerate Plan`
  - `同意并执行 Plan`
  - `Resume Execution`
  - `Submit Decision and Continue`
  - `Revise Plan Instead`

启用 Plan 后：

- `plan.md` 应初始化为空草稿
- `remarks.json` 初始化为空数组
- `plan.state.json` 初始化为 `version = 1`、`status = drafting`、`tasks = []`

### 2.3 WorkspacePage Changes

`WorkspacePage` 已经拥有顶部文件 Tab 系统，因此只需增加一个能力：

```typescript
openAliasFileTab(path: string): Promise<void>
```

然后把这个回调传给 `ConversationView`，使 Chat 详情区按钮可以直接打开：

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
```

在 `plan.md` Tab 中增加一个 Review Surface：

- 上方：当前 `plan.md` 的 markdown 预览
- 中间：悬浮评论入口
- 下方：已有 remarks comment 卡片列表，包含目标章节、类型和时间
- 备注录入通过评论对话框完成

预览区交互补充：

- 鼠标悬停到可评论块时，`Add Comment` 应相对当前内容块定位
- 点击后自动选择关联章节
- 点击后弹出评论对话框
- 当前块的摘要文本可作为 `target.excerpt` 写入 remark

### 2.4 Right Panel Data Loading

右侧详情区不显示 `plan.md` 正文预览，因此只需要：

1. 读取 `plan.state.json` 作为状态与进度来源
2. 读取会话 metadata 作为按钮和状态机控制依据
3. 如有需要，读取 `remarks.json` 仅用于展示备注数量或结构化列表，而不是展示正文内容

---

## 3. Chat Context Injection

### 3.1 Why

当前聊天上下文来自：

- `messages.json`
- `extraSystemMessages`
- `dataContext`

Plan 模式下，需要让 LLM 自动理解：

- 当前计划内容
- 当前计划状态
- 备注内容

### 3.2 Injection Strategy

在 `callAgentChat()` 中，如果：

- `activeType === 'chat'`
- `chatSubmode === 'plan'`

则在构造 `buildChatContextMessages()` 前额外注入一个或两个系统消息：

```text
PLAN_CONTEXT
- status: reviewing
- planPath: @state/projects/<projectId>/plans/<conversationId>/plan.md
- remarksPath: @state/projects/<projectId>/plans/<conversationId>/remarks.json

Current Plan:
...

Current Remarks:
...
```

原则：

- 不修改固定 base system template 去承载整份 `plan.md`
- 只在当前会话开启 `chatSubmode = plan` 时动态注入
- `PLAN_CONTEXT` 应以当前 live `plan.md`、`remarks.json`、`plan.state.json` 为准
- 在执行态还应额外注入 `PLAN_EXECUTION_TOOL_RULES`，明确要求使用受控 `plan.*` 工具推进状态

### 3.4 Chat Input Semantics

Plan Mode 下，聊天输入的默认解释必须结合当前 `plan.state.json.status` 与任务状态：

```text
drafting/reviewing + send
  -> treat as authoring or modifying current plan

executing + waiting_user + send
  -> treat as decision input for current waiting task

executing + waiting_user + click Regenerate Plan
  -> treat current chat draft as plan refinement request

completed + send
  -> increment version and start a new plan draft
```

实现建议：

- `send` 事件本身不做显式生命周期切换按钮替代
- 只有在 `completed` 时，发送行为允许触发“新版本起草”
- `Regenerate Plan` 点击时，优先读取当前聊天输入框内容并将其作为计划优化指令

### 3.5 Version Rollover

当当前 Plan 全部完成后，下一次聊天输入不应覆盖当前版本，而应：

1. `plan.state.json.version += 1`
2. `status = drafting`
3. 清空或重建 `tasks[]`
4. 保留上一次版本的已完成时间和历史 remark 记录策略

### 3.3 Prompt Rule

Plan 模式下应额外增加一条 system instruction：

- 优先基于当前 `plan.md` 进行增量优化
- 不要把计划重新拆成完全无关的新方案
- 若用户点击执行，则进入更新 `plan.state.json` 的执行态

### 3.4 User Input Semantics in Plan Mode

Chat 输入框中的自由文本只用于：

- 计划修订
- 计划解释
- 备注补充
- 执行中决策输入
- 执行结果讨论

它不直接触发生命周期状态迁移。

---

## 4. Plan Execution Progress Model

### 4.1 Why not reuse Workflow Progress

现有 workflow progress 来自 run state，语义是“预定义 graph 节点执行”。

Plan execution 的语义是“文档驱动计划任务状态”，因此必须独立建模。

### 4.2 Progress Source

Plan 进度统一从 `plan.state.json.tasks[]` 派生。

关键规则：

- `tasks[]` 是执行阶段唯一的结构化任务真源
- 执行阶段不得每轮动态解析 `plan.md` 来决定任务列表
- 在计划成型或计划重建时，应同步生成/更新 `tasks[]`
- 若 `tasks[]` 缺失或与当前版本不一致，则在批准执行前进行一次规范化重建

UI 计算：

```typescript
completedCount = tasks.filter(t => t.status === 'completed').length
progressPercent = tasks.length ? Math.round(completedCount / tasks.length * 100) : 0
```

### 4.3 Optional Eventing

执行态不应继续依赖“最后一条 assistant 文本”作为唯一状态推进来源，而应增加受控执行工具：

```typescript
plan.get_state
plan.complete_current_task
plan.wait_for_user
plan.block_current_task
```

约束：

- 这些工具只能作用于当前 conversation 的 live `plan.state.json`
- LLM 不得直接 `fs.write` / `fs.apply_patch` 修改 `plan.state.json`
- 所有状态迁移必须由后端校验合法性
- `plan.complete_current_task` 负责完成当前 task，并在可能时激活下一个可执行 task
- `plan.wait_for_user` / `plan.block_current_task` 必须携带明确原因

同时增加显式推送事件：

```typescript
plan:state-changed
```

要求：

- 后端在通过 `plan.*` 工具、`savePlanDraft`、`savePlanState` 或 `ensurePlanArtifacts` 改变 live state 后，主动向 Renderer 推送
- `WorksPage` 应订阅该事件，并优先采用后端推送的 live state 更新右侧 Plan UI
- 只有在后端未更新状态时，前端才允许执行本地 fallback 状态推进

因此，MVP 不再采用“仅在进入会话 / 切换状态时重新拉取”的弱同步方案。

### 4.4 Execution Hard Constraints

执行态必须增加以下硬约束：

- 禁止创建或更新 `@project/*计划.md`、`@project/*任务.md`、`@project/*plan.md`、`@project/*task.md` 作为进度追踪影子文档
- 若用户需要最终交付物正文，只允许写入 `Deliverables` 已声明的目标文件
- `plan.md` 与 `plan.state.json` 仍是唯一计划真源；任何聊天消息、execution log 或影子文档都不得替代它们

### 4.5 Waiting for User Decision

当某个任务需要用户决策时：

```typescript
task.status = 'waiting_user'
plan.status = 'executing'
```

系统行为：

1. 右侧 Plan 面板显示当前等待任务与等待原因
2. 用户在 Chat 输入框补充内容
3. Runtime 将该输入解释为当前 waiting task 的决策输入
4. 当前 task 恢复 `in_progress`
5. 执行继续向下推进

默认不执行重新编写整份计划，除非用户显式点击对应 UI 动作。

---

## 5. Remarks Writing Model

### 5.1 Minimal Viable Remarks

备注不直接写入 `plan.md` 主体内容，而是写入 `remarks.json`。

优点：

- 避免污染主文档
- 便于后续让 LLM读取“文档 + 备注”双上下文
- 方便独立删除和审计

### 5.2 Optional Mirror Section

如果需要让用户在 `plan.md` 中看到备注，也可以让 UI 保存时同步更新一个：

```markdown
## Remarks
- ...
```

但这不应替代 `remarks.json` 作为系统真实来源。

---

## 6. Post-Execution Conversation Rules

### 6.1 Completed Plan

当 `plan.state.json.status = completed` 时：

- 若用户仍处于 Plan Mode，则下一次发送内容默认解释为“创建 `version + 1` 的新计划草稿”
- 不自动再次执行
- 不覆盖当前已完成版本

### 6.2 New Revision

若用户需要基于当前结果修改计划：

- 应先进入新版本，再进行计划修订
- 一旦任务语义发生实质变化，应进入新版本或重新审批流程

### 6.3 Resume Execution

只有在：

- `plan.status = executing | blocked`
- 且存在可继续推进的任务

时，`Resume Execution` 才应作为可点击 UI 动作出现。

补充规则：

- 若当前不存在 `waiting_user` task，则 `Resume Execution` 可直接触发恢复执行
- 若当前存在 `waiting_user` task，则主按钮区的 `Resume Execution` 应禁用
- 此时必须改由等待卡片中的 `Submit Decision and Continue` 提交决策
- 等待卡片还应提供 `Revise Plan Instead`，使用户显式转入计划修订路径

---

## 7. Cleanup Rules

### 6.1 Delete Conversation

删除 conversation 时：

1. 删除 `conversations/<conversationId>/`
2. 删除 `@state/projects/<projectId>/plans/<conversationId>/`
3. 更新 `conversations/index.json`

### 6.2 Delete Project / Orphan Cleanup

删除项目或清理孤立项目时：

1. 删除整个：

```text
$RUNTIME_DATA/projects/<projectId>/
```

或至少删除：

```text
state/projects/<projectId>/
conversations/
runs/
```

Plan 工件必须纳入这一清理范围。

---

## 8. File Changes

| File | Action | Description |
|---|---|---|
| `electron/stores/runtimeStore.ts` | MODIFY | 新增 Plan 工件路径与读写方法 |
| `electron/main.ts` | MODIFY | 新增 Plan IPC；Chat 上下文注入 |
| `electron/preload.ts` | MODIFY | 暴露 Plan IPC |
| `electron/electron-env.d.ts` | MODIFY | 类型定义 |
| `src/stores/appStore.ts` | MODIFY | Conversation 扩展与 Plan store actions |
| `src/hooks/useConversationWorkspace.ts` | MODIFY | 暴露 Plan 会话操作 |
| `src/pages/WorksPage/WorksPage.tsx` | MODIFY | Plan 面板、执行入口 |
| `src/pages/WorksPage/WorksPage.css` | MODIFY | Plan 面板样式 |
| `src/pages/WorkspacePage/WorkspacePage.tsx` | MODIFY | 打开 alias file tab 能力 |
| `src/components/MarkdownEditor/*` | OPTIONAL | 若备注 UI 需要扩展工具栏或侧栏 |

---

## 9. Testing

### 9.1 Story 5-20-1

- 首次开启 Plan，创建项目级目录和 3 个工件文件
- 会话重开后恢复 metadata
- 删除 conversation 时只删该 conversation 的 Plan 子目录

### 9.2 Story 5-20-2

- Chat 详情区正确切换普通 Chat / Plan 面板
- 状态与进度来自 `plan.state.json`
- 右侧不显示 `plan.md` 正文预览

### 9.3 Story 5-20-3

- 点击按钮在顶部打开 `plan.md`
- 编辑保存成功
- 备注成功写入 `remarks.json`
- 顶部 Tab 中显示计划预览与备注输入区
- 发送下一条 Chat 消息时，LLM 上下文包含 Plan 与备注

### 9.4 Story 5-20-4

- 点击执行后 `plan.state.json.status = executing`
- 任务进度正确渲染
- 重启应用后进度恢复
- 执行中 `waiting_user` 任务收到用户输入后继续当前 task
- `completed` 状态下继续聊天不会自动再次执行
- 执行态仅暴露 `plan.*` 工具，drafting/reviewing 不暴露
- 通过 `plan.*` 工具更新状态后，`plan:state-changed` 会推动 `WorksPage` UI 同步刷新
- 执行态写入 `@project/*计划.md` / `@project/*任务.md` 会被硬拦截

---

## 10. Risks

### Risk 1: Plan 与 Chat 消息边界混乱

缓解：

- Plan 工件独立存储
- Chat 仅保存消息历史

### Risk 2: 编辑器备注能力过度扩张

缓解：

- v1 只做结构化备注
- 行级批注延后

### Risk 3: 清理逻辑遗漏 Plan 数据

缓解：

- 将 Plan 工件路径纳入项目级命名空间
- Story 5-20-1 强制覆盖 cleanup 验收
