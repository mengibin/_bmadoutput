# Tech-Spec: Run Mode Post-Completion Prompt Profile (No Active Step)

**Created:** 2026-01-25  
**Status:** Ready for Development  
**Source Story (if applicable):** `_bmad-output/implementation-artifacts/7-11-run-mode-post-completion-prompt-profile.md`

## Overview

### Problem Statement

当前 Run 模式在 workflow 结束（`variables.workflowStatus="complete"`）后存在两类体验/协议问题：

1) `phase: Completed` 被当作“硬停止”语义使用，导致 workflow 结束后无法自然继续对话/继续使用工具。  
2) workflow 已 completed 后仍注入 active step 信息（`currentNodeId/forNodeId/NODE_BRIEF/step md` 等），会误导模型继续执行 step，造成 UI 上“收尾回复缺失/回答被截断/对话逻辑混乱”的体感。

### Solution

引入 **Run Mode Post-Completion Prompt Profile**（简称 Post-Completion Profile）：

- 当 workflow 已 completed 时：
  - `phase: Completed` 仅作为 UI 的“流程结束”标签，不硬停止会话
  - 仍保持 Run 模式对话与工具调用
  - Prompt 注入只保留 workflow 级 `RUN_DIRECTIVE`，移除全部 active step 注入（无 `currentNodeId/forNodeId/NODE_BRIEF/step md`）
  - `RUN_DIRECTIVE` 注入 `Protocol (post-run)`，明确“workflow 已完成，不假设 active step”
  - 对 `@state/workflow.md` 的修改必须走 **confirmation widget** 并由 Runtime 强制拦截（一次性确认令牌）

### Scope (In/Out)

**In Scope**
- completed 后的 prompt profile 切换与注入规则（PromptComposer + context-builder/system prompt + step md 注入点）
- completed 后继续对话/工具调用的语义（Completed 仅为标签）
- completed 后对 `@state/workflow.md` 的状态变更强制确认机制（runtime-enforced）
- 单元测试与回归测试覆盖

**Out of Scope**
- UI 视觉/文案的大改（仅需要保证输入框可用；文案可后续迭代）
- 新的工具或新的持久化结构（确认令牌不落盘）
- 修复其它历史遗漏的 sprint-status 条目（仅更新 7-11）

## Context for Development

### Codebase Patterns

- Run 模式的 LLM 请求由“system prompt（base rules/tool policy/persona）+ conversation window + run-only 注入”组成。
- run-only 注入当前存在多个入口（必须全部覆盖）：
  - PromptComposer 产生 `RUN_DIRECTIVE` / `NODE_BRIEF` / `USER_INPUT(forNodeId)`
  - ExecutionEngine 可能注入 step markdown（如 `STEP_FILE_CONTENT`）
  - context-builder 的 SystemPromptComposer 可能渲染 `Current Step` / `Available Transitions`
- 结构化用户确认推荐复用现有 `ui.ask_user(type="confirmation")` + 前端 `WIDGET_SUBMIT` 协议。

### Files to Reference

- Prompt 注入/组装：
  - `crewagent-runtime/electron/services/promptComposer.ts`
  - `crewagent-runtime/electron/services/prompt-templates/run-directive-protocol.md`
  - `crewagent-runtime/electron/services/prompt-templates/run-directive-protocol-post-run.md` (**NEW**)
- Run 执行与 step 注入：
  - `crewagent-runtime/electron/services/executionEngine.ts`
- System prompt（run-mode context）：
  - `crewagent-runtime/electron/core/context-builder/systemPromptComposer.ts`
- Widget/确认相关：
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`（`ui.ask_user` 工具定义）
  - `crewagent-runtime/src/pages/RunsPage/components/widgets/ConfirmationWidget.tsx`
  - `crewagent-runtime/src/pages/RunsPage/components/widgets/widgetSubmission.ts`（`WIDGET_SUBMIT` 生成）

### Technical Decisions

1) **两种 Profile**：Normal Run vs Post-Completion（completed 时无 active step 注入）。  
2) **completed 判定真源**：以 `@state/workflow.md` frontmatter 的 `variables.workflowStatus==="complete"` 为准。  
3) **确认机制 runtime-enforced**：即使模型不遵守 system 指令，也不能静默改 `@state/workflow.md`。  
4) **一次性确认令牌**：仅允许一次状态变更；避免历史确认被复用或被 prompt 注入滥用。  
5) **不新增 RUN_DIRECTIVE 字段**：RUN_DIRECTIVE 里原来没有 `runId` 就不加，保持现状。  

## Implementation Plan

### Tasks

- [ ] 新增 post-run protocol 模板文件：`crewagent-runtime/electron/services/prompt-templates/run-directive-protocol-post-run.md`
- [ ] PromptComposer：增加 completed 分支
  - completed：RUN_DIRECTIVE 不包含 `currentNodeId`，注入 post-run protocol；不发送 NODE_BRIEF；USER_INPUT 不包含 forNodeId
  - not completed：保持现状（active-run protocol + NODE_BRIEF + forNodeId）
- [ ] SystemPromptComposer（run-mode）：completed 时不渲染 Current Step / Available Transitions（避免 “Unknown Step”）
- [ ] ExecutionEngine/chatToolLoop：completed 下对状态变更工具调用做强制确认
  - 识别 state-changing calls：`fs.apply_patch(@state/workflow.md)`、`fs.write(@state/workflow.md)`、`workflow.rewind`
  - 未确认：返回 tool error（建议 code=`STATE_CHANGE_REQUIRES_CONFIRMATION`）并引导模型先 `ui.ask_user(type="confirmation")`
  - 已确认：允许一次状态变更调用；执行后清空确认
- [ ] 单元测试/回归测试：
  - PromptComposer completed 变体（无 currentNodeId/forNodeId、无 NODE_BRIEF、包含 post-run protocol）
  - SystemPromptComposer completed 变体（不渲染 step/transition）
  - Engine 拦截逻辑（未确认拒绝、确认后允许一次）

### Acceptance Criteria

- [ ] AC1: workflow completed 后 `phase: Completed` 仅为标签，输入框仍可用且可继续对话（Run 模式）
- [ ] AC2: completed 后 prompt 进入 Post-Completion Profile（无 step/node 注入，仅 RUN_DIRECTIVE + post-run protocol；USER_INPUT 无 forNodeId）
- [ ] AC3: completed 后 tools 仍按 ToolPolicy 可用（不强制 tool_choice=none）
- [ ] AC4: completed 后修改 `@state/workflow.md` 必须先确认；确认一次只允许一次状态变更

## Additional Context

### Dependencies

- 依赖现有 widget 能力：`ui.ask_user(type="confirmation")` + `WIDGET_SUBMIT`。
- 依赖现有 run 状态文件：`@state/workflow.md`。

### Testing Strategy

- Unit：PromptComposer/SystemPromptComposer/Engine（拦截 + 一次性令牌）
- Integration：跑通任意 workflow 到 completed → 继续对话；引导状态变更（确认/取消/确认后恢复 normal profile）

### Notes

- Post-Completion Profile 的目标不是“禁止工具”，而是“去掉 active step 注入 + 让 Completed 不硬停止”。  
- 状态变更的确认是 Runtime 强制，而不仅是提示词约束。  
- 如状态变更将 workflow 从 completed 改回未完成，后续应恢复 Normal Run Profile（重新注入 active step 信息）。  

## Traceability

- Story: `_bmad-output/implementation-artifacts/7-11-run-mode-post-completion-prompt-profile.md`
- Design: `_bmad-output/implementation-artifacts/design-7-11-run-mode-post-completion-prompt-profile.md`

