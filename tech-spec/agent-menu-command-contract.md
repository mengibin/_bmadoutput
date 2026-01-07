# CrewAgent Runtime — Agent Menu & Command Routing Contract (v1)

> 目的：定义 **Agent Session** 下，如何根据 `agents.json`（见 `crewagent-runtime/spec/bmad-package-spec/v1.1/schemas/agents.schema.json`）中的 `menu` 与用户输入，确定要执行的命令（workflow/exec/action/data/multi），并把命令路由到 **WorkflowRun / ScriptRun / 内建动作**。
>
> 本契约优先对齐 BMAD 的行为模型（参考 `_bmad/bmm/agents/analyst.md` 里的 menu/handler 语义），但实现上建议由 Runtime 做确定性解析与调度，LLM 只负责“对话与产出”。

---

## 1. 范围与非目标

### 1.1 范围（本契约覆盖）

- Agent 的选择与激活（进入 Agent Session）
- 用户输入的解析与匹配（数字选择 / trigger / 模糊匹配 / 多命中澄清）
- `menuItem` 字段语义：
  - `workflow`（启动 v1.1 micro-file + graph 工作流）
  - `exec`（执行脚本入口 Markdown）
  - `action`（内建动作或 agent prompt）
  - `data`（为 workflow/exec/action 提供上下文）
  - `triggers`（用于扩展匹配与 multi-handler）
- 运行态：Agent Session 与 Run（WorkflowRun/ScriptRun）的切换与优先级

### 1.2 非目标（本契约不覆盖）

- 经典 BMAD `workflow.yaml + workflow.xml` 执行（MVP 不支持；若遇到需显式报错）
- `.bmad` 包导入、schema 校验、graph 执行细节（见 runtime 架构文档）
- LLM Provider 的差异（由 LLMAdapter 处理）

---

## 2. 术语

- **Agent**：`agents.json` 中的一个 agent 定义（id、persona、menu、tools 等）。
- **Agent Session**：用户选定一个 Agent 后的交互模式；输入优先用于菜单命令解析，未命中则回退到“聊天/澄清”。
- **menuItem**：`agent.menu[]` 的一项（schema 允许 additionalProperties）。
- **Command**：Runtime 解析后的结构化命令（可执行），例如 `StartWorkflow`、`ExecScript`、`RunAction`。
- **Run**：一次可暂停/恢复的执行实例（WorkflowRun 或 ScriptRun）。
- **ProjectRoot**：用户工程目录（产物默认写入这里）。
- **RuntimeStoreRoot**：应用私有目录（`.bmad` 解包缓存、state、logs 存放于此，不在 Project 内）。

---

## 3. 数据模型（Runtime 内部契约）

> 说明：以下是 Runtime 内部结构，不要求写入 `.bmad`，但要求“可从 schema 字段无损映射”。

### 3.1 输入与上下文

```ts
type InputText = string;

type TargetSurface = 'electron' | 'web';

interface AgentSessionContext {
  projectRoot: string;              // 真实路径，仅 Runtime 可见
  runtimeStoreRoot: string;         // 真实路径，仅 Runtime 可见
  targetSurface: TargetSurface;     // 用于 ide-only/web-only gating
  activeAgentId: string;
  activeRunId?: string;             // 当前正在运行的 run（若有）
  locale?: string;
}
```

### 3.2 解析输出：ResolvedCommand

```ts
type CommandKind =
  | 'ShowMenu'
  | 'StartWorkflow'
  | 'ResumeRun'
  | 'ExecScript'
  | 'RunAction'
  | 'Chat'
  | 'DismissAgent'
  | 'ClarifyChoice';

interface ResolvedCommandBase {
  kind: CommandKind;
  confidence: 'exact' | 'high' | 'medium' | 'low';
  reason?: string;
  matchedMenuItemIndex?: number; // 1-based
  matchedMenuItemId?: string;    // 若你在 additionalProperties 中引入 id（可选）
}

interface StartWorkflowCommand extends ResolvedCommandBase {
  kind: 'StartWorkflow';
  workflowRef: WorkflowRef;
  dataRef?: DataRef;
}

interface ExecScriptCommand extends ResolvedCommandBase {
  kind: 'ExecScript';
  execRef: ExecRef;
  dataRef?: DataRef;
}

interface RunActionCommand extends ResolvedCommandBase {
  kind: 'RunAction';
  actionRef: ActionRef;
  dataRef?: DataRef;
}

interface ClarifyChoiceCommand extends ResolvedCommandBase {
  kind: 'ClarifyChoice';
  candidates: Array<{ index: number; label: string }>;
}

type ResolvedCommand =
  | (ResolvedCommandBase & { kind: 'ShowMenu' | 'ResumeRun' | 'Chat' | 'DismissAgent' })
  | StartWorkflowCommand
  | ExecScriptCommand
  | RunActionCommand
  | ClarifyChoiceCommand;
```

### 3.3 Ref 类型：WorkflowRef / ExecRef / DataRef / ActionRef

#### 3.3.1 PathTemplate

为兼容 BMAD 风格（如 `{project-root}`），定义运行时路径模板：

```ts
type MountAlias = '@project' | '@pkg' | '@state';

type PathTemplate = string; // 允许包含：{project-root} / {package-root} / {state-root} / {artifacts-root}
```

模板变量（MVP 必须支持）：

- `{project-root}` → `@project`
- `{artifacts-root}` → `@project/artifacts`
- `{package-root}` → `@pkg`
- `{state-root}` → `@state`

> 说明：LLM 永远只看到 mount alias，Runtime 负责映射到真实路径，避免泄露 RuntimeStoreRoot。

#### 3.3.2 WorkflowRef

```ts
type WorkflowRef =
  | { type: 'workflowId'; workflowId: string }                 // 优先：匹配 bmad.json.workflows[].id
  | { type: 'packagePath'; workflowMdPath: PathTemplate };     // 指向 workflow.md（相对 package-root）
```

解析顺序（规范）：

1. 若 `menuItem.workflow` 等于某个 `bmad.json.workflows[].id` → `workflowId`
2. 否则若 `menuItem.workflow` 以 `.md` 结尾 → `packagePath`（相对 `{package-root}`，或显式带 `{package-root}`）
3. 否则 → 视为 `workflowId`（但若找不到则报错：UnknownWorkflow）

#### 3.3.3 ExecRef

```ts
type ExecRef =
  | { type: 'markdown'; mdPath: PathTemplate }                 // 需要 LOAD/READ 并执行其指令
  | { type: 'workflowMd'; workflowMdPath: PathTemplate };      // exec 指向一个 workflow.md 时，等同 StartWorkflow
```

> 判断 `workflowMd` 的规则：文件名为 `workflow.md` 或 frontmatter 能解析到 `schemaVersion/workflowType/currentNodeId`。

#### 3.3.4 DataRef

```ts
type DataRef =
  | { type: 'path'; path: PathTemplate; parseAs?: 'text' | 'json' | 'yaml' | 'csv' | 'xml' };
```

默认解析策略：
- `.json` → json
- `.yaml|.yml` → yaml
- `.csv` → csv
- `.xml` → xml（原文 + 可选 DOM/简单结构）
- 其他 → text

#### 3.3.5 ActionRef

```ts
type ActionRef =
  | { type: 'builtin'; name: string }      // 例如: menu.show / agent.dismiss / run.resume
  | { type: 'promptId'; id: string }       // action="#id" → agent.prompts[].id
  | { type: 'inline'; text: string };      // action="some instruction"
```

---

## 4. menuItem 字段语义（规范）

> 参考 schema：`trigger` / `triggers` / `description` / `workflow` / `exec` / `action` / `data` / `validate-workflow` / `ide-only` / `web-only`。

### 4.1 `trigger`（单触发词）

- 用于 **精确命中**：大小写不敏感，忽略前后空格。
- 推荐格式：kebab-case；但需兼容 BMAD 风格的 `*something` 或 `[M]`。

### 4.2 `triggers`（扩展触发与 multi-handler）

`triggers` 在 schema 中是“对象数组（additionalProperties true）”，本契约规定两种用途：

1) **同义触发**：`{ type: 'alias', match: '...' }`
2) **multi-handler**：`{ type: 'handler', match: '...', workflow|exec|action, data? }`

建议的 handler 结构（约定，不由 schema 强制）：

```json
{
  "type": "handler",
  "match": "SPM or fuzzy match start party mode",
  "exec": "{project-root}/.../workflow.md",
  "data": "..."
}
```

### 4.3 `description`

- 用于 UI 展示与模糊匹配文本池的一部分。

### 4.4 `workflow`

`workflow` 表示“启动一个工作流”。MVP 仅支持 **v1.1 micro-file + graph**：

- ✅ 支持：
  - `workflowId`（匹配 `bmad.json.workflows[].id`）
  - 指向 `workflow.md` 的路径（`.md` 结尾；默认相对 `{package-root}`）
- ❌ 不支持：
  - `workflow.yaml/.yml`（经典 BMAD workflow）。必须返回 `NotSupportedClassicWorkflow`，并提示用户该命令需要 classic runner（未来扩展）。

执行效果：

- 解析为 `StartWorkflowCommand`（见 3.2）
- 若同时存在 `data`：先加载/解析 data，再启动 workflow

### 4.5 `exec`

`exec` 表示“执行一个 Markdown 脚本入口文件”。

支持两种模式：

1) **ExecScript（默认）**：`exec` 指向普通 `.md` 文件  
   - Runtime 以 `ExecScriptCommand` 路由到 ScriptRun
   - LLM 需要 LOAD/READ 该文件并遵循其指令

2) **StartWorkflow（兼容）**：`exec` 指向 `workflow.md`（或文件 frontmatter 可识别为 workflow）  
   - Runtime 将其提升为 `StartWorkflowCommand`
   - graph 解析规则：同目录/同 workflowRef 解析到 `workflow.graph.json`

若 `exec` 指向 `.xml/.yaml`：MVP 一律 `NotSupportedClassicWorkflow`（避免“伪执行”）。

### 4.6 `data`

`data` 表示“为 workflow/exec/action 提供上下文输入”。

规范要求：

- `data` 必须在执行 handler 之前加载（BMAD `_bmad` 的 handler 行为）
- 加载后的 data 必须以 **显式消息**注入给 LLM（而不是隐式记忆）
- 可选：把 data 的引用路径追加到 workflow state 的 `inputDocuments`（若你选择在 state 中保存）

解析与注入建议（MVP）：

- 若解析为结构化对象（json/yaml/csv）：注入“摘要 + 可选原文片段”，并提供 `fs.read` 让 LLM按需读取
- 若为纯文本：注入前 N KB（受 toolPolicy 限额），剩余部分提示 LLM 用 `fs.read` 分段

### 4.7 `action`

`action` 表示“执行一个动作”，按优先级解析：

1) `action` 以 `#` 开头：`#id` → 查找 `agent.prompts[]` 中 `id == "id"`  
   - 找到 → `ActionRef: promptId`
   - 找不到 → 返回 UnknownPromptId

2) `action` 是已注册内建动作名：如 `menu.show` / `agent.dismiss` / `run.resume`  
   - `ActionRef: builtin`

3) 其他字符串：作为 `ActionRef: inline`（由 LLM 执行的“内联指令”）  
   - 仅建议用于轻量、无状态的动作（例如“以该 persona 回答用户一个问题”）

### 4.8 `validate-workflow`

`validate-workflow` 用于启动前校验（字段名保持 BMAD 风格）：

- 若存在，Runtime 必须在执行前验证：
  - workflowRef 可解析
  - 对于 StartWorkflow：graph + workflow.md + steps 存在且可读
- 校验失败时，禁止启动，并给出可操作的错误（缺哪个文件/哪个 id）。

### 4.9 `ide-only` / `web-only`

用于 UI/路由 gating：

- `ide-only: true` 且 `targetSurface == 'web'` → 不展示且不参与匹配
- `web-only: true` 且 `targetSurface == 'electron'` → 不展示且不参与匹配
- 两者都为 true：视为无效配置（Runtime 可在导入时告警）

---

## 5. 命令解析与匹配（规范算法）

### 5.1 输入预处理（MUST）

- `input = input.trim()`
- 若为空字符串：返回 `ShowMenu`
- 标准化（用于匹配，不改变原始输入）：
  - `lower = input.toLowerCase()`
  - 允许移除前导 `*`（BMAD cmd 风格）用于匹配：`"*menu" -> "menu"`

### 5.2 菜单编目（MUST）

给定 `agent.menu`，Runtime 必须生成“可匹配候选集合”：

- **MenuItemCandidate**：每个 menuItem 生成 1 个候选
- **HandlerCandidate**：若 menuItem.triggers 中存在 `{type:'handler'}`，每个 handler 额外生成 1 个候选（multi-handler）

候选的 label（用于澄清）推荐为：`"<index>. <description>"`

### 5.3 选择优先级（MUST）

按顺序尝试：

1) **数字选择**：`/^[0-9]+$/`  
   - `n` 在可见菜单范围内 → 直接命中第 n 项（confidence=exact）  
   - 否则 → 返回 `ClarifyChoice`（提示有效范围）

2) **精确 trigger 命中**（confidence=exact）  
   对每个候选，构造 triggers 文本池（按顺序）：
   - `menuItem.trigger`（若存在）
   - `menuItem.cmd`（若存在；为了兼容 `_bmad/.../agents/*.md` 的 cmd 属性）
   - `menuItem.triggers[]` 中 `type:'alias'` 的 `match`
   - （可选增强）从 `description` 中提取形如 `[M]` 的单字符 code 作为 alias
   若 `normalize(input)` 与任一 trigger 相等 → 命中

3) **模糊匹配**（confidence=high/medium/low）  
   - 匹配池：`trigger/cmd/alias/description/handler.match`
   - 最低要求：大小写不敏感 substring 命中（MVP）
   - 评分（建议）：
     - 完整词匹配 > 子串匹配
     - 命中 trigger/cmd > 命中 description
     - handler.match 命中 > 仅命中 multi 的父项描述
   - 若最高分唯一且超过阈值 → 命中
   - 若并列多命中 → 返回 `ClarifyChoice`（列出候选）

4) **未命中**：返回 `Chat`（让 agent persona 进行澄清/对话），并可附带建议：“输入数字或 trigger”。

### 5.4 多命中澄清（MUST）

当返回 `ClarifyChoice`：

- UI 必须展示候选列表（带编号）
- 下一次用户输入若为数字，则直接按数字选择（回到 5.3-1）

---

## 6. 命令执行语义（从 ResolvedCommand 到行为）

### 6.1 ShowMenu

- UI 渲染 `agent.menu` 的可见项（按原顺序）
- UI 文案建议：按钮主标签优先使用 `menuItem.trigger`（fallback 为 description），`description` 仅作为说明（tooltip/副标题）
- 不调用 LLM（除非你希望 LLM 用 persona 复述菜单；推荐 UI 直接渲染，减少 token）

### 6.2 StartWorkflow

- `RunManager.createRun(...)`
- 初始化 `@state/workflow.md`（从 package workflow.md 拷贝/或从路径解析）
- 启动 `WorkflowRun` 的 ExecutionEngine（ToolCalls loop）
- 将默认产物目录设为 `@project/artifacts/`
- Prompt 组装示例：见 `_bmad-output/tech-spec/prompt-composer-examples.md`

### 6.3 ExecScript

- 创建 `ScriptRun`（可选是否可恢复；MVP 可不做 state，只做 logs）
- PromptComposer 注入：
  - agent persona（activeAgentId）
  - exec 文件内容（由 LLM 通过 `fs.read` 获取）
  - data（若存在）

### 6.4 RunAction

- builtin：
  - `menu.show` → ShowMenu
  - `agent.dismiss` → 退出 Agent Session（清空 activeAgentId / activeRunId）
  - `run.resume` → ResumeRun（若存在可恢复 run）
- promptId：
  - 将 prompt.content 作为一次性“指令”注入给 LLM（可配合工具）
- inline：
  - 作为 user message 注入给 LLM 执行（建议限制为无副作用场景）

### 6.5 ResumeRun

- 从 RuntimeStore 列出最近 run（或指定 runId）
- 读取 `@state/workflow.md`，恢复 ExecutionEngine

### 6.6 Chat（fallback）

- 不启动 workflow
- 以 agent persona 对话
- 工具策略：默认开启所有可用 tools；由 Settings 的 SystemToolPolicy 控制系统级开关/限额，agent.tools 可进一步收紧（如需更安全，可在 UI 侧对写入类 tool 增加确认/差异展示）

---

## 7. 错误模型（推荐标准化）

建议所有失败返回结构化错误（便于 UI 提示与日志审计）：

- `UnknownWorkflow`
- `NotSupportedClassicWorkflow`
- `UnknownPromptId`
- `DataLoadFailed`
- `PermissionDenied`（mount/sandbox）
- `ValidationFailed`（validate-workflow）

---

## 8. Agent Session 与 Run 的输入优先级（建议约束）

为避免“运行中误触菜单/聊天误改项目”，建议把输入通道明确成两种 mode：

- **AgentMode**：未启动 run（或用户显式退出 run）
  - 输入优先走 CommandRouter
  - 未命中 → Chat（以 agent persona 对话）

- **RunMode**：存在 `activeRunId`
  - 默认把输入交给 ExecutionEngine（作为 workflow 的用户输入）
  - 仅当输入满足“全局命令格式”时才拦截给 CommandRouter（推荐用前缀降低歧义）：
    - `/menu` → ShowMenu
    - `/dismiss` → DismissAgent（并停止 run 或仅退出 agent 视产品定义）
    - `/pause` `/stop` `/resume` → Run 控制

> 备注：BMAD 的 agent 文件里常用 `*menu/*dismiss` 作为 cmd；你也可以把 `*` 前缀视为全局命令前缀，与 `/` 等价。
