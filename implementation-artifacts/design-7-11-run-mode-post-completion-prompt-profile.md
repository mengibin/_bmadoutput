# Design: Run Mode Post-Completion Prompt Profile (No Active Step)

**Story:** `7-11-run-mode-post-completion-prompt-profile.md`  
**设计原则:** Completed 仅是标签、Prompt profile 明确分层、状态变更必须 runtime 强制确认

---

## 设计目标

1. **Completed 非硬停止**：workflow 完成后仍保持 Run 模式可继续对话与工具调用
2. **去除 active step 概念**：completed 后不注入 step/node 信息，避免误导模型“继续跑 step”
3. **状态变更需确认（强制）**：completed 后对 `@state/workflow.md` 的任何修改必须先获得用户确认

---

## 关键决策

### 1) 两种 Prompt Profile

| Profile | 条件 | 注入内容 |
|---|---|---|
| **Normal Run** | workflow 未 completed | `RUN_DIRECTIVE`(含 currentNodeId + active-run protocol) + `NODE_BRIEF` +（可选）step context + `USER_INPUT`(含 forNodeId) |
| **Post-Completion** | workflow 已 completed | `RUN_DIRECTIVE`(不含 currentNodeId + post-run protocol) + `USER_INPUT`(不含 forNodeId)；不发送 `NODE_BRIEF`/step context |

### 2) completed 判定（source of truth）

以 `@state/workflow.md` frontmatter 为准：
- `variables.workflowStatus === "complete"` → completed
- 其他 end-node heuristics 可作为兼容兜底，但 **Post-Completion Profile** 必须至少覆盖 `workflowStatus` 路径

---

## Prompt 组装改动点（覆盖所有注入路径）

> 目标：completed 后 “active step” 信息 **不出现** 在任何 system/user 注入层。

### A) PromptComposer：RUN_DIRECTIVE / NODE_BRIEF / USER_INPUT

#### 变更点
- 引入 `workflowCompleted: boolean`（由 ExecutionEngine 从 state 计算后传入 compose）
- completed 时：
  - `RUN_DIRECTIVE`：不输出 `- currentNodeId: ...`
  - 不发送 `NODE_BRIEF`
  - `USER_INPUT`：不输出 `- forNodeId: ...`
  - 注入 post-run protocol（模板见下）

#### 模板拆分
- 保留：`crewagent-runtime/electron/services/prompt-templates/run-directive-protocol.md`（active-run protocol）
- 新增：`crewagent-runtime/electron/services/prompt-templates/run-directive-protocol-post-run.md`（post-run protocol）

#### 伪代码（compose 分支）

```ts
const completed = isWorkflowCompleted(state)

const runDirective = completed
  ? renderRunDirective({ /* without currentNodeId */ protocol: postRunProtocol })
  : renderRunDirective({ /* with currentNodeId */ protocol: activeRunProtocol })

const nodeBrief = completed ? null : renderNodeBrief(...)

const userInput = rawUserInput
  ? completed
      ? renderUserInput({ raw: rawUserInput })
      : renderUserInput({ raw: rawUserInput, forNodeId: state.currentNodeId })
  : null
```

---

### B) ExecutionEngine：step markdown / STEP_FILE_CONTENT 注入

#### 变更点
- 当 workflow 已 completed 时：
  - 禁止注入 step md（`STEP_FILE_CONTENT` / `stepFile` / `nodeBrief` 等）
  - 仍保持 run 模式对话与工具调用（工具列表不强制关闭）

> 重要：completed 后 Prompt 仍会保留 RUN_DIRECTIVE（graph/workflowId/runId 等需要保持；本 story 不新增 `runId` 到 RUN_DIRECTIVE）。

---

### C) SystemPromptComposer（context-builder run mode）：Current Step / Transitions

#### 变更点
system prompt 的 run-mode “Run Context” 区块在 completed 后应：
- 不渲染 `Current Step`（避免出现 “Unknown Step (unknown)”）
- 不渲染 `Available Transitions`
- 可选：渲染 `Workflow Status: completed`（不包含 currentNodeId）

---

## 状态变更确认（Runtime-Enforced）

### 1) 需要确认的动作（state-changing tools）

在 Post-Completion Profile 下，以下工具调用视为状态变更，必须先确认：
- `fs.apply_patch` 且 `path === "@state/workflow.md"`
- `fs.write` 且 `path === "@state/workflow.md"`
- `workflow.rewind`（无论 workflow 是否 completed，都要求显式 confirmed；但 completed 下还必须有用户确认）

### 2) 确认获取方式（统一使用 confirmation widget）

LLM 调用：

```json
{
  "name": "ui.ask_user",
  "arguments": {
    "widgetId": "workflow_state_change_confirm",
    "type": "confirmation",
    "message": "I want to modify @state/workflow.md to ... (describe changes). Proceed?"
  }
}
```

用户提交消息格式（前端现有约定）：

```text
WIDGET_SUBMIT
{
  "widgetId": "workflow_state_change_confirm",
  "type": "confirmation",
  "value": { "confirmed": true }
}
```

### 3) 引擎侧强制规则（一次性令牌）

- 引擎在每轮开始时解析“最近一次 user 输入”：
  - 若是 `WIDGET_SUBMIT` 且 `widgetId==="workflow_state_change_confirm"` 且 `confirmed===true` → `stateChangeConfirmed=true`
- 执行 toolCalls 时：
  - 若 `isCompleted && isStateChangingTool(call) && !stateChangeConfirmed`：
    - **不执行**工具调用
    - 返回 tool result error（例如 `STATE_CHANGE_REQUIRES_CONFIRMATION`），并要求模型先走确认流程
  - 若允许执行一次状态变更工具调用后：立刻 `stateChangeConfirmed=false`（避免复用）

> 目的：即使模型不遵守 system 指令，也无法“静默改状态”。

---

## 测试策略（补齐 story 缺口）

### Unit Tests
1) `PromptComposer`：
   - completed：RUN_DIRECTIVE 不含 currentNodeId；不发 NODE_BRIEF；USER_INPUT 不含 forNodeId；包含 post-run protocol
   - not completed：行为保持不变
2) `SystemPromptComposer`：
   - completed：不渲染 Current Step / Available Transitions
3) `ExecutionEngine` / `chatToolLoop`：
   - completed + 无确认：拦截 `fs.apply_patch(@state/workflow.md)`，返回 `STATE_CHANGE_REQUIRES_CONFIRMATION`
   - completed + 有确认：允许一次 `fs.apply_patch(@state/workflow.md)`；第二次再次要求确认

### Manual Verification
1) 跑通任意 workflow → completed
2) 继续输入自由对话 → 正常回复、工具可用
3) 诱导模型修改 `@state/workflow.md`：
   - 未确认：必须先出现确认流程
   - 确认后：允许修改；若变为未完成，下一轮恢复 Normal Run profile 注入

---

## 文件清单（实现时）

| File | Changes |
|------|---------|
| `crewagent-runtime/electron/services/promptComposer.ts` | completed 分支：RUN_DIRECTIVE/USER_INPUT 变体；不发 NODE_BRIEF |
| `crewagent-runtime/electron/services/executionEngine.ts` | completed 下不注入 step context；实现 state-change tool 强制确认 |
| `crewagent-runtime/electron/core/context-builder/systemPromptComposer.ts` | completed 下不渲染 Current Step/Transitions |
| `crewagent-runtime/electron/services/prompt-templates/run-directive-protocol-post-run.md` | **NEW**：post-run protocol 文本模板 |

