# CrewAgent Runtime — PromptComposer 组装指南（含 v1.1 examples 示例）

> 本文回答：在 `_bmad-output/architecture/runtime-architecture.md` 和 `_bmad-output/tech-spec/agent-menu-command-contract.md` 中提到的 `PromptComposer`，运行时应该如何把 **用户输入 + `.bmad` 包内容 + Run 状态 + Agent persona + tool policy** 组装成一次 LLM 调用的 prompt/messages。
>
> 示例来自：
> - `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/multi-workflows/`
> - `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/hello-branch/`

---

## 1. PromptComposer 的输入（建议）

```ts
type SessionMode = 'agent' | 'chat' | 'run';

interface ComposeInput {
  mode: SessionMode;

  // 运行环境（真实路径仅 runtime 内部使用，LLM 只能看到 mount alias）
  projectRoot: string;
  runtimeStoreRoot: string;

  // 当前包与 workflow
  packageId: string;
  packageMount: '@pkg';
  workflowRef?: { type: 'workflowId' | 'packagePath'; value: string };

  // 当前会话/流程 persona
  activeAgentId: string;               // session / run 的“默认 persona”
  effectiveAgentId: string;            // node.agentId ?? activeAgentId
  // mode=chat: 不需要把 persona 显式写给 LLM（但仍会发送 ToolPolicy；默认全开，可被 SystemToolPolicy/agent.tools 收紧）

  // run 状态（mode=run 时必填）
  runId?: string;
  stateMount?: '@state';
  stateSummary?: {
    currentNodeId: string;
    stepsCompleted: string[];
    variables: Record<string, unknown>;
    decisionLogCount: number;
    artifacts: string[];
  };

  // 当前节点信息（mode=run 时必填）
  nodeBrief?: {
    nodeId: string;
    nodeType: 'step' | 'decision' | 'merge' | 'end' | 'subworkflow';
    title?: string;
    stepFilePathInPkg: string;          // 来自 graph.nodes[].file
    inputs?: string[];
    outputs?: string[];
    transitions?: Array<{
      to: string;
      label: string;
      isDefault?: boolean;
      conditionText?: string;
    }>;
  };

  // tool policy（默认全开；由 Settings 的 SystemToolPolicy + agents.json(agent.tools) 合并）
  toolPolicy: {
    fs: { enabled: boolean; maxReadBytes: number; maxWriteBytes: number };
    mcp: { enabled: boolean; allowedServers?: string[] };
  };

  // 触发命令或用户输入
  commandName?: string;                 // e.g. "hello-linear"
  userInput?: string;                   // 用户刚输入的文本（run/agent 都可能）

  // 可选：menuItem.data 注入
  dataContext?: {
    label: string;
    path: string;                       // 只暴露为 mount alias 路径（@project/... 或 @pkg/...)
    preview: string;                    // 截断后的预览/摘要（受 maxReadBytes）
  };
}
```

> 要点：`PromptComposer` 的目标不是“把所有文件内容塞给模型”，而是拼装 **规则 + 导航 + 最小摘要**，让模型用 ToolCalls 去按需读取真正的权威文件（workflow.md/graph/step/artifacts）。

---

## 2. Prompt 分层（推荐固定顺序）

你可以把 system/user 拆成多个“片段”，最后合并为 `messages[]`：

1) **System(BaseRuntimeRules)**：固定模板（跨 agent/workflow 通用）
2) **System(ToolPolicy)**：从 Settings 的 SystemToolPolicy + agent.tools 合并出本次允许的工具能力与限额（并声明 `@pkg` 只读；默认全开）
3) **System(Persona)**：从 `agents.json` 渲染 persona（或直接用 `agent.systemPrompt`）
4) **User(Launcher / Continue)**：针对“启动/继续 run”的指令（对齐 micro-file+graph）
5) **User(NodeBrief)**：把当前 node 的关键导航信息写清（nodeId、step file、inputs/outputs、允许的 next、artifacts 默认根）
6) **User(DataContext)**（可选）：把 menuItem.data 的摘要注入（明确“这是额外上下文”）
7) **User(UserInput)**：用户刚刚输入的自然语言（如果是 runMode，视为当前 step 的用户回复）

> 兼容性建议：允许多条 system message；若 provider 不支持，则在发送前合并成一条 system。

---

## 2.1 为什么你看到“Cursor 和 LLM 多次自主交互”

这不是“多次对话”，而是一次用户输入触发的 **ExecutionEngine 内部循环**：

1) Runtime 调 LLM → LLM 返回 toolCalls（读文件/写文件/改任务列表）  
2) Runtime 执行 toolCalls → 把 tool 结果回传给 LLM  
3) Runtime 再调 LLM → 直到 LLM 不再发 toolCalls，并且：
   - 要么明确向用户提问并等待（need user input）
   - 要么 workflow 完成（done）

因此 **PromptComposer 只需要在关键时刻“重新锚定”**（例如 start/resume/node 变化/用户回复），其余轮次可以只追加 tool 结果即可。

---

## 2.2 Tool 读到的文件内容，是否必须回传给 LLM？

需要把“Runtime 自己读”与“LLM 通过 toolCall 读”分开看：

### A) Runtime 内部读（不需要回传）

Runtime 为了 **校验/沙箱/进度 UI** 读取文件（例如：解析 `@state/workflow.md` frontmatter、校验 graph 跳转合法性、展示 artifacts 列表）——这些属于 Runtime 自己的工作，不需要也不应该强制回传给 LLM。

> 例：LLM 对 `@state/workflow.md` 做 `fs.apply_patch` 后，Runtime 重新读取并解析 frontmatter 来更新 UI 进度，这次“读”可以完全不出现在 LLM 的消息里。

### B) LLM 通过 toolCall 发起的读（必须回传结果，但不必是“全文”）

只要是 LLM 发起了 `fs.read/fs.list/fs.search` 等 toolCall，Runtime 就必须回一个 tool result message 给 LLM；否则模型无法继续推理（它会认为工具没有执行或上下文缺失）。

但“回传结果”不等于“必须回传全文内容”。推荐策略是：

- **小文件**：返回完整 `content`
- **大文件/超限**：返回 `contentPreview`（截断）+ `bytes` + `truncated: true` +（可选）`sha256`
- **敏感路径**（如果你未来引入敏感分区）：返回 `omitted: true` + `reason` + 引导模型改用更窄的读取方式

示例（推荐的 fs.read 返回形态）：

```json
{
  "ok": true,
  "path": "@project/artifacts/epics.md",
  "bytes": 120000,
  "truncated": true,
  "contentPreview": "# Epics\\n... (truncated)",
  "sha256": "…"
}
```

> 你在 `create-story-micro-ideal-trace.md` 里看到的 `contentPreview`/`entriesPreview`，就是在表达“现实里通常不会把全文塞回给模型”的这层策略。

### C) 想“完全不把内容给 LLM”，应该怎么做？

如果你希望某些读取/解析完全由 Runtime 负责（例如从 `sprint-status.yaml` 找第一个 backlog storyKey），不建议让 LLM 走 `fs.read` 再自己解析；而是提供更语义化的工具，返回“派生结果”：

- `bmad.findSprintStatus({ root: "@project" }) -> { path: "..." }`
- `bmad.pickBacklogStoryKey({ sprintStatusPath }) -> { storyKey }`

这样 LLM 仍然能继续推进（因为它得到了工具结果），但你不需要把原始文件全文回传给它（token/隐私都更友好）。

---

## 2.3 不回全文时，如何尽量保证 LLM 推理正确性？

先明确：不回全文会降低“模型一次性看到全局”的能力，因此正确性必须通过 **“按需取证 + 硬校验 + 可回退”** 来获得，而不是靠模型“猜”。

推荐组合策略（MVP 可分阶段落地）：

1) **Progressive Disclosure（先定位再精读）**
   - 先 `fs.search`（限定 globs/目录）定位到“匹配位置/章节”
   - 再 `fs.read` 读取“命中附近的小窗口”（建议 fs.read 支持 `startLine/endLine` 或 `offset/length`）
   - 让模型在关键判断前“引用到可验证的片段”，而不是基于 preview 猜全貌

2) **把“大文件”变成“小中间产物”**
   - 例如 create-story 的 Step-03 先把 epics/PRD/architecture/UX 中与 storyKey 相关的信息抽取成 `@project/artifacts/create-story/context.md`
   - Step-04/05 主要依赖这个 context 文件（小、可控、可复现），减少反复读取大文档导致的漂移

3) **把“需要精确解析”的工作交给 Runtime（语义化工具）**
   - YAML/JSON/CSV 的精确字段提取、排序选择、状态机更新等，尽量不要让 LLM 自己在长文本里“人肉解析”
   - 用工具返回派生结果（storyKey、status、路径、结构化片段），LLM 只负责写作/决策解释

4) **Runtime 做硬校验（Graph + Schema + Checklist）**
   - graph：禁止跳到非允许后继节点
   - state：frontmatter 必须可解析且字段合法
   - 产物：可对关键产物做 checklist 校验（缺 section/缺字段/链接不存在）→ 失败就把错误回传给 LLM 修正

5) **不确定就停（Need-more-evidence 规则）**
   - 在 system/base rules 里明确要求：如果信息不在已读取片段中，必须继续 toolCall 获取证据或向用户提问；禁止凭空补全“事实类”内容

### 2.3.1 “先搜索定位 → 再按行号精读”的多轮 toolCall 模式（推荐）

这类操作通常需要 **至少两轮** LLM↔tool 往返，因为第二步（读哪几行）依赖第一步（搜索命中行号）：

**Round A（定位）**：LLM 先发 `fs.search`（关键词通常来自 userInput / variables.storyKey / step 指令）。  
**Round B（精读）**：Runtime 回传命中行号后，LLM 再发 `fs.read(startLine,endLine)` 读取“命中附近窗口”。

示例（简化）：

1) LLM → toolCalls

```json
[
  {
    "name": "fs.search",
    "arguments": {
      "query": "1-2-user-authentication",
      "globs": ["@project/artifacts/epics.md"]
    }
  }
]
```

2) Runtime → tool results（带行号）

```json
{
  "ok": true,
  "matches": [
    { "path": "@project/artifacts/epics.md", "line": 132, "text": "## 1.2 User Authentication ..." }
  ]
}
```

3) LLM → toolCalls（按行号读取窗口）

```json
[
  {
    "name": "fs.read",
    "arguments": {
      "path": "@project/artifacts/epics.md",
      "startLine": 112,
      "endLine": 200
    }
  }
]
```

> 注意：LLM 一次响应可以返回多个 toolCalls（用于并行读多个文件）；但 **有依赖链**（search→read window）的部分通常分两轮完成更稳。

### 2.3.2 ToolHost 如何“逼模型走窄读”（不需要额外 LLM）

当 `fs.read` 不带范围且文件过大时，ToolHost 直接返回“截断 + 提示”，模型就会自然改用 `search + range read`：

```json
{
  "ok": true,
  "path": "@project/artifacts/epics.md",
  "bytes": 120000,
  "truncated": true,
  "contentPreview": "# Epics\\n... (truncated)",
  "hint": "File too large. Use fs.search to locate relevant sections, then fs.read({startLine,endLine}) to fetch a small window."
}
```

Runtime 在这里做的是确定性策略（限额/截断/提示），不需要再调用一次 LLM 来决定“回传什么”。

> 结论：不回全文并不是“减少信息”，而是把信息获取从“推送全文”改为“模型按需拉取 + Runtime 校验兜底”。这也是 Cursor/Claude Code 类产品在长上下文/大仓库里保持可用性的核心手段。

---

## 2.4 用户输入后，如何形成 `user` message 的 content（建议规范）

你需要区分两类 `user` message：

### A) Runtime 注入的“控制指令”（不是用户原话）

用于：StartWorkflow / ResumeRun / Node 变化时重新锚定。建议统一成一个可机器解析的文本壳（即使不用 JSON 也要字段固定）。

模板（推荐）：

```text
RUN_DIRECTIVE
- intent: start|continue|resume
- workflow: <workflowId or workflow.md path>
- state: @state/workflow.md
- graph: @pkg/<...>/workflow.graph.json
- artifactsRoot: @project/artifacts/
- currentNodeId: <nodeId>
- effectiveAgentId: <agentId>
- autopilot: true (continue until you need user input or workflow ends)
```

completed 后（Post-Completion Profile）建议模板：  
- 不发送 `NODE_BRIEF` / step file / step markdown  
- `RUN_DIRECTIVE` 不包含 `currentNodeId`  
- 注入 Post-Run Protocol（允许继续对话与工具；状态变更需确认）

```text
RUN_DIRECTIVE
- intent: continue
- workflow: <workflowId or workflow.md path>
- state: @state/workflow.md
- graph: @pkg/<...>/workflow.graph.json
- artifactsRoot: @project/artifacts/
- effectiveAgentId: <agentId>
- autopilot: true|false

Protocol (post-run):
1) This workflow is already completed. Do NOT assume there is an active step to execute.
2) Continue as a normal conversation in Run mode: answer questions, review artifacts, and help the user iterate.
3) Tools are still available. Use tools when helpful.
4) If you need to modify @state/workflow.md (including changing completion status), you MUST ask the user for confirmation first.
5) If the user confirms a state change and the workflow becomes non-completed, resume the normal workflow protocol on subsequent turns.
```

### B) 用户原始输入（用户回答问题/补充信息）

用于：runMode 下用户回答 step 的问题。建议 **带上 nodeId 标签**，避免模型把回答应用到错误 step（尤其是 resume/跨天）。

模板（推荐）：

```text
USER_INPUT
- forNodeId: <currentNodeId>
<raw user text>
```

Post-Completion Profile（workflow 已 completed）下：不再有 active step，因此不绑定节点：

```text
USER_INPUT
<raw user text>
```

> 注意：不要把用户输入“改写成你理解的摘要”再发给 LLM；可以原文 +（可选）一行“你是否指的是…”的澄清，但要明确标注为 runtime 的注释。

---

## 2.5 `start workflow` vs `continue workflow` 怎么区分（以及其实你可以不区分）

### 2.5.1 区分的依据（不要靠用户文本猜）

在 Runtime 层按 run 生命周期判断即可：

- **start**：Run 刚创建（`runId` 新生成，`@state/workflow.md` 刚初始化，通常 `stepsCompleted` 为空）
- **resume**：Run 已存在但会话重启/重开项目（从 `@state/workflow.md` 读回状态后继续）
- **continue**：同一次 run 内持续推进（toolcall loop 或用户刚回复）

### 2.5.2 你也可以统一成一种说法

从模型角度，“start/continue”只是提示语；真正决定执行位置的是：

- `@state/workflow.md` 的 `currentNodeId`
- `@pkg/workflow.graph.json` 的允许出边
- 当前 node 的 step 文件指令

因此你可以永远只发一种 RUN_DIRECTIVE：

```text
RUN_DIRECTIVE
- intent: continue
- state: @state/workflow.md
- graph: ...
...
```

只要 `currentNodeId` 正确，模型自然会从 entry 开始或从中途继续。

---

## 3. 固定模板（你可以直接落代码的文字骨架）

### 3.1 System(BaseRuntimeRules)（建议）

关键约束必须包含：

- 你在运行一个 `.bmad` workflow（micro-file+graph）
- `workflow.graph.json` 是“跳转真源”，不允许跳到不在出边集合里的 node
- `workflow.md`（state）需要在每个 node 完成后更新 frontmatter
- mount alias 规则（避免泄露 RuntimeStoreRoot）
- 默认产物目录在 Project 内（用户可见）

示例（精简版）：

```text
You are running a CrewAgent v1.1 micro-file workflow.

Mounts:
- @project: user project (read/write). Default artifacts root is @project/artifacts/.
- @pkg: package content (read-only).
- @state: run state (read/write, user-invisible). Workflow state file is @state/workflow.md.

Rules:
- workflow.graph.json is the source of truth for allowed transitions.
- Always follow the current node's step file instructions before writing artifacts/state.
- After completing a node, update @state/workflow.md frontmatter: stepsCompleted, currentNodeId, variables (if needed), decisionLog (if decision), artifacts, updatedAt (if required).
- Do not write to @pkg. Do not reveal real filesystem paths; only use mount paths.
```

### 3.2 System(ToolPolicy)

从 Settings 的 SystemToolPolicy（系统级开关/限额）与 `agents.json` 的 `agent.tools`（agent 级策略，可进一步收紧）合并出最终策略并声明出来（默认：SystemToolPolicy 全开）。

例如（来自 `examples/multi-workflows/agents.json` 的 analyst：`fs.enabled=true, mcp.enabled=false`）：

```text
Tool policy:
- fs: enabled (maxReadBytes=524288, maxWriteBytes=1048576)
- mcp: disabled

Prefer fs.apply_patch for edits. All writes must be under @project or @state.
```

### 3.3 System(Persona)

从 `agents.json` 渲染 persona。以 `examples/multi-workflows/agents.json` 的 `analyst` 为例：

```text
You are Mary (Business Analyst).
Role: Strategic Business Analyst + Requirements Expert
Identity: Senior analyst focused on translating vague needs into actionable specs.
Communication style: Ask clarifying questions, then summarize constraints and acceptance criteria in concise bullets.
Principles:
- Ground findings in verifiable evidence.
- Write requirements with precision and clear scope boundaries.
- Prefer small, testable increments.
Stay in character unless the user dismisses the agent.
```

> 如果 `agent.systemPrompt` 存在：直接使用它替代渲染结果。

---

## 4. 示例 A：从 Agent（analyst）启动 `hello-linear`（multi-workflows）

### 4.1 背景（来自 examples）

- 包：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/multi-workflows/bmad.json`
  - workflow index：`hello-linear` 对应：
    - `workflow`: `workflows/hello-linear/workflow.md`
    - `graph`: `workflows/hello-linear/workflow.graph.json`
- graph：`.../workflows/hello-linear/workflow.graph.json`
  - entryNodeId：`step-01-ask`
  - `step-01-ask.file`: `workflows/hello-linear/steps/step-01-ask.md`
- step-01：`.../workflows/hello-linear/steps/step-01-ask.md`
  - outputs：`artifacts/hello-linear/input.md`
  - setsVariables：`topic`
  - next：`step-02-generate`

### 4.2 PromptComposer 输出（messages[] 示例）

下面是“启动 run 后的第一轮”推荐 messages（示意，省略 tools 定义本体）：

```json
[
  {
    "role": "system",
    "content": "You are running a CrewAgent v1.1 micro-file workflow...\nMounts: ...\nRules: ... (see BaseRuntimeRules)"
  },
  {
    "role": "system",
    "content": "Tool policy:\n- fs: enabled (maxReadBytes=524288, maxWriteBytes=1048576)\n- mcp: disabled\nPrefer fs.apply_patch for edits."
  },
  {
    "role": "system",
    "content": "You are Mary (Business Analyst)... (persona rendered from agents.json)"
  },
  {
    "role": "user",
    "content": "RUN_DIRECTIVE\n- intent: start\n- workflow: hello-linear\n- state: @state/workflow.md\n- graph: @pkg/workflows/hello-linear/workflow.graph.json\n- artifactsRoot: @project/artifacts/\n- currentNodeId: step-01-ask\n- effectiveAgentId: analyst\n- autopilot: true\n\nProtocol:\n1) Read @state/workflow.md frontmatter to get currentNodeId.\n2) Read the graph and open the step file for currentNodeId.\n3) Follow the step instructions exactly.\n4) If the step requires user answers, ask the user and WAIT.\n5) Only after you have required inputs: write artifacts under @project/artifacts/... and update @state/workflow.md frontmatter."
  },
  {
    "role": "user",
    "content": "Node brief:\n- currentNodeId: step-01-ask\n- step file: @pkg/workflows/hello-linear/steps/step-01-ask.md\n- outputs (default map): artifacts/hello-linear/input.md -> @project/artifacts/hello-linear/input.md\n- allowed next: step-02-generate (label=next, default=true)\n"
  }
]
```

> 预期模型行为：先 `fs.read` state/graph/step，然后提出 step-01 里的 2 个问题；用户回答后再写 `@project/artifacts/hello-linear/input.md` 并 patch `@state/workflow.md`。

### 4.3 当用户回答问题时（USER_INPUT 示例）

假设模型在 `step-01-ask` 问了：
- “主题是什么？”
- “希望输出格式是什么？”

用户输入是：

> 我想生成一个团队周报模板，格式用 bullet。

推荐 Runtime 追加一条 user message（不要改写用户原话）：

```text
USER_INPUT
- forNodeId: step-01-ask
我想生成一个团队周报模板，格式用 bullet。
```

---

## 5. 示例 B：同一个 workflow 的下一节点（persona 切换）

当 step-01 完成后，state 的 `currentNodeId` 会变为 `step-02-generate`，graph 指定该 node 的 `agentId: "architect"`（Winston）。

PromptComposer 的变化点：

- `effectiveAgentId` 从 `analyst` 切为 `architect`
- system persona 改为 Winston
- Node brief 更新 inputs/outputs

示意 messages（只展示变化的关键片段）：

```json
[
  { "role": "system", "content": "(BaseRuntimeRules unchanged)" },
  { "role": "system", "content": "(ToolPolicy unchanged)" },
  { "role": "system", "content": "You are Winston (Architect)... (from agents.json id=architect)" },
  {
    "role": "user",
    "content": "RUN_DIRECTIVE\n- intent: continue\n- workflow: hello-linear\n- state: @state/workflow.md\n- graph: @pkg/workflows/hello-linear/workflow.graph.json\n- artifactsRoot: @project/artifacts/\n- currentNodeId: step-02-generate\n- effectiveAgentId: architect\n- autopilot: true\n\nNode brief:\n- currentNodeId: step-02-generate\n- step file: @pkg/workflows/hello-linear/steps/step-02-generate.md\n- inputs: artifacts/hello-linear/input.md -> @project/artifacts/hello-linear/input.md\n- outputs: artifacts/hello-linear/result.md -> @project/artifacts/hello-linear/result.md\n- allowed next: end-99 (label=next, default=true)"
  }
]
```

> 预期模型行为：读 `@project/artifacts/hello-linear/input.md`，生成 `result.md`，并更新 `@state/workflow.md`。

---

## 6. 示例 C：分支 decision 节点（hello-branch）

来自 `crewagent-runtime/spec/bmad-package-spec/v1.1/examples/hello-branch/`：

- graph：`@pkg/workflow.graph.json`
  - decision node：`decide-02-project-type`（agentId=pm）
  - edges：
    - `greenfield` → `step-03a-greenfield`（conditionText: `variables.projectType == 'greenfield'`）
    - `brownfield` → `step-03b-brownfield`（isDefault=true）
- step：`@pkg/steps/step-02-decide-project-type.md`
  - 要求设置 `variables.projectType`
  - 要求写入 `decisionLog`（from/to/label/reason/decidedAt）

PromptComposer 的关键：把“允许的分支 + default + conditionText”写进 Node brief，减少模型走错分支。

示意 messages：

```json
[
  { "role": "system", "content": "(BaseRuntimeRules)" },
  { "role": "system", "content": "(ToolPolicy)" },
  { "role": "system", "content": "You are John (Product Manager)... (agents.json id=pm)" },
  {
    "role": "user",
    "content": "RUN_DIRECTIVE\n- intent: continue\n- workflow: hello-branch\n- state: @state/workflow.md\n- graph: @pkg/workflow.graph.json\n- artifactsRoot: @project/artifacts/\n- currentNodeId: decide-02-project-type\n- effectiveAgentId: pm\n- autopilot: true\n\nNode brief:\n- currentNodeId: decide-02-project-type (type=decision)\n- step file: @pkg/steps/step-02-decide-project-type.md\n- inputs: artifacts/intake.md -> @project/artifacts/intake.md\n- allowed next:\n  - step-03a-greenfield (label=greenfield, conditionText=\"variables.projectType == 'greenfield'\")\n  - step-03b-brownfield (label=brownfield, default=true, conditionText=\"variables.projectType == 'brownfield'\")\n\nDecision rules:\n- If intake is unclear, ask the user to clarify greenfield vs brownfield.\n- Then set variables.projectType and append one entry to decisionLog in @state/workflow.md.\n"
  }
]
```

---

## 7. DataContext（menuItem.data）的注入方式（推荐）

当 menuItem 有 `data`（例如 BMAD agent 的 handler 语义），你应当在启动/继续前把 data 的“摘要”注入为一条额外 user message，格式固定，便于模型引用。

示例（模板）：

```text
Extra context (from menuItem.data):
- label: project-context
- path: @project/docs/project-context.md
- preview:
  <first N KB here, truncated>
If you need more, use fs.read(@project/docs/project-context.md).
```

> 注意：data 只是“附加上下文”，不能替代 workflow/step 文件的权威指令。

---

## 8.（扩展）示例 D：BMAD 经典 workflow（create-story：workflow.yaml + instructions.xml + workflow.xml runner）

> 说明：你给的 `create-story` 是 **经典型**（`workflow.yaml` + `instructions.xml` + `template.md` + `checklist.md`），由 `_bmad/core/tasks/workflow.xml` 这个“执行引擎”驱动。  
> 如果你要做到 Cursor 的 `*create-story` 那种“自动跑完整流程（除非缺信息）”，核心就是两点：
>
> 1) **ExecutionEngine toolcall loop**：同一次触发内部多轮 LLM↔tools 直到 need user input / done。  
> 2) **YOLO/autopilot**：把“需要用户确认/可选步骤/模板分段确认”全部自动继续（BMAD 里用 `#yolo` 概念）。

### 8.1 关键文件（来自你项目）

- Workflow config：`@project/_bmad/bmm/workflows/4-implementation/create-story/workflow.yaml`
- Instructions：`@project/_bmad/bmm/workflows/4-implementation/create-story/instructions.xml`
- Template：`@project/_bmad/bmm/workflows/4-implementation/create-story/template.md`
- Checklist：`@project/_bmad/bmm/workflows/4-implementation/create-story/checklist.md`
- Runner（执行引擎）：`@project/_bmad/core/tasks/workflow.xml`

> 如果未来你把 BMAD 内容也当成“私有包”放 RuntimeStore，那么上述路径会变成 `@pkg/...`；messages 结构不变，只换 mount。

### 8.2 建议的 RUN_DIRECTIVE（classic-workflow 变体）

对于 classic workflow，我建议 RUN_DIRECTIVE 增加两个字段：`runType` 和 `yolo`。

```text
RUN_DIRECTIVE
- runType: bmad-classic
- intent: start|continue|resume
- workflowRunner: @project/_bmad/core/tasks/workflow.xml
- workflowConfig: @project/_bmad/bmm/workflows/4-implementation/create-story/workflow.yaml
- autopilot: true
- yolo: true

Constraints:
- Follow workflow.xml runner rules and execute instructions.xml steps in order.
- Do not ask the user to "continue" after template-output; treat yolo=true as auto-continue.
- Only ask the user when required inputs are missing (e.g., story key / missing sprint-status / missing docs).
```

### 8.3 从 SM Agent 菜单触发 `*create-story` 的一次“启动” messages（示意）

```json
[
  { "role": "system", "content": "(BaseRuntimeRules + mounts + no real paths)" },
  { "role": "system", "content": "(ToolPolicy from agent.tools)" },
  { "role": "system", "content": "(Persona: SM / Bob... plus: for *create-story always yolo)" },
  {
    "role": "user",
    "content": "RUN_DIRECTIVE\n- runType: bmad-classic\n- intent: start\n- workflowRunner: @project/_bmad/core/tasks/workflow.xml\n- workflowConfig: @project/_bmad/bmm/workflows/4-implementation/create-story/workflow.yaml\n- autopilot: true\n- yolo: true\n\nProtocol:\n1) LOAD and READ workflowRunner completely.\n2) Pass workflowConfig as 'workflow-config' parameter and execute workflowRunner instructions.\n3) Continue autonomously using tools until you MUST ask the user for missing inputs, or the workflow ends.\n"
  }
]
```

> 这会触发你观察到的“多次自主交互”：模型会用 toolCalls 去读 `workflow.yaml`、读 config、发现/读 epics/architecture/ux、写 story 文件、更新 sprint-status 等。

### 8.4 当 workflow 需要用户补充信息时：USER_INPUT（classic-workflow 的标注）

例如 `instructions.xml` 第 1 步会在缺 `sprint-status.yaml` 或缺 story key 时询问用户。此时你只需要把用户回答作为一条 USER_INPUT 追加：

```text
USER_INPUT
- for: bmad-classic
- workflow: create-story
1-2-user-authentication
```

可选：为了减少模型漂移，你可以在同一轮调用中 **先发一个 `RUN_DIRECTIVE intent: continue`** 再发 USER_INPUT（或合并在同一条 user message 里）。

### 8.5 `start` / `continue` / `resume` 的落点（classic 同理）

- `start`：新建 run 时（第一次触发菜单命令）
- `continue`：同一次 run 内继续推进（toolcall loop 或用户回复后）
- `resume`：应用重启/重新打开项目后，从 RuntimeStore 读到该 run 并继续

重要的是：对经典 workflow 来说，“从哪继续”更多依赖 **输出文件与 runner 指令**，不是 micro-file 的 `currentNodeId`。因此建议 RuntimeStore 记录：
- runType、workflowConfig 路径、已生成 story 文件路径（如果已生成）、最后一次等待用户的提示文本（可选）
