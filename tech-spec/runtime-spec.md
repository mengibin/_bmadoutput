# CrewAgent Runtime 设计细化（Cursor 风格：工具硬实现，流程由 LLM 驱动；ToolCalls 优先，MCP 预留）

> 目标：做一个类似 Cursor 的本地运行客户端（Electron），可加载 `.bmad` 包，按 `workflow.md` 的 Frontmatter 作为状态（Document-as-State）执行 `steps/*.md`，并通过 ToolCalls 完成文件类操作；后续再接入严格 MCP 驱动外部重工具。

> `.bmad` 规范包见：`crewagent-runtime/spec/bmad-package-spec/v1.1/`（含 schema、模板与示例包）。

> 与 `_bmad-output/architecture/runtime-architecture.md` 一致：Runtime 采用 **Project-First + Private RuntimeStore**：
> - 用户可见产物默认写入 Project（例如 `@project/artifacts/`）
> - `.bmad` 解包缓存、run state（`@state/workflow.md`）与日志存放在 RuntimeStore（用户不可见、不污染项目）

## 1. `.bmad` 包（v1.1）约定

### 1.1 ZIP 结构（最小可跑）

```text
{name}.bmad  (zip)
  bmad.json                 # 包清单/入口（必选）
  workflow.graph.json       # 图结构（必选，权威真源）
  workflow.md               # 入口文档 + Frontmatter 状态（必选）
  steps/                    # micro-file steps（必选）
    step-01-*.md
    decide-*.md
    end-*.md
  agents.json               # Agent persona + prompts + tool policy（必选）
  assets/                   # 可选：图/模板/静态文件

  # 可选：多 workflow（一个包多个流程）
  workflows/<workflow-id>/
    workflow.graph.json
    workflow.md
    steps/
      ...
```

### 1.2 Step 来源与顺序（micro-file + graph）

- **来源**：节点文件在 `steps/*.md`（每个 node 一个文件，step/decision/merge/end 都可建模成 node）。
- **权威顺序/跳转**：以 `workflow.graph.json` 为准（Builder 渲染画布；Runtime 校验 `currentNodeId` 的合法跳转）。
- **索引（给 LLM 快速打开）**：`workflow.md` 正文建议包含 steps 索引链接，但它不是“真源”，只是可读索引。

> 备注：BMAD-METHOD 的 micro-file workflow 通常在 `workflow.md` 里写 “Load and execute steps/step-01…”；CrewAgent Builder 会把它“程序化”为 `workflow.graph.json`，Runtime 只需按图校验跳转 + 提供工具。

### 1.3 `workflow.md` Frontmatter（Document-as-State）

必须字段（MVP/分支友好）：
- `schemaVersion: "1.1"`（与包 spec 对齐）
- `workflowType: string`
- `currentNodeId: string`
- `stepsCompleted: string[]`（已完成 nodeId 集合）
- `variables: object`（分支变量，默认 `{}`）
- `decisionLog: Array<{from,to,label,reason?,decidedAt?}>`（默认 `[]`）

建议字段（增强可恢复与可审计）：
- `runId: string`（uuid）
- `project_name`, `user_name`, `date`
- `inputDocuments: string[]`（上下文注入入口）
- `artifacts: string[]`（runtime 记录输出物路径）
- `updatedAt: string`（ISO datetime）

## 2. Runtime 组件拆分（Electron）

### 2.1 Main Process（“能力与安全边界”在这里）

- `ProjectManager`：打开/切换 ProjectRoot；确保默认产物目录（如 `ProjectRoot/artifacts/`）存在。
- `RuntimeStore`：应用私有目录根（推荐 Electron `app.getPath('userData')`）；保存包缓存、run state、logs（用户不可见）。
- `PackageManager`：导入 `.bmad`（zip 解压到 RuntimeStore/packages/）、JSON Schema 校验、生成包清单（`bmad.json`）；如 `bmad.json.workflows[]` 存在，则在 UI 提供“选择要运行的 workflow”（默认使用 `entry` 指定的入口）。
- `RunManager`：创建/恢复 Run（一次执行实例），在 RuntimeStore/projects/<projectId>/runs/<runId>/ 下初始化 `@state/workflow.md` 与日志目录。
- `WorkflowStateStore`：frontmatter 读写（解析/校验/原子写 tmp→rename）。
- `GraphStore`：加载 `workflow.graph.json`，提供 node/edge 查询与跳转合法性校验（`currentNodeId -> nextNodeId`）。
- `StepIndex`：为 UI/LLM 生成“可读索引”（可从 graph + steps 生成；也可回填到 `workflow.md` 正文）。
- `AgentRegistry`：加载 `agents.json`，提供 persona/prompt 模板。
- `PromptComposer`：拼 system/user prompt（包含 tool 规范 + 沙箱规则）。
- `LLMAdapter`：先实现 OpenAI ToolCalls；本地模型后续兼容（可先不做）。
- `ToolHost`：内置 ToolCalls（文件/patch/搜索/日志写入），并在写入时执行沙箱与校验（frontmatter YAML 可解析；可选：状态跳转必须符合 graph）。
- `ExecutionEngine`：对话编排器（LLM ↔ ToolCalls）。**不替 LLM 决策走哪条分支**；Runtime 只做“硬边界”（工具/沙箱/校验/日志）与“合法性校验”（next node 必须是图允许的后继）。
- `LogStore`：JSONL 追加写（执行记录/工具调用/错误），供 UI 展示。

### 2.2 Renderer（“体验与可视化”）

- Start（无 Project 时）：New/Open Project + Recent Projects
- Settings（固定在底部）：Package 信息/导入/缓存、LLM Provider、Theme、SystemToolPolicy（系统级 tools 开关与限额）
- Project Context：
  - Files：项目树形浏览（**侧边滑出面板**）+ 文件预览入口
  - Works：Conversation 历史 + 新建入口（**侧边滑出面板**）
  - Workspace（右侧持久区）：多 Tab（固定 Conversation/Chat + 多文件 Tab，支持查看/编辑/保存）
  - Conversation 内可切换类型，并显示 workflow 启动数量与继续执行入口
- Run Workspace：
  - Chat 流式输出
  - ToolCalls 面板（读写/patch 展示）
  - Artifacts 面板（生成文件列表）
  - Logs/Progress（基于 `@state/workflow.md` + `workflow.graph.json`）

## 3. 执行模型：ToolCalls 优先（MVP 主干）

### 3.1 Cursor 兼容执行循环（推荐默认）

1) Runtime 打开 run：对 LLM 暴露三类 mount alias（避免泄露真实路径）：
   - `@project`：用户工程目录（读写）
   - `@pkg`：当前 `.bmad` 包内容（只读）
   - `@state`：当前 run 状态（读写但用户不可见，核心文件 `@state/workflow.md`）
2) Runtime 发送初始指令（system 级约束 + tool policy；默认全开，可由 Settings 的 SystemToolPolicy 配置），要求 LLM：
   - 读取 `workflow.md` frontmatter 的 `currentNodeId/stepsCompleted/variables`
   - 根据 `workflow.graph.json` 的出边决定 next node，并读取对应 `steps/<nodeId>.md`
   - 完成本步产出后，**用普通文件工具**更新 `workflow.md` frontmatter（追加 `stepsCompleted`、更新 `currentNodeId/variables/decisionLog/updatedAt/artifacts`）
3) 进入 `chatLoop`：
   - LLM 输出（可能包含 toolCalls）
   - ToolHost 执行 toolCalls → 把结果回给 LLM
4) 结束条件（由 LLM/用户决定）：
   - LLM 判断 workflow 完成并明确告知；或
   - 用户点击 Pause/Stop
5) UI 通过监听/重读 `workflow.md` frontmatter 展示进度（current node 高亮、stepsCompleted 进度、artifacts 列表等）。

### 3.2（可选）更强一致性：专用“状态更新”工具（严格模式）

Cursor/Claude Code 的 BMAD 集成方式是“规则/命令文件 + 通用文件工具”。从 `BMAD-METHOD` 的安装器与生成模板看，不存在 `runtime.complete_step` 这类专用状态 API；LLM 更新 frontmatter 也是通过普通文件编辑/patch 完成的。

因此 CrewAgent 的默认模式应与其一致：**LLM 直接用 `fs.apply_patch/fs.write` 更新 `workflow.md`**；Runtime 负责：
- 原子落盘（tmp→rename）
- frontmatter YAML 可解析校验
- `currentNodeId` 跳转合法性校验（必须是图允许的后继）

如果你希望更强一致性（例如禁止 LLM 直接改状态字段、统一追加 decisionLog、避免并发写），可以额外提供：
- `runtime.update_state({ patch })` 或 `runtime.complete_node({ nodeId, artifacts?, variables?, decision? })`（Runtime 负责读-改-写与校验）

并提供“最少但够用”的文件工具：
- `fs.read({ path })`
- `fs.write({ path, content, mode: "overwrite"|"append" })`
- `fs.apply_patch({ patches: [...] })`（推荐，便于 diff 与回滚）
- `fs.list({ path })`
- `fs.search({ query, globs? })`

> MVP 先把这些做扎实：沙箱 + 限额 + 原子落盘 + YAML 校验 + 日志。专用状态工具可以作为“严格模式”开关，不影响 Cursor 兼容性。

## 4. 安全与可靠性（必须从一开始做）

### 4.1 文件沙箱（NFR-SEC-02）

- 所有 fs 工具只能访问 mount roots：`@project/@pkg/@state`
  - Runtime 负责把 mount alias 映射到真实路径，并在 `realpath` 后做前缀校验，防 `..` 和 symlink escape
  - `@pkg` 强制只读；任何写入直接拒绝
- 单次读取大小限制（例如 512KB/文件，超过则要求 LLM 指定范围或走摘要）
- 写入默认走 `apply_patch`；`write` 只允许写到 `@project` 或 `@state`
- **建议加一层 frontmatter 校验**：当写入的目标文件包含 YAML frontmatter（尤其是 `workflow.md`）时，Runtime 在落盘前解析 YAML；解析失败则拒绝写入并返回错误给 LLM 修正（这不依赖 Cursor 的扩展 API，而是 ToolHost 的实现细节）。

### 4.2 超时与资源限制（NFR-REL-02）

- ToolCalls 默认超时 5min（可配置）
- stdout/stderr（未来 MCP）要截断与大小上限

### 4.3 可恢复（NFR-REL-01）

- crash 后只需重新加载 run：
  - 从 `@state/workflow.md` 的 `currentNodeId/stepsCompleted/variables` 恢复 next node
  - 从 `@state/logs/execution.jsonl` 恢复审计线索（不要求恢复完整对话）

## 5. MCP（后续接入的“重工具通道”）

你已确认策略：
- **本地文件/轻操作**：先用 ToolCalls（内置 ToolHost）
- **复杂外部工具**：再接入严格 MCP（stdio/jsonrpc）

落地建议：在 ToolHost 下预留一个 `McpToolRegistry`：
- `ToolRegistry = BuiltinTools + McpTools`
- LLM 仍然只看到统一的 tool 列表；后端把调用路由到 Builtin 或 MCP

MVP 可以先把 MCP 做成“禁用/占位”，只做接口不做实现，避免阻塞主干。

## 6. MVP 里程碑（推荐）

1) 导入 `.bmad` → 选择 Project → 创建 run（RuntimeStore）→ 展示 steps 列表
2) 仅实现 `fs.read/fs.write/fs.apply_patch` 的 ToolCalls（含沙箱与 frontmatter 校验）
3) 让 LLM 能跑通 step-01 → 写入一个 artifact → 通过 `fs.apply_patch` 更新 frontmatter
4) 增加 `fs.apply_patch/fs.search`，完善日志与 UI
5) （可选）增加 `runtime.update_state`/`runtime.complete_node` 严格模式
6) 之后再引入 MCP（stdio driver）与审批/权限

## 7. 流程嵌套（Sub-Workflow）支持吗？

支持（建议 v1.2+ 明确约定）。做法是把“调用子流程”建模成一种 **Node 类型**，runtime 用 **调用栈（callStack）** 驱动执行与恢复。

### 7.1 数据结构建议

- 在 `step-xx.md` 的 Frontmatter 增加：
  - `type: "step" | "subworkflow"`（默认 `step`）
  - `subworkflow: "./subflows/foo/workflow.md"`（当 `type=subworkflow`）
  - `passContext: true|false`（是否把父流程 artifacts/inputDocuments 注入子流程）

- 在根 `workflow.md` Frontmatter 增加（用于 pause/resume）：
  - `callStack: Array<{ workflow: string; nodeId: string }>`（保存“当前执行到哪个子流程”）

### 7.2 执行语义（推荐：call/return）

当 runtime 执行到 `type=subworkflow`：
1) 将 `{ workflow: childPath, nodeId: currentNodeId }` push 到 `callStack`
2) 进入子流程：按子流程自己的 `workflow.md` + `steps/` 执行，直到子流程完成
3) pop `callStack`，把父流程该 step 标记完成，并合并子流程产物到父流程 `artifacts`

> 需要做循环依赖检测（防止 A→B→A）与最大嵌套深度限制。
