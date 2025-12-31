# CrewAgent Runtime — 主体架构设计（Project-First / v1.1 micro-file+graph / Agent Menu 驱动）

> 本文是 `_bmad-output/tech-spec/runtime-spec.md` 的“工程化展开版”，吸收 `_bmad-output/analysis/research/bmad-runtime-report/README.md` 的结论，并结合本次讨论的产品约束：
>
> 1) 默认产物写入 **Project 目录**（用户可见）。  
> 2) `.bmad` 包、运行状态、日志等 **全部进入 Runtime 私有目录**（用户不可见，不污染项目）。  
> 3) MVP 仅支持 **v1.1 micro-file + graph**。  
> 4) 入口既可从 Agent，也可从 Workflow；从 Agent 时由 `menu + 用户输入` 驱动执行。  
> 5) persona：节点有 `agentId` 用节点 persona；节点没有则继承当前流程/会话 persona（`activeAgentId`）。

---

## 1. 目标与边界

### 1.1 产品目标（MVP）

- Electron 本地客户端，可导入 `.bmad` 包并在用户 Project 上执行
- 执行方式对齐 Cursor/BMAD：**工具硬实现 + 流程由 LLM 推进 + ToolCalls 优先**
- workflow 可暂停/恢复：恢复只依赖 state（frontmatter）+ graph + 日志（可选）
- Agent Menu：像 BMAD agent 一样“展示菜单→等待用户输入→执行菜单项”
- 开发联调：优先跑通示例包 `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/`（对照 `_bmad-output/implementation-artifacts/runtime/dev-guide-create-story-micro.md` 与 `_bmad-output/implementation-artifacts/runtime/create-story-micro-ideal-trace.md`）

### 1.2 非目标（MVP 不做）

- 经典 BMAD：`workflow.yaml + workflow.xml`（保留扩展点，遇到需明确报错）
- 复杂外部工具（先不接 MCP，或先占位禁用）
- 分布式协作/云同步（先单机）

### 1.3 开发必读（必须）

- E2E golden path（如何联调/验收）：`_bmad-output/implementation-artifacts/runtime/dev-guide-create-story-micro.md`
- 理想交互 Trace（对照实现 ExecutionEngine）：`_bmad-output/implementation-artifacts/runtime/create-story-micro-ideal-trace.md`
- LLM 对话协议（OpenAI-compatible / ToolCalls）：`_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- 示例包本体（入口/graph/steps/assets）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/`

---

## 2. 核心原则（为什么这样拆）

- **Project-First**：产物默认写到 Project（可被用户看到、提交 git、复用）。
- **Private RuntimeStore**：包/状态/日志不落项目，避免污染与泄露；支持“一项目多 run”。
- **Graph as Source of Truth**：`workflow.graph.json` 是可校验的跳转真源（Builder/Runtime 共用）。
- **Document-as-State**：`workflow.md` frontmatter 是最小可恢复状态（但存放在 RuntimeStore）。
- **LLM Decides, Runtime Enforces**：分支由 LLM 决策；Runtime 负责沙箱、校验、原子写与审计。

---

## 3. 关键概念模型

### 3.1 Project / Package / Run

- **Project**
  - `projectRoot`：用户工程目录（真实路径，仅 Runtime 内部可见）
  - `projectId`：可用 `sha256(realpath(projectRoot))` 或随机 uuid + 映射表
  - `artifactsRoot`：默认 `projectRoot/artifacts/`（可配置）

- **Package（.bmad）**
  - `.bmad` zip 被解压到 RuntimeStore 的 packages 缓存（只读）
  - 通过 `bmad.json` 定位 entry 或 workflows 索引

- **Run**
  - `runId`：uuid
  - 绑定：`(projectId, packageId, workflowRef, activeAgentId)`
  - 状态文件：`@state/workflow.md`（真实存储在 RuntimeStore，不在项目）
  - 日志文件：`@state/logs/execution.jsonl`

### 3.2 存储布局（强约束）

**Project（用户可见）**

```text
<ProjectRoot>/
  artifacts/                     # 默认产物（用户可见）
  ...                            # 用户代码与文件
```

**RuntimeStore（用户不可见）**

```text
<RuntimeStoreRoot>/
  packages/
    <packageId>/                 # 解压后的 .bmad 包缓存（只读）
  projects/
    <projectId>/
      runs/
        <runId>/
          state/
            workflow.md          # Document-as-State（frontmatter）
            workflow.graph.json  # 可缓存（或只读引用 package）
            agents.json          # 可缓存（或只读引用 package）
            logs/
              execution.jsonl
          cache/
            ...                  # 可选：LLM 摘要/索引
```

> 说明：RuntimeStoreRoot 推荐用 Electron `app.getPath('userData')`，从而天然“对用户不可见/不进项目”。

---

## 4. “路径虚拟化”与沙箱（让 LLM 既能改 Project，又不看到私有目录）

### 4.1 Mount Alias（对 LLM 暴露的唯一路径视图）

LLM 工具只允许看到三类根：

- `@project/...` → ProjectRoot（读写）
- `@pkg/...` → 当前 package 解压根（只读）
- `@state/...` → 当前 run 的 state 根（读写，但用户不可见）

### 4.2 默认路径规则（减少 LLM 犯错）

- step/graph 中出现的 `artifacts/...`（相对路径）→ 解析为 `@project/artifacts/...`
- step 中提到 `workflow.md`（状态）→ 实际路径固定为 `@state/workflow.md`
- package 内的相对路径（如 `steps/*.md`）→ 解析为 `@pkg/steps/*.md`

### 4.3 ToolHost 强制校验

- 任何 `fs.*` 都必须带 mount（或 Runtime 自动补全默认 mount）
- `@pkg` 只读：拒绝写
- `@state` 写入必须 YAML 可解析（命中 workflow.md 时）+ 原子写入
- `@project` 写入允许，但必须 `realpath` 前缀校验，防 `..` / symlink escape

---

## 5. 入口模型：从 Agent 或从 Workflow

入口细化（UI/状态机/消息序列）：见 `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`

### 5.1 从 Agent 开始（你指定的主入口）

关键点：**不是先选流程**；而是进入 Agent Session 后，靠 `menu + 用户输入` 触发工作。

- UI：显示当前 agent persona 与 menu（可渲染）
- Router：对用户输入做解析（见契约文档）
- 执行：
  - `menuItem.workflow` → 启动 WorkflowRun
  - `menuItem.exec` → 执行 ScriptRun（或识别为 workflow.md 则转 WorkflowRun）
  - `menuItem.action` → 内建动作或 prompt 执行

### 5.2 从 Workflow 开始（次入口）

- UI 选择一个 workflow（来自 package entry 或 workflows[]）
- 若用户未选 agent，则要求选一个作为 `activeAgentId`（流程 persona）

### 5.3 Persona 继承规则（硬约束）

每次执行 node 时：

- `effectiveAgentId = node.agentId ?? run.activeAgentId`
- PromptComposer 用 `effectiveAgentId` 生成 persona/systemPrompt

---

## 6. 执行引擎：micro-file + graph（MVP 主干）

### 6.1 核心循环（ToolCalls 优先）

1. 读取 `@state/workflow.md` frontmatter（currentNodeId/stepsCompleted/variables/decisionLog/artifacts）
2. 读取 `@pkg/workflow.graph.json`，拿到 `currentNodeId` 的出边
3. 读取当前 node 的 step 文件 `@pkg/<node.file>`
4. LLM 产出：
   - 对用户提问（若需要）
   - 写入 `@project/artifacts/...` 或修改 `@project/...` 代码
   - 用 `fs.apply_patch` 更新 `@state/workflow.md` frontmatter（完成当前 node、更新 currentNodeId 等）
5. Runtime 校验跳转合法性（graph 后继）后，继续下一轮

### 6.2 状态字段（对齐 spec）

`@state/workflow.md` frontmatter 至少包含（见 schema）：
- `schemaVersion` / `workflowType`
- `currentNodeId`
- `stepsCompleted: string[]`
- `variables: object`
- `decisionLog: Array<{from,to,label,reason?,decidedAt?}>`
- `artifacts: string[]`（这里记录的是 `@project/...` 的相对路径，例如 `artifacts/intake.md`）

> 约定：`artifacts` 记录 **Project 内相对路径**，便于用户直接定位与提交。

---

## 7. 组件拆分（Electron：Main / Renderer）

### 7.1 Main Process（能力与安全边界）

- `ProjectManager`
  - 打开/切换 projectRoot
  - 生成/读取 projectId 映射
  - 确保 `projectRoot/artifacts/` 存在（默认）

- `RuntimeStore`
  - 计算 `RuntimeStoreRoot`
  - 提供 packages / projects / runs 的目录 API

- `PackageManager`
  - 导入 `.bmad` zip → 解压到 `RuntimeStoreRoot/packages/<packageId>`
  - JSON Schema 校验（bmad/agents/graph/frontmatter/step）
  - 输出 `PackageIndex`（workflows 列表、agents 列表、graph 索引）

- `RunManager`
  - `createRun(projectId, packageId, workflowRef, activeAgentId)`
  - 初始化 `@state/workflow.md`（从 package workflow.md 拷贝 + 填 runId/activeAgentId 等）
  - 提供 `resumeRun(runId)` / `listRuns(projectId)`

- `AgentRegistry`
  - 加载 package 的 `agents.json`
  - 提供 `getAgent(id)` / `listAgents()`

- `CommandRouter`（Agent Menu 解析）
  - 输入：用户文本 + agent.menu
  - 输出：ResolvedCommand（详见契约文档）

- `PromptComposer`
  - `systemBase`：通用运行规则 + mount 说明
  - `systemPolicy`：tool policy（fs/mcp 限额等）
  - `systemPersona`：agent persona 或 systemPrompt
  - `userLauncher`：要求按 graph 执行、写 artifacts 到 project、更新 state
  - 示例与拼装方式：见 `_bmad-output/tech-spec/prompt-composer-examples.md`

- `LLMAdapter`
  - Provider 插件（OpenAI ToolCalls 优先）
  - 流式事件：token delta / toolcall delta

- `ToolHost`
  - Builtin tools：`fs.read/list/search/apply_patch/write`
  - 沙箱/限额/原子写/YAML 校验/graph 跳转校验
  - （预留）`McpToolRegistry`

- `ExecutionEngine`
  - 维护对话历史
  - 执行 toolcall loop（LLM ↔ ToolHost）
  - 产生 UI 事件（assistant token、tool executed、state updated）

### 7.2 Renderer（体验与可视化）

- Start（无 Project 时）：New/Open Project + Recent Projects
- Settings（固定底部）：Package 信息/导入/缓存、LLM Provider、Theme
- Project Context：
  - Files（ProjectRoot 树形浏览 + Markdown 预览，viewer 可扩展）
  - Works（Conversation 历史 + 新建入口：Agent / Workflow / Chat）
  - Conversation 内：类型切换、workflow 启动数量与继续执行入口
- Run Workspace：
  - Chat（流式）
  - ToolCalls 面板（读写/patch 展示）
  - Artifacts 面板（扫描 `projectRoot/artifacts/` + 展示 `state.frontmatter.artifacts`）
  - Run 列表与 Resume

---

## 8. 安全、可靠性、可恢复性（必须从 MVP 做）

### 8.1 文件与路径安全

- mount alias → realpath → 前缀校验（防 `..` / symlink escape）
- `@pkg` 只读
- 单次读写大小限制（由 agent.tools.fs.maxReadBytes/maxWriteBytes + 全局默认合并）

### 8.2 状态写入一致性

当写 `@state/workflow.md` 时：
- 必须 frontmatter YAML 可解析
- 必须符合 `workflow-frontmatter.schema.json`
- `currentNodeId` 的变更必须满足 graph 允许的后继（`currentNodeId_old -> currentNodeId_new`）
- 原子写：tmp→rename

### 8.3 可恢复

- 重新打开项目时：
  - 从 RuntimeStore 的 `runs/` 列表中读取最近 run
  - 读取 `@state/workflow.md` 得到 `currentNodeId` 与进度即可恢复
  - 日志用于审计与 UI 回放（非强依赖）

---

## 9. 扩展点（为后续 Epic 预埋）

- **ClassicWorkflowRunner**（未来）：支持 `workflow.yaml + workflow.xml`（单独 runner，不污染 v1.1 引擎）
- **Strict State Mode**（可选）：提供 `runtime.complete_node()` 来收敛状态更新，减少 LLM 直接 patch frontmatter 的错误
- **MCP**：ToolRegistry = Builtin + MCP（统一暴露 tools 给 LLM）
- **Subworkflow**（v1.2+）：利用 `callStack` 执行嵌套流程（见 runtime-spec.md 的建议）
- **审批/权限**：对 `@project` 的写入可引入策略（例如只允许写 artifacts/ 与特定目录）
- **索引/检索**：对 Project 与 artifacts 建立索引，提供 `search` / `semantic_retrieve` 工具
