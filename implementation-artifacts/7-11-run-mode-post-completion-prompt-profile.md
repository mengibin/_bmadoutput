# Story 7.11: Run Mode Post-Completion Prompt Profile (No Active Step)

Status: review

## Story

As a **Runtime user**,  
I want `phase: Completed` to be only a “workflow ended” label (not a hard stop),  
So that I can keep chatting and still use tools after the workflow completes.

## Acceptance Criteria

1. **Completed 非硬停止**
   - **Given** workflow 已完成（例如 `variables.workflowStatus="complete"`）
   - **When** engine 返回 `phase: Completed`
   - **Then** UI 只展示“流程结束”标签，输入框仍可用，后续仍可继续对话（Run 模式）

2. **Post-Completion Prompt Profile（无 active step）**
   - **Given** workflow 已完成
   - **When** Runtime 组装下一轮 LLM prompt
   - **Then**：
     - 仍注入 `RUN_DIRECTIVE`（保留 workflow 级信息：graph/workflowId/runId 等；不新增 `runId` 字段）
     - `RUN_DIRECTIVE` 不包含 `currentNodeId`
     - 不发送 `NODE_BRIEF`
     - 不注入 step markdown（如 `STEP_FILE_CONTENT` / `stepFile` / `nodeBrief` / currentNodeId/forNodeId）
     - `USER_INPUT` 不包含 `forNodeId`
     - `RUN_DIRECTIVE` 注入新的 `Protocol (post-run)`（允许继续对话/工具；状态变更需确认）

3. **工具调用仍允许**
   - **Given** workflow 已完成
   - **When** 后续对话继续
   - **Then** tools 仍按 ToolPolicy 暴露（不强制 `tool_choice=none`）

4. **状态变更需确认（Runtime 强制）**
   - **Given** workflow 已完成
   - **When** LLM 尝试修改 `@state/workflow.md`（包括改变 completion 状态）
   - **Then** Runtime 必须先获得用户确认（confirmation widget），否则拒绝执行并要求确认
   - **And** 确认是一次性令牌：确认后允许一次状态变更操作，随后必须重新确认

## Design

### Summary
- completed 后进入 **Post-Completion Profile**：只注入 `RUN_DIRECTIVE`（含 post-run protocol），不注入任何 step/node 信息（无 active step 概念）。
- `phase: Completed` 仅作为 UI 标签；对话与工具调用继续（仍在 Run 模式）。
- 对 `@state/workflow.md` 的修改在 completed 后必须走 **confirmation widget**，并由 Runtime 强制拦截（一次性确认令牌）。
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-7-11-run-mode-post-completion-prompt-profile.md`

### UX / UI
- completed 后 UI 展示状态：`Completed（流程已结束，可继续对话）`；输入框保持可用。
- 当需要变更 `@state/workflow.md` 时：通过 `ui.ask_user(type="confirmation")` 弹出确认控件（复用现有 ConfirmationWidget），用户确认/取消后继续。

### API / Contracts
- `ui.ask_user`（confirmation）：
  - `widgetId: "workflow_state_change_confirm"`
  - `type: "confirmation"`
  - `message`: 必须描述将要对 `@state/workflow.md` 做的变更（包括是否会把 workflow 从 completed 改回非 completed）
- 用户提交协议（前端约定）：
  - `WIDGET_SUBMIT\n{ widgetId, type:"confirmation", value:{ confirmed:true|false } }`
- Runtime 侧错误约定（tool result）：
  - 未确认时拒绝状态变更工具调用，并返回 `STATE_CHANGE_REQUIRES_CONFIRMATION`（提示模型先请求确认）

### Data / Storage
- completed 判定以 `@state/workflow.md` frontmatter 为真源：
  - `variables.workflowStatus === "complete"` → completed
- “确认令牌”不落盘：只从“最近一次用户输入”的 `WIDGET_SUBMIT` 解析得到，且只允许消费一次。

### Errors / Edge Cases
- completed 但模型仍尝试继续按 step 执行：应被 prompt profile（无 step 注入 + post-run protocol）纠正。
- 用户取消确认：不得执行任何状态变更；模型应继续以对话方式解释与给出替代方案。
- 模型试图复用旧确认：必须被一次性令牌规则阻止（需要重新确认）。
- 状态变更后 workflow 不再 completed：后续恢复 **Normal Run Profile**（重新注入 active step 信息）。

### Test Plan
- Unit:
  - PromptComposer：completed 时不输出 `currentNodeId/forNodeId`，不发 `NODE_BRIEF`，注入 post-run protocol。
  - SystemPromptComposer：completed 时不渲染 current step / transitions（避免 “Unknown Step”）。
  - ExecutionEngine/chatToolLoop：completed + 未确认时拦截 `@state/workflow.md` 的写操作；确认后允许一次。
- Integration:
  - 跑通 workflow → completed → 继续对话（工具可用）。
  - completed 后请求修改状态：先确认，再执行；执行后恢复 normal profile（当 workflow 变为未完成）。

## Tasks / Subtasks
- [x] PromptComposer：新增 Post-Completion Profile 分支（RUN_DIRECTIVE 无 currentNodeId；USER_INPUT 无 forNodeId；不发 NODE_BRIEF）
- [x] SystemPromptComposer：completed 时不渲染 current step / transitions
- [x] ExecutionEngine/chatToolLoop：completed 后对 `@state/workflow.md` 的写操作强制 confirmation widget（一次性令牌）
- [x] 新增 prompt template：`run-directive-protocol-post-run.md`
- [x] 单元测试 + 回归测试

## Dev Agent Record

### Debug Log
- 2026-01-25: Added unit tests for post-completion prompt profile and runtime confirmation gate.
- 2026-01-25: Implemented completed-aware prompt injection (no active step), step md omission, and one-shot confirmation gating.
- 2026-01-25: Ran `crewagent-runtime` tests (vitest) and verified green.

### Completion Notes
- `phase: Completed` remains a UI label; run sessions continue to accept chat + tools.
- Completed profile removes active-step injection (`currentNodeId/forNodeId/NODE_BRIEF/STEP_FILE_CONTENT`) and uses a new post-run protocol.
- Runtime blocks state-changing calls to `@state/workflow.md` after completion unless the user confirms via `workflow_state_change_confirm` (one-shot).

### File List
- `_bmad-output/implementation-artifacts/7-11-run-mode-post-completion-prompt-profile.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/services/promptComposer.test.ts`
- `crewagent-runtime/electron/services/prompt-templates/run-directive-protocol-post-run.md`
- `crewagent-runtime/electron/core/context-builder/systemPromptComposer.ts`
- `crewagent-runtime/electron/core/context-builder/context-builder.test.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/services/llmAdapter.ts`
- `crewagent-runtime/electron/services/llmAdapter.test.ts`

### Change Log
- Added post-completion protocol template and prompt profile switching (completed vs normal run).
- Removed step injection after completion (prompt + step md) and removed `forNodeId` persistence in completed mode.
- Added runtime-enforced, one-shot confirmation gate for workflow state changes after completion.
- Fixed OpenAI-compatible streaming edge case where the last SSE event can be dropped if the delimiter is missing (prevents truncated assistant output).

## References
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-7-11-run-mode-post-completion-prompt-profile.md`
- Design: `_bmad-output/implementation-artifacts/design-7-11-run-mode-post-completion-prompt-profile.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Related: `_bmad-output/implementation-artifacts/7-7-integrate-context-builder-run-mode.md`
