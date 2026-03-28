# Design: Story 5-20 – Chat Plan Mode

**Story Group:** `5-20-chat-plan-mode`  
**设计原则:** 项目优先、Chat 内聚、文档即工件、最小侵入、状态可恢复

---

## 设计目标

1. **不新增顶层会话类型**：保留现有 `workflow / agent / chat` 结构。
2. **让 Plan 成为 Chat 的工作层**：在 Chat 详情区中可打开、执行、查看进度。
3. **让 `plan.md` 成为真实工件**：而不是一段消息文本。
4. **保证项目级隔离**：所有 Plan 工件优先按项目归档。
5. **支持前后端闭环验收**：每个 Story 都必须有可以独立测试的 UI + 状态 + 存储结果。

---

## 交互定位

### 为什么不是第四个顶层会话类型

当前 Works 页面顶层仅有：

- `workflow`
- `agent`
- `chat`

如果新增 `plan` 顶层类型，需要改动：

- 会话类型切换按钮
- 创建会话入口
- 多处 `activeType === 'chat'` / `agent` / `workflow` 分支
- Works、Workspace、Store 的既有状态模型

这会扩大改动面，并破坏“Plan 是 Chat 内协作步骤”的产品语义。

### 目标交互

目标不是“新建一个 Plan 会话”，而是：

```text
Chat 会话
  └── Plan 开关
        ├── 关闭：普通 Chat
        └── 打开：Chat Plan Mode
```

因此本次设计采用：

- 顶层仍然是 `chat`
- 子模式为 `chatSubmode = 'plan'`
- Plan UI 只在 Chat 详情区与顶部 Tab 中出现
- 所有 Plan 生命周期变化都通过显式 UI 动作触发

---

## 信息架构

### Works 页面

#### 左侧

- Conversation 列表，不变

#### 中间

- Chat 消息流，不变

#### 右侧

- 普通 Chat：显示现有 Chat details
- Plan 模式：显示 Plan 控制面板

### Workspace 顶部 Tabs

- `Conversation` 标签，不变
- 新增文件标签：`plan.md`

`plan.md` 并不是新页面，而是沿用现有 Workspace 顶部文件 Tab 机制打开。

---

## Plan 面板布局

### 状态 1：Plan 未开启

显示：

- `Plan` 开关（关闭态）
- 状态标签：`default`

### 状态 2：Plan 已开启但未执行

显示：

- `Plan` 开关
- 生命周期标签：`Drafting` / `Reviewing`
- `查看/编辑 plan.md` 按钮
- `Regenerate Plan` 按钮
- `同意并执行 Plan` 按钮
- 计划状态与进度概览

补充规则：

- 打开 `Plan` 开关时，`plan.md` 初始为空草稿，不自动写入默认模板正文。
- 用户在聊天框中的输入，默认用于编写或修改当前计划。

### 状态 3：Plan 执行中

显示：

- `Plan` 开关
- 生命周期标签：`Executing`
- 当前阶段卡片
- 任务进度条
- 任务列表
- `查看/编辑 plan.md` 按钮
- `Regenerate Plan` 按钮
- 若存在等待用户决策：显示等待任务与继续提示
- `Resume Execution` 次级动作
- 最近更新时间

补充规则：

- 若当前存在 `waiting_user` 任务，则聊天框输入默认解释为该任务的决策输入。
- 若用户此时点击 `Regenerate Plan`，则当前聊天框输入被解释为计划优化要求，而不是任务决策。

### 状态 4：Plan 已完成或阻塞

显示：

- 生命周期标签：`Completed` / `Blocked`
- 任务完成率
- 可选的下一步操作按钮

补充规则：

- 当前 Plan 进入 `Completed` 后，用户下一次在聊天框输入并发送内容，应开启 `version + 1` 的新 Plan 草稿。

---

## 顶部 Tab 行为

### 打开方式

用户点击右侧面板中的 `查看/编辑 plan.md`：

1. 若 Tab 已存在，则切换到该 Tab
2. 若 Tab 不存在，则在 Workspace 顶部新建一个文件 Tab
3. 文件来源路径固定为：

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
```

### 编辑方式

- 直接复用现有 Markdown Editor
- 支持预览/编辑切换
- 支持保存
- 保存后回到 Conversation 时，不要求显示正文预览
- 在 `plan.md` 顶部 Tab 中，提供“文档预览 + 悬浮评论入口 + 评论弹窗”的审阅区，样式参考评论式审阅卡片
- `plan.md` 顶部 Tab 负责审阅与备注，不负责切换 Plan 生命周期状态

---

## 备注设计

### v1 备注模型

本次不实现真正的行级评论系统，采用结构化备注列表。

每条备注支持：

- `content`
- `target.kind`: `document | section`
- `target.heading?`
- `createdAt`

这允许用户表达：

- “整篇计划补充风险项”
- “给《实施步骤》章节补充回滚策略”

### 录入入口

在 `plan.md` Tab 中提供 `Add Comment` 操作，通过预览区域悬浮入口触发：

- 鼠标悬停到可评论块时，在预览左侧浮出 `Add Comment`
- 点击后弹出评论对话框
- 对话框内显示目标章节和摘录内容
- 输入备注内容
- 保存到 `remarks.json`

补充交互：

- 当鼠标悬停在预览中的可评论块上时，`Add Comment` 图标应跟随当前内容块出现
- 点击后自动定位到相关章节
- 同时把当前块的摘要文本带入备注上下文，便于形成“针对该部分”的 comment

备注列表展示为 comment 卡片，至少包含：

- remark type：`Document` / `Section`
- target heading
- content
- createdAt

### 为什么不做行级锚点

原因：

- 现有 Markdown Editor 不具备注释锚点模型
- 文本变更后锚点稳定性难保证
- 会显著扩大数据模型和编辑器改造范围

v1 目标是先实现“计划文档 + 结构化备注 + Chat 回流优化”的核心闭环。

---

## 状态流设计

### Conversation 级状态

```text
chat
  └── chatSubmode = default | plan
```

### Plan 生命周期状态

```text
drafting -> reviewing -> executing -> completed
                         └-> blocked
```

### 任务级状态

```text
pending -> in_progress -> waiting_user -> in_progress -> completed
                                 └-> blocked
```

### 常见流转

#### 初始化

```text
Chat
  -> 用户打开 Plan 开关
  -> 创建 Plan 工件
  -> plan.md = 空白草稿
  -> planStatus = drafting
```

#### 编辑与备注

```text
drafting/reviewing
  -> 用户在聊天框中输入内容
  -> 生成或修改 plan.md
  -> 打开 plan.md 审阅
  -> 增加备注
  -> 返回 Chat
  -> LLM 继续优化
```

#### 批准执行

```text
drafting/reviewing
  -> 用户点击同意并执行
  -> planStatus = executing
  -> 展示进度
```

#### 执行中等待决策

```text
executing + waiting_user
  -> 用户在聊天框中输入内容并发送
  -> 默认解释为当前任务决策输入
  -> 继续当前 task
```

```text
executing + waiting_user
  -> 用户在聊天框中输入内容
  -> 点击 Regenerate Plan
  -> 当前输入改为计划优化要求
  -> 返回计划修订流
```

#### 执行完成后新版本

```text
completed
  -> 用户发送下一条聊天消息
  -> version + 1
  -> 进入新的 drafting
```

#### 执行中等待用户决策

```text
executing
  -> 某个 task 进入 waiting_user
  -> 右侧显示等待说明
  -> 用户在 Chat 输入补充内容
  -> 默认继续当前 task 执行
```

#### 执行完成后开启新版本

```text
completed
  -> 用户发送下一条聊天消息
  -> version + 1
  -> 进入新的 drafting/reviewing
```

---

## Action Model

### UI-Only Lifecycle Actions

以下动作只允许由 UI 触发：

- Enable / Disable Plan Mode
- View / Edit `plan.md`
- Add Remark
- Regenerate Plan
- Approve and Execute
- Resume Execution
- Submit Decision and Continue
- Revise Plan Instead

### Chat Input Semantics

Chat 输入框只负责：

- 计划讨论
- 计划修订说明
- 备注补充说明
- 执行中的补充决策
- 执行结果追问

Chat 输入框不负责：

- 直接触发 Plan 生命周期状态迁移
- 发送控制协议或命令 envelope

因此本设计明确不采用聊天内控制命令协议。

---

## 存储设计原则

### 工件路径

```text
@state/projects/<projectId>/plans/<conversationId>/plan.md
@state/projects/<projectId>/plans/<conversationId>/plan.state.json
@state/projects/<projectId>/plans/<conversationId>/remarks.json
```

### 原则

1. `projectId` 一级归档
2. `conversationId` 二级归档
3. `plan.md` 面向用户
4. `plan.state.json` 面向系统状态与进度
5. `remarks.json` 面向结构化备注

---

## 与现有系统的关系

### 与聊天消息持久化的关系

- 聊天消息继续存放在 `conversations/<conversationId>/messages.json`
- Plan 工件单独存放在 `@state/projects/<projectId>/plans/...`
- 两者职责分离：
  - messages = 对话历史
  - plan artifacts = 计划工件与执行状态

### 与 Chat 上下文的关系

- `plan.md` 不直接成为固定系统模板的一部分。
- Runtime 在每轮请求前动态注入当前计划上下文。
- 这样既能保持系统模板稳定，也能保证计划内容始终使用最新版本。

### 与日志系统的关系

- Plan 状态不进入运行期内存日志数组
- 调试日志仍然走现有 execution log
- Plan 进度以 `plan.state.json` 为准

---

## UI 风格方向

### 视觉方向

Plan 面板应与现有暗色 Runtime 风格一致，但要比普通 Chat 详情区更具“工作台”感：

- 明确的状态色
- 高对比的主按钮
- 任务卡片而不是纯文本块
- 让用户一眼知道现在是在“计划阶段”还是“执行阶段”

### 组件建议

- `Plan toggle`
- `Status badge`
- `Recent remarks list`
- `Progress bar`
- `Task checklist card`

---

## 失败与恢复设计

### 文件缺失

若 `plan.md` 或 `plan.state.json` 缺失：

- 提示 Plan 工件不完整
- 提供 `重新初始化 Plan` 操作

### 路径冲突

如果 conversation metadata 指向不存在文件：

- 进入降级状态
- 保留 Chat 可用
- 提示用户修复或重建 Plan

### 应用重启恢复

恢复优先级：

1. Conversation Metadata
2. `plan.state.json`
3. `plan.md`
4. `remarks.json`

---

## Story 对应关系

- Story 5-20-1：项目级存储与会话元数据
- Story 5-20-2：右侧 Plan 面板与控制区
- Story 5-20-3：顶部 Tab 编辑与备注
- Story 5-20-4：批准执行与进度可视化

本设计文档作为这四条 Story 的统一 UI 与状态设计基线。
