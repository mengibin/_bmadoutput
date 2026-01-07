# Story 5-12: Interactive Embedded Widgets (Forms & Selection)

## Overview
**Epic**: 5 – Observability, Settings & Recovery
**Priority**: High
**Status**: `review`

## Goal
Enable rich, structured interaction between the User and the Agent within the chat interface. Instead of relying solely on free-text, the Agent can request structured inputs (forms, selections, approvals) via embedded widgets, and the UI provides a user-friendly way to provide that data.

## Business Value
- **Precision**: Reduces ambiguity in user responses. When an Agent asks "Which files should I edit?", a checklist is far superior to a text description.
- **Efficiency**: Speeds up human-in-the-loop workflows (e.g. approving a plan).
- **Agentic Capability**: Unlocks advanced patterns like "Plan Review", "Parameter Collection", and "Clarification".

## Acceptance Criteria

### 0. In-Chat Embedding (NEW)
- **Given** the agent triggers a widget request
- **When** the chat renders the response
- **Then** the widget appears **inline within the chat message stream** (embedded in the chat bubble area), not in a separate panel or modal.

### 1. Widget Protocol (Data Structure)
- **Trigger**: The Widget is triggered by a specific event from the Agent.
    - *Technical impl*: Likely a specific Tool Call (e.g., `ui.ask_selection`, `ui.ask_form`) OR a structured message content block.
- **Payload**: The request must include:
    - `type`: "form" | "selection" | "confirmation" | "plan_review" | "feedback" | "action_selection"
    - `message`: Instructions for the user (e.g., "Please select the files to process")
    - `schema/options`: The data definition (JSON Schema for forms, List of items for selection).
    - `widgetId`: Unique ID to track the interaction.

### 2. Supported Widgets
- **Form Widget**:
    - Input: JSON Schema (subset: string, number, boolean, enum).
    - UI: Renders fields with validation.
    - Output: JSON object matching schema.
- **Selection Widget**:
    - Input: List of options `{ label: string, value: string, description?: string }`, `multi: boolean`.
    - UI: Checkbox list (multi) or Radio list (single).
    - Output: Array of selected values.
- **Action Proposal Widget (Plan Review)**:
    - Input: A Markdown plan or list of steps.
    - UI: Display the plan with "Approve" (Green) and "Request Changes" (Red) buttons.
        - "Request Changes" opens a text input for feedback.
- **Multi-Item Feedback Widget** (NEW):
    - Input: List of items `{ id: string, content: string }`.
    - UI: Each item displayed with a **text input field** for user feedback.
    - Output: `{ [itemId]: string }` — feedback text for each item.
    - Use Case: LLM 列出多个问题/建议，用户对每一条单独输入反馈内容。
- **Action Selection Widget** (NEW):
    - Input: List of actions `{ id: string, label: string, description?: string }`.
    - UI: Radio list of actions + "Continue" button (primary action).
    - Output: Selected action ID.
    - Use Case: LLM 提供多个可选的下一步操作（如执行不同 workflow），用户选择一个并点击继续。
- **Confirmation Widget**:
    - Input: a yes/no confirmation request.
    - UI: Confirm / Cancel buttons.
    - Output: `{ confirmed: boolean }`.

### 3. Interaction Flow
- **Active State**: When first rendered, the widget is interactive.
- **Submission**:
    - User completes interaction and clicks "Submit" / "Send".
    - Data is sent to Runtime (as a Tool Result or User Message).
- **Submitted State**:
    - Once submitted, the widget transforms into a "Read-Only" state.
    - Visual indication (e.g., grayed out, "Submitted" badge).
    - Prevents re-submission of the same request.

## Technical Components / Changes
1.  **`EmbeddedWidgetRegistry`**: A component map that renders the correct widget based on `type`.
2.  **`WidgetContainer`**: Wrapper component in the Message List handling the "Active" vs "Submitted" state.
3.  **Specific Components**:
    - `FormWidget.tsx` (using `react-hook-form` or similar simple JSON schema mapper).
    - `SelectionWidget.tsx`.
    - `PlanReviewWidget.tsx`.
4.  **System Prompt Guidance (NEW)**:
    - Add a concise rule to the system prompt: when structured input is needed, the assistant **must** call `ui.ask_user` (not simulate forms in plain text).
    - Tool schema remains the source of truth; runtime validates payloads.

## Dependencies
- **Story 5.11**: Requires the Chat UI components (`MessageList`, `MessageItem`) to host these widgets.
- **Story 5.8**: Message persistence must support widget state (`widgetId`, `submitted`).

## Technical Context

### Message Extension for Widgets
Extend `ConversationMessage` (from Story 5-11) to support widget payloads:
```typescript
interface WidgetPayload {
  widgetId: string
  type: 'form' | 'selection' | 'confirmation' | 'plan_review' | 'feedback' | 'action_selection'
  message: string
  schema?: object           // JSON Schema for forms
  options?: WidgetOption[]  // for selection
  actions?: WidgetAction[]  // for action_selection
  planContent?: string      // Markdown for plan_review
  items?: Array<{ id: string, content: string }> // for feedback
  submitted?: boolean
  submittedValue?: unknown  // stored result after submission
}

interface ConversationMessage {
  // ... existing fields from Story 5-11
  widget?: WidgetPayload    // NEW: if present, render as widget
}
```

### IPC Events
| Event | Direction | Description |
|-------|-----------|-------------|
| `widget:request` | Main → Renderer | (Future) Agent requests a widget (via ToolCall result) |
| `widget:submit` | Renderer → Main | (Future) User submits widget data |

**Implementation Note (this iteration)**:
- Trigger mechanism uses a structured assistant message block: ` ```widget\n{...}\n``` `.
- Submission is sent back to Runtime as a **user message** with a `WIDGET_SUBMIT` envelope (JSON payload), and the widget message is persisted via `updateConversationMessage()`.

## Related Artifacts
- Story: `_bmad-output/implementation-artifacts/5-11-enhanced-chat-interface.md`
- Tech Spec: `tech-spec-5-12-interactive-embedded-widgets.md`

## Verification Plan

### Manual Verification
1.  **Selection**: Mock a `ui.ask_selection` request. Verify checkbox list appears. Select items -> Submit. Verify output JSON.
2.  **Form**: Mock a `ui.ask_form` request. Verify form fields. Test validation (required fields). Submit -> Verify output.
3.  **Plan Review**: Mock a plan review widget. Verify Markdown plan renders. Click "Approve" -> Verify submission. Click "Request Changes" -> Verify feedback input appears.
4.  **Read-Only**: After submission, try to change values. Verify actions are disabled.
5.  **Persistence**: Reload page (if persistence is implemented) or scroll up/down. Verify widget state remains consistent.

### Automated Tests
- Component tests for each Widget type.
- Interaction tests: active -> submit -> read-only transition.

## Tasks / Subtasks
- [x] 1. 扩展消息模型与持久化（支持 `widget` 与提交态）
  - [x] 1.1 扩展 `ConversationMessage.widget` 类型（`WidgetPayload` + `WidgetType`）
  - [x] 1.2 增加消息更新持久化：`updateConversationMessage()`（Renderer Store + IPC + RuntimeStore）
- [x] 2. 实现 Widget 组件体系（Registry + Container + 各 widget）
  - [x] 2.1 `EmbeddedWidgetRegistry`：按 `type` 渲染具体组件
  - [x] 2.2 `WidgetContainer`：处理 Active vs Submitted（只读 + Badge）
  - [x] 2.3 `FormWidget`（JSON Schema 子集）、`SelectionWidget`、`PlanReviewWidget`
  - [x] 2.4 新增：`FeedbackWidget`、`ActionSelectionWidget`、`ConfirmationWidget`
- [x] 3. 集成到聊天 UI（RunWorkspace / WorksPage）
  - [x] 3.1 `MessageItem` 渲染 `message.widget`
  - [x] 3.2 `MessageList`/`ChatPanel` 透传 `onWidgetSubmit` + `widgetDisabled`
  - [x] 3.3 WorksPage：解析 ` ```widget` 块并渲染；提交后更新消息并继续运行/对话
  - [x] 3.4 RunWorkspace：支持解析 widget 并提交（含流式结束后拆分 widget 块）
- [x] 4. 测试与验证
  - [x] 4.1 单测：widget parser 与主要 widget 渲染
  - [x] 4.2 单测：RuntimeStore `updateConversationMessage()` 持久化更新
- [x] 5. System Prompt Guidance（NEW）
  - [x] 5.1 增加 `ui.ask_user` tool schema + Runtime 校验（Tool schema 为 source of truth）
  - [x] 5.2 更新 system base rules：需要结构化输入时必须使用 `ui.ask_user`，并输出 ` ```widget` 块

## Dev Agent Record

### Implementation Plan
- 使用结构化消息块 ` ```widget\n{...}\n``` ` 作为前端触发协议，避免本轮引入新的后端 ToolCall/IPC 协议。
- 扩展 `ConversationMessage` 并实现消息更新持久化，用于保存 `submitted/submittedValue`。
- 实现 Widget 组件集与消息列表渲染集成；提交后将结果包装为 `WIDGET_SUBMIT` 用户消息继续对话/运行。
- 增加 `ui.ask_user` tool（校验并返回 `widgetBlock`），并在 system prompt 中强制使用该工具（禁止纯文本模拟表单）。
- 补充 parser/UI/persistence 单测。

### Debug Log
- `npm test` (crewagent-runtime) — 通过。
- `npm run lint` (crewagent-runtime) — 通过（仍有 TypeScript 版本提示）。

### Completion Notes
- 支持 6 类 Widget：`form`、`selection`、`plan_review`、`feedback`、`action_selection`、`confirmation`。
- Widget 可在 WorksPage/RunWorkspace 的消息流中渲染，并在提交后锁定为只读态（持久化到 `messages.json`）。
- 运行态回传以 `WIDGET_SUBMIT` 用户消息形式发送给 Runtime，便于 LLM 读取结构化结果（后端 ToolCall 等待机制可后续补齐）。
- 新增 `ui.ask_user` tool：Runtime 校验 widget payload 并返回 `widgetBlock`，system prompt 强制模型使用该工具来触发结构化输入。

## File List
- crewagent-runtime/src/stores/appStore.ts
- crewagent-runtime/src/hooks/useConversationWorkspace.ts
- crewagent-runtime/src/pages/WorksPage/WorksPage.tsx
- crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx
- crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx
- crewagent-runtime/src/pages/RunsPage/RunsPage.css
- crewagent-runtime/src/pages/RunsPage/components/ChatPanel.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageList.tsx
- crewagent-runtime/src/pages/RunsPage/components/MessageItem.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/EmbeddedWidgetRegistry.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/WidgetContainer.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/FormWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/SelectionWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/PlanReviewWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/FeedbackWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/ActionSelectionWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/ConfirmationWidget.tsx
- crewagent-runtime/src/pages/RunsPage/components/widgets/widgetMessageParser.ts
- crewagent-runtime/src/pages/RunsPage/components/widgets/widgetSubmission.ts
- crewagent-runtime/src/pages/RunsPage/components/widgets/widgetMessageParser.test.ts
- crewagent-runtime/src/pages/RunsPage/components/widgets/widgets.test.tsx
- crewagent-runtime/electron/main.ts
- crewagent-runtime/electron/preload.ts
- crewagent-runtime/electron/electron-env.d.ts
- crewagent-runtime/electron/services/toolHost.ts
- crewagent-runtime/electron/services/fileSystemToolHost.ts
- crewagent-runtime/electron/services/fileSystemToolHost.test.ts
- crewagent-runtime/electron/services/prompt-templates/system-base-rules.md
- crewagent-runtime/electron/services/prompt-templates/system-base-rules-chat.md
- crewagent-runtime/electron/stores/runtimeStore.ts
- crewagent-runtime/electron/stores/runtimeStore.test.ts

## Change Log
- 2026-01-05: 完成 5-12 Widget 协议（结构化消息块）、UI 组件、提交持久化与 Works/Run 集成；补充单测。
- 2026-01-05: 增加 `ui.ask_user` tool schema + Runtime 校验，并更新 system prompt 规则以强制使用结构化 widget 交互。
