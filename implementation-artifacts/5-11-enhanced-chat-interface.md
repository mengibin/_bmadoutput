# Story 5-11: Enhanced Chat Interface Foundation (Streaming & Iceberg Model)

## Overview
**Epic**: 5 – Observability, Settings & Recovery
**Priority**: High
**Status**: `review`

## Goal
Redesign the Chat Page (RunWorkspace) to provide a modern, responsive user experience. Focus on smooth streaming rendering, isolation of Markdown styling for chat messages, and hiding internal execution details (Thoughts, Tool calls) to reduce visual noise.

## Business Value
- **User Experience**: Drastically improves the "feel" of the application, making it feel "alive" and responsive.
- **Clarity**: Reduces cognitive load by hiding internal execution details (The "Iceberg" Model), letting users focus on the conversational content.
- **Maintainability**: dedicated components for Chat Markdown ensure changes to chat styling don't break file viewers.

## Acceptance Criteria

### 1. Progressive Streaming UI
- **Streaming Rendering**: The chat UI must render tokens as they arrive (simulated or real).
- **Smooth Auto-scroll**: The chat should auto-scroll to the bottom as new content streaming in.

### 2. Clean Message Presentation (The "Iceberg" Model)
- **User Messages**: Distinct styling (Primary Color).
- **Assistant Content**: Markdown-rendered text.
- **Internal Events (Thoughts/Tool Calls)**:
    - **Hidden by default** in the main chat bubble.
    - **Visualization**: Show a discrete, animated indicator (e.g., "Thinking...", "Running fs.read...").
    - **Interaction**: Clicking the indicator reveals the details (either expands in place OR switches to the "Log" tab/sidebar).
    - **Log Tab Integration**: Ensure the "Log" tab in the sidebar remains the source of truth for full execution history.

### 3. Dedicated Markdown Renderer
- **Isolation**: Create a `MessageMarkdown` component specifically for chat messages.
- **Styling**: Chat-specific typography (e.g. smaller code blocks, different font sizes) separate from the File Viewer's `MarkdownRenderer`.
- **Features**: Support standard Markdown (code blocks, lists, bold/italic, links).

## Technical Components / Changes
1.  **Refactor `RunWorkspace.tsx`**: Break into cleaner sub-components:
    - `ChatPanel`
    - `MessageList`
    - `MessageItem`
    - `ChatInput`
2.  **`MessageItem` Logic**:
    - Handle `message.type` (content vs thought vs tool).
    - State for "collapsed" vs "expanded" thoughts.
3.  **`MessageMarkdown`**: New component in `src/pages/RunsPage/components/MessageMarkdown.tsx`.
    - Reuse logic from `MarkdownRenderer` but apply internal styles.

## Related Artifacts
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-11-enhanced-chat-interface.md`

## Verification Plan

### Manual Verification
1.  **Streaming**: Send a long message response. Verify text renders progressively.
2.  **Thoughts**: Trigger a workflow that "thinks". Verify only "Thinking..." appears. Click it -> Verify details are shown.
3.  **Tools**: Trigger a tool call. Verify "Running tool..." animation.
4.  **Markdown**: Send a message with complex Markdown (code, lists). Verify it renders correctly and looks distinct from the File Viewer.

### Automated Tests
- Unit tests for `MessageItem` rendering logic (hidden vs visible).
- Unit tests for `MessageMarkdown`.

## Tasks / Subtasks
- [x] 1. 梳理现有 RunWorkspace 与消息数据流，确认与 Tech Spec 对齐（含 ConversationMessage 扩展、持久化兼容）
  - [x] 1.1 扩展 `ConversationMessage` 类型（`partType`/`toolName`/`duration` 等），默认值向后兼容
  - [x] 1.2 若存在消息持久化 schema/序列化，更新以兼容新字段
- [x] 2. 拆分 RunWorkspace 并引入聊天子组件
  - [x] 2.1 新建 `ChatPanel` / `MessageList` / `MessageItem` / `ChatInput`
  - [x] 2.2 新建 `ThinkingIndicator` / `ToolStatusIndicator` 并完成交互
  - [x] 2.3 新建 `MessageMarkdown`（复制 file viewer 的解析逻辑并做样式隔离）
  - [x] 2.4 更新 `RunsPage` 样式
- [x] 3. 实现 Iceberg Model 展示逻辑
  - [x] 3.1 根据 `message.partType` 渲染分支（`content` / `thinking` / `tool_start` / `tool_result`）
  - [x] 3.2 默认折叠思考/工具信息，点击后展开或跳转 Log 面板
  - [x] 3.3 与 Log 面板保持一致（主聊天仅显示摘要/指示器）
- [x] 4. 实现流式渲染与自动滚动
  - [x] 4.1 接入 `llm:stream-*` IPC 事件（preload + renderer）
  - [x] 4.2 维护 streaming state（`partialContent` / `isStreaming` 等）
  - [x] 4.3 自动滚动：用户上滑时暂停，回到底部自动恢复
- [x] 5. 测试与验证
  - [x] 5.1 单测：`MessageItem` 渲染分支、`MessageMarkdown`、`ChatInput` Enter/Shift+Enter
  - [x] 5.2 手动验证：流式渲染、Iceberg 指示器交互、自动滚动、Markdown 样式

## Dev Agent Record

### Implementation Plan
- 扩展消息模型与持久化读取默认值，保证旧数据兼容。
- 拆分 RunWorkspace 并引入 ChatPanel / MessageList / MessageItem / ChatInput / MessageMarkdown 等组件。
- 增加 Iceberg 指示器与流式事件监听，配合自动滚动逻辑。
- 补充单测与手动验证路径。

### Debug Log
- `npm test` (crewagent-runtime) — 通过。
- `npm run lint` (crewagent-runtime) — lint 已清理通过（仍有 TypeScript 版本提示）。

### Completion Notes
- 新增聊天组件拆分与 `MessageMarkdown`，支持独立 Markdown 渲染样式。
- 实现 Iceberg 模式（Thinking/Tool 指示器默认折叠，可点击展开并切换 Logs）。
- 接入 `llm:stream-*` 事件并支持流式更新与自动滚动。
- 扩展 ConversationMessage 类型并为旧消息默认 `partType=content`。
- 测试：`npm test`、`npm run lint`（crewagent-runtime）。

## File List
- crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx
- crewagent-runtime/src/pages/RunsPage/RunsPage.css
- crewagent-runtime/src/pages/RunsPage/components/ChatPanel.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageList.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageItem.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageMarkdown.tsx
- crewagent-runtime/src/pages/RunsPage/components/ThinkingIndicator.tsx
- crewagent-runtime/src/pages/RunsPage/components/ToolStatusIndicator.tsx
- crewagent-runtime/src/pages/RunsPage/components/ChatInput.tsx
- crewagent-runtime/src/pages/RunsPage/components/chatInputUtils.ts
- crewagent-runtime/src/pages/RunsPage/components/MessageItem.test.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageMarkdown.test.tsx
- crewagent-runtime/src/pages/RunsPage/components/ChatInput.test.tsx
- crewagent-runtime/src/pages/WorksPage/WorksPage.tsx
- crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx
- crewagent-runtime/src/components/files/fileViewers.tsx
- crewagent-runtime/src/hooks/useWorkflowProgress.ts
- crewagent-runtime/src/stores/appStore.ts
- crewagent-runtime/electron/main.ts
- crewagent-runtime/electron/stores/runtimeStore.ts
- crewagent-runtime/electron/stores/runtimeStore.test.ts
- crewagent-runtime/electron/preload.ts
- crewagent-runtime/electron/electron-env.d.ts
- crewagent-runtime/electron/services/executionEngine.ts
- crewagent-runtime/electron/services/executionEngine.test.ts
- crewagent-runtime/electron/services/fileSystemToolHost.ts
- crewagent-runtime/electron/services/fileSystemToolHost.test.ts
- crewagent-runtime/electron/services/llmAdapter.ts
- crewagent-runtime/electron/services/llmAdapter.test.ts
- crewagent-runtime/electron/services/runManager.ts
- crewagent-runtime/electron/services/runManager.test.ts
- crewagent-runtime/vitest.config.ts
- _bmad-output/implementation-artifacts/sprint-status.yaml

## Change Log
- 2026-01-04: 完成 5-11 聊天 UI 拆分、Iceberg 模式、流式渲染与自动滚动实现；补充单测与持久化兼容。
- 2026-01-05: 清理仓库 lint 错误并确保 `npm run lint` 通过。
