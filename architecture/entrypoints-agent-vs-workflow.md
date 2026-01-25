# CrewAgent Runtime — 两种入口：Workflow-First vs Agent-First（v1.1 micro-file+graph）

> 目标：把 Runtime 的两类启动方式写清楚（UI/状态机/消息序列/停点判断），为后续 Epic & Story 拆分提供依据。
>
> 相关文档：
> - 架构总览：`_bmad-output/architecture/runtime-architecture.md`
> - Agent 菜单契约：`_bmad-output/tech-spec/agent-menu-command-contract.md`
> - Prompt 组装示例：`_bmad-output/tech-spec/prompt-composer-examples.md`
> - v1.1 examples：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/`

---

## 1. 概念对齐（只保留关键）

- **ProjectRoot**：用户工程目录（用户可见）；默认产物写入 `ProjectRoot/artifacts/`。
- **RuntimeStoreRoot**：应用私有目录（用户不可见）；保存 run state、logs、包缓存。
- **Mounts（对 LLM 暴露）**：
  - `@project`：ProjectRoot（读写）
  - `@pkg`：当前 `.bmad` 包内容（只读）
  - `@state`：当前 run 状态（读写但用户不可见），核心文件 `@state/workflow.md`
- **Run**：一次 workflow 执行实例（可暂停/恢复）。
- **Autopilot**：默认自动推进 workflow；只有“缺必要输入”时才停下来问用户。

---

## 2. 两种入口的差异（你后续拆 Epic 时可用）

| 入口 | 用户在 UI 的第一步 | 命令解析 | persona 起点 | 典型用途 |
|---|---|---|---|---|
| Workflow-First | 先选 workflow（再选/默认 agent） | 不需要 menu 匹配 | `activeAgentId`（流程默认 persona） | “我就要跑这个流程” |
| Agent-First | 先选 agent（看到 menu） | 必须走 CommandRouter（数字/trigger/模糊/澄清） | agent persona（会话 persona） | “像 BMAD 一样从角色开始工作” |

> 两者最终都会落到同一套 `RunManager + ExecutionEngine`，区别主要在“如何选定 workflowRef / activeAgentId / dataContext”。

---

## 3. 统一的 Run 状态机（UI & 引擎共同遵守）

> 核心：**一次用户输入**会触发 ExecutionEngine 内部的多轮 `LLM → toolCalls → LLM`；UI 只在“停点”显示交互。

### 3.0 Session 状态机（入口层：AgentMode → RunMode）

> 这一层回答“从 Agent 或 Workflow 启动后，UI/引擎整体在什么状态里”。`RunMode.*` 的细节由 3.1/3.3 描述。

```
SessionMode:
  AgentMode
    ├─(CommandRouter 命中 StartWorkflow)→ RunMode.Running
    ├─(Workflow-First 点击 Run)→ RunMode.Running
    └─(Dismiss)→ Idle

  RunMode
    ├─Running
    ├─WaitingUser
    ├─Completed
    └─Failed
```

> 备注：UI 还支持一个 **ChatMode**（ConversationType=`chat`）用于“随便聊”：
> - 不绑定 workflow/run，不注入 agent persona（保持通用聊天体验）
> - 但仍可使用 tools（默认开启；由 SystemToolPolicy（Settings）+ agent.tools 合并出的 ToolPolicy 控制可见性与限额，比如允许 `fs.*` 读写 @project/@state）
> - PromptComposer：`mode=chat`（BaseRules(conversation) + ToolPolicy + USER_INPUT）
> - `USER_INPUT` 在 chat 模式下不需要 `forNodeId`

```
RunPhase:
  Running
    ├─(LLM 返回 toolCalls)→ 执行工具 → 继续 Running
    ├─(LLM 无 toolCalls 且未完成)→ WaitingUser
    └─(完成条件满足)→ Completed
  WaitingUser
    └─(用户输入 USER_INPUT)→ Running
  Completed
    └─(用户输入 USER_INPUT)→ Running（Post-Completion Profile；无 active step 注入）
  Failed
```

### 3.1 UI 如何判断“停点”

- 当 LLM 响应里 **存在 toolCalls**：UI 不停（继续内部循环）。
- 当 LLM 响应里 **没有 toolCalls**：这是“用户可见停点”，UI 必须判断：
  - `isWorkflowComplete(@state/workflow.md, graph) == true` → Completed（显示“流程结束”标签；输入框仍可用，可继续对话）
  - 否则 → WaitingUser（显示 LLM 的问题/提示，打开输入框）

### 3.2 UI 如何更新进度（无需 LLM 维护“任务面板”）

- ToolHost 执行 `fs.apply_patch` 写入 `@state/workflow.md` 后：
  - Main 进程解析 frontmatter → 推送 `currentNodeId/stepsCompleted/artifacts/variables` 给 UI
  - UI 用 `workflow.graph.json` 高亮当前节点、展示进度条、展示 artifacts 列表

### 3.3 参考伪代码（更贴近实现）

> 说明：这是“输入路由 + run 内部 pump loop”的最小实现骨架。  
> 关键语义：**有 toolCalls 就继续 loop；无 toolCalls 就进入 UI 停点（WaitingUser 或 Completed）**。

```ts
enum SessionMode {
  Agent = 'agent',
  Run = 'run',
}

enum RunPhase {
  Running = 'Running',
  WaitingUser = 'WaitingUser',
  Completed = 'Completed',
  Failed = 'Failed',
}

async function startRunFromWorkflowFirst(selection: { workflowRef: unknown; activeAgentId: string }) {
  const run = RunManager.createRun({
    projectRoot: Project.root,
    packageId: Package.current.id,
    workflowRef: selection.workflowRef,
    activeAgentId: selection.activeAgentId,
  })
  Session.setMode(SessionMode.Run, run.id)
  await pump(run, { intent: 'start' })
}

async function startRunFromAgentMenu(cmd: { kind: 'StartWorkflow'; workflowRef: unknown; dataRef?: unknown }) {
  const run = RunManager.createRun({
    projectRoot: Project.root,
    packageId: Package.current.id,
    workflowRef: cmd.workflowRef,
    activeAgentId: Session.activeAgentId, // 来自 Agent Session
    dataRef: cmd.dataRef,
  })
  Session.setMode(SessionMode.Run, run.id)
  await pump(run, { intent: 'start' })
}

async function onUserText(rawText: string) {
  if (Session.mode === SessionMode.Agent) {
    const cmd = CommandRouter.resolve(rawText, CurrentAgent.menu, { targetSurface: 'electron' })
    if (cmd.kind === 'StartWorkflow') return startRunFromAgentMenu(cmd)
    if (cmd.kind === 'ShowMenu') return UI.showMenu(CurrentAgent.menu)
    if (cmd.kind === 'DismissAgent') return Session.dismissAgent()
    return chatAsAgent(rawText)
  }

  // RunMode
  if (isGlobalCommand(rawText)) return handleGlobalCommand(rawText) // /pause /stop /resume /menu ...

  const run = RunManager.get(Session.runId)
  const completed = isWorkflowComplete(run.state, run.graph)
  run.pendingUserInputs.push({
    type: 'USER_INPUT',
    ...(completed ? {} : { forNodeId: run.state.currentNodeId }),
    raw: rawText,
  })
  await pump(run, { intent: 'continue' })
}

async function pump(run: Run, directive: { intent: 'start' | 'continue' | 'resume' }) {
  run.phase = RunPhase.Running
  UI.setRunPhase(run.id, RunPhase.Running)

  const maxIterations = 50
  for (let iter = 0; iter < maxIterations; iter++) {
    const messages = PromptComposer.compose({
      run,
      directive,
      // 建议：只在 start/resume/用户刚回复/currentNodeId 或 effectiveAgentId 变化时
      // 注入 RUN_DIRECTIVE；workflow 未完成时才注入 NodeBrief（completed 后进入 Post-Completion Profile：无 active step 注入）。
    })

    const response = await LLM.call({
      messages,
      tools: ToolRegistry.visibleFor(run.effectiveAgentId),
    })

    run.history.appendAssistant(response)
    UI.streamOrAppendAssistant(response) // 流式或一次性显示

    if (response.toolCalls?.length) {
      for (const call of response.toolCalls) {
        const result = await ToolHost.execute(call)
        run.history.appendToolResult(call.id, result)

        if (call.name === 'fs.apply_patch' && call.args?.path === '@state/workflow.md') {
          run.state = StateStore.read(run) // 解析 frontmatter
          UI.updateWorkflowProgress(run.id, run.state, run.graph)
        }
      }

      if (isWorkflowComplete(run.state, run.graph)) {
        run.phase = RunPhase.Completed
        UI.setRunPhase(run.id, RunPhase.Completed)
        // 注：Completed 只是 workflow 状态标签；输入框仍可用，用户可继续对话（Post-Completion Profile）。
        return
      }

      continue // 继续 loop
    }

    // 无 toolCalls：进入“用户可见停点”
    if (isWorkflowComplete(run.state, run.graph)) {
      run.phase = RunPhase.Completed
      UI.setRunPhase(run.id, RunPhase.Completed)
    } else {
      run.phase = RunPhase.WaitingUser
      UI.setRunPhase(run.id, RunPhase.WaitingUser)
      UI.focusInput()
    }
    return
  }

  run.phase = RunPhase.Failed
  UI.setRunPhase(run.id, RunPhase.Failed)
  UI.showError('LLM exceeded max iterations')
}

function isWorkflowComplete(state: WorkflowState, graph: WorkflowGraph): boolean {
  if (state.variables?.workflowStatus === 'complete') return true
  const node = graph.nodeById(state.currentNodeId)
  return node?.type === 'end' && state.stepsCompleted.includes(node.id)
}
```

---

## 4. Workflow-First（直接从 workflow 开始）

### 4.1 UI 流程

1) 选择 ProjectRoot  
2) 导入/选择 `.bmad` 包（得到 `@pkg/...`）  
3) 选择 workflow（`workflowRef`）  
4) 选择 `activeAgentId`（若 workflow/node 未指定 persona 或你希望用户显式指定默认 persona）  
5) 点击 Run → 创建 run（初始化 `@state/workflow.md`）→ 进入 ExecutionEngine

### 4.2 首次消息（RUN_DIRECTIVE start）

> 建议统一用一个“可机器解析的壳”，而不是自然语言 `Start workflow:`。

```text
RUN_DIRECTIVE
- runType: bmad-micro
- intent: start
- workflow: <workflowId or workflow.md path>
- state: @state/workflow.md
- graph: @pkg/<...>/workflow.graph.json
- artifactsRoot: @project/artifacts/
- currentNodeId: <from @state/workflow.md>
- effectiveAgentId: <node.agentId ?? activeAgentId>
- autopilot: true
```

### 4.3 运行中的循环（自动跑 / 何时停）

- ExecutionEngine 在 Running 内部循环调用 LLM：直到
  - 需要用户输入（step 明确要求 ask/WAIT）→ WaitingUser
  - `@state/workflow.md` 显示 end 完成 → Completed

### 4.4 用户输入（USER_INPUT）

当 workflow 未完成且处于 WaitingUser 时，用户输入必须被包装并带上 `forNodeId`：

```text
USER_INPUT
- forNodeId: <currentNodeId>
<raw user text>
```

> 这样可避免 resume/跨天时把回答误用到错误节点。

当 workflow 已 completed（Post-Completion Profile）时：不再有 active step，因此不绑定节点（省略 `forNodeId`）：

```text
USER_INPUT
<raw user text>
```

### 4.5 example：create-story-micro（Workflow-First）

示例包：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/`

- workflow：`@pkg/workflow.md`
- graph：`@pkg/workflow.graph.json`
- entryNode：`step-01-select-story`

典型停点：若项目内找不到 `sprint-status.yaml` 且用户未指定 storyKey，`step-01` 会提问并 WAIT；用户输入 storyKey 后继续自动跑到 end。

---

## 5. Agent-First（先选 Agent，用 menu + 用户输入驱动）

### 5.1 UI 流程

1) 选择 ProjectRoot  
2) 导入/选择 `.bmad` 包（得到 agents/workflows）  
3) 选择 Agent（进入 Agent Session，展示 menu）  
4) 用户输入（数字/trigger/模糊）→ CommandRouter → 执行命令

### 5.2 CommandRouter 的职责（必须确定性）

见：`_bmad-output/tech-spec/agent-menu-command-contract.md`

- 数字：选第 N 项
- trigger 精确命中
- 模糊匹配：多命中则澄清
- 未命中：回退到 Chat（agent persona 对话）

### 5.3 从 Agent-First 进入 Run（StartWorkflow）

当 menuItem 命中 `workflow` 或 `exec(workflow.md)`：

1) `workflowRef` 解析（workflowId 或 workflow.md path）  
2) 创建 run（初始化 `@state/workflow.md`）  
3) 进入 ExecutionEngine（与 Workflow-First 相同）

> 也就是说：Agent-First 只是“如何选择 workflowRef + dataContext + activeAgentId”的入口，真正的执行与停点逻辑与 Workflow-First 完全一致。

### 5.4 RunMode 下输入如何路由（避免“聊天误改项目 / 误触菜单”）

强烈建议把输入分成两种 mode（来自契约文档的建议）：

- **AgentMode（无 activeRunId）**：输入优先走 CommandRouter；未命中 → Chat
- **RunMode（存在 activeRunId）**：输入默认当作 `USER_INPUT` 交给 ExecutionEngine  
  - 只有显式前缀命令才拦截为全局控制（建议 `/` 或 `*`）：
    - `/menu`：显示菜单
    - `/pause` `/stop` `/resume`：控制 run
    - `/dismiss`：退出 agent（是否连带 stop run 取决于产品定义）

### 5.5 example：create-story-micro（Agent-First）

`create-story-micro/agents.json` 里提供了一个最小 menu：

- trigger: `create-story`
- workflow: `workflow.md`

完整序列（高层）：

1) 用户在 Agent Session 输入：`create-story`
2) Router → `StartWorkflow(workflow.md)`
3) Runtime 发 `RUN_DIRECTIVE intent:start`，进入 autopilot loop
4) 若 step-01 需要 storyKey：WaitingUser
5) 用户输入 storyKey → `USER_INPUT forNodeId: step-01-select-story`
6) 引擎继续 Running 直到 end-99 完成

---

## 6. 两种入口如何共用 PromptComposer（落地建议）

把 PromptComposer 设计成“同一套 API，不同入口只改输入参数”：

- 入口差异只体现在：
  - `activeAgentId`（Workflow-First 由用户/默认决定；Agent-First 来自会话 agent）
  - `workflowRef`（Workflow-First 由 UI 选择；Agent-First 由 menu 解析）
  - `dataContext`（Agent-First 可能来自 menuItem.data）
- 进入 Run 后：每轮只要 `currentNodeId` 或 `effectiveAgentId` 变化，就重新注入 `RUN_DIRECTIVE + NodeBrief` 锚定；其余轮次只追加 tool 结果即可。

示例与模板：`_bmad-output/tech-spec/prompt-composer-examples.md`

---

## 7. MVP 验收点（写 Story 时直接用）

- 两种入口都能启动同一个 micro workflow，并在同一套 Run 状态机内运行。
- `autopilot=true` 时，除非缺必要输入，否则不会在中间停下来要求用户确认。
- UI 停点判断仅依赖：是否存在 toolCalls + `@state/workflow.md` 是否完成。
- persona 规则一致：`effectiveAgentId = node.agentId ?? activeAgentId`。
