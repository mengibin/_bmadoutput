# 开发用例：create-story-micro（v1.1 micro-file + graph）

面向 Runtime 开发阶段：把 `create-story-micro` 当作 **端到端（E2E）golden path**，用来驱动 `PackageManager / RunManager / ExecutionEngine / ToolHost(fs)` 的实现与验收。

## 1) 先看哪些文件（从“入口”到“步骤”）

- 包入口清单：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/bmad.json`
- Agent（persona + menu 触发）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/agents.json`
- Workflow 状态文件（Document-as-State）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/workflow.md`
- 图真源（跳转与校验）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/workflow.graph.json`
- Step 脚本（LLM 执行规范）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/steps/step-01-select-story.md`
- Story 模板（产物骨架）：`crewagent-runtime/spec/bmad-package-spec/v1.1/examples/create-story-micro/assets/story-template.md`
- 理想交互 Trace（实现 ExecutionEngine 的“对照脚本”）：`_bmad-output/implementation-artifacts/runtime/create-story-micro-ideal-trace.md`
- LLM 对话协议（Tool result 回填规则 / OpenAI-compatible）：`_bmad-output/tech-spec/llm-conversation-protocol-openai.md`

## 2) 这个例子覆盖了哪些 Runtime 关键能力

- **Agent-First 入口**：`agents.json` 里 `sm.menu.trigger=create-story` → 启动 `workflow.md`。
- **mount alias**：LLM 只看到 `@project/@pkg/@state`；`@pkg` 必须只读；状态写入 `@state/workflow.md`；产物写 `@project/artifacts/`。
- **LLM-as-Engine**：Runtime 不“执行流程逻辑”，只做工具、沙箱、校验；流程推进由 LLM 读 step 指令并更新 state。
- **最小停点**：只有缺必需输入时才停下来问用户（Step-01/02 都有 WAIT 语义）。

## 3) E2E 场景（建议用于开发联调）

> 场景：项目里没有 `@project/artifacts/sprint-status.yaml`，因此 Step-01 会向用户询问 `storyKey`。

1. 用户在 Agent Session 输入 `create-story` → Runtime 解析 menu 并 StartWorkflow（见 `agents.json`）。
2. Runtime 创建 run，把包内容挂到 `@pkg`，把项目挂到 `@project`，把 state 挂到 `@state`：
   - 初始 `@state/workflow.md.currentNodeId = step-01-select-story`（见 `workflow.md`）。
3. 执行 Step-01（见 `steps/step-01-select-story.md`）：
   - 找不到 sprint-status → 询问用户输入 `storyKey`，并 WAIT（此时 **不允许推进** `currentNodeId`）。
4. 用户输入 `1-2-user-authentication` 后，LLM（sm persona）应：
   - 写 `@project/artifacts/create-story/target.md`
   - patch `@state/workflow.md`：`stepsCompleted+=step-01...`、写入 variables、`currentNodeId=step-02...`、`artifacts+=artifacts/create-story/target.md`
5. Step-02/03/04 自动推进，最终生成 story：
   - `@project/artifacts/stories/1-2-user-authentication.md`（骨架来自 `assets/story-template.md`；规则来自 `step-04-generate-story.md`）
6. Step-05 可选更新 sprint-status；end-99 生成总结 `@project/artifacts/create-story/summary.md`。

## 4) 作为开发验收的硬约束（Runtime 必须强制）

- `@pkg` 只读：任何写入都拒绝。
- `@state/workflow.md` 写入必须：
  - frontmatter YAML 可解析（失败则拒绝并返回可修复错误）
  -（可选但推荐）`currentNodeId` 变更必须是 `workflow.graph.json` 允许的后继
  - 原子写（tmp→rename）
- `artifacts/*` 输出默认映射到 `@project/artifacts/*`（用户可见）。

## 5) “跑通了”的最低验收清单

- `@state/workflow.md.stepsCompleted` 至少包含：`step-01...step-05` 与 `end-99`。
- `@state/workflow.md.artifacts` 至少包含：
  - `artifacts/create-story/target.md`
  - `artifacts/create-story/inputs.md`
  - `artifacts/create-story/context.md`
  - `artifacts/stories/1-2-user-authentication.md`
  - `artifacts/create-story/sprint-status-update.md`（无 sprint-status 时也应生成“跳过说明”）
  - `artifacts/create-story/summary.md`
- 项目目录中对应文件真实存在（写在 `@project/...`）。
