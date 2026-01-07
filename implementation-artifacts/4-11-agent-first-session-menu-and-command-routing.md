# Story 4.11: Agent-First Session (Menu) & Command Routing Contract

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Consumer**,
I want to start from an Agent Session (menu-driven) and route my input deterministically,
So that Runtime behaves like BMAD/Cursor: “choose agent → show menu → user input triggers workflow/exec/action”.

## Acceptance Criteria

1. **Agent Session Entry**
   - **Given** I select an agent in the UI
   - **Then** the UI enters "Agent Session" mode (`activeAgentId` is set, and any active workflow run is cleared / `activeRunId` is empty).
   - **And** the Agent's Persona and Menu are displayed as a numbered list (1-based indices) consistent with `_bmad-output/tech-spec/agent-menu-command-contract.md`.

2. **Command Routing (Deterministic)**
   - **Given** I type input in the Agent Session
   - **Then** Runtime resolves it to a `ResolvedCommand` per `_bmad-output/tech-spec/agent-menu-command-contract.md`:
     - **Menu Visibility**: Only consider visible menu items after applying `ide-only` / `web-only` gating for the current surface.
     - **Empty Input**: If `input.trim()` is empty, return `ShowMenu`.
     - **Numeric Selection**: Matches `^[0-9]+$` to a **1-based** menu index.
       - If out-of-range, return `ClarifyChoice` with valid range/options.
     - **Exact/Fuzzy Trigger**: Matches `trigger`, optional `cmd` (if present), `triggers[].match` (alias/handler), and `description` (with fuzzy fallbacks).
     - **Multi-Handler**: Supports expanded candidates from `triggers: [{type: 'handler'}]`.
     - **Clarification**: Returns `ClarifyChoice` if multiple candidates match with similar scores.
       - UI must render the candidate list with indices; next numeric input selects by index.
     - **Fallback**: Returns `Chat` if no command matches.

3. **Command Execution**
   - **StartWorkflow**: Calls `RunManager.createRun` (Story 4.8) and transitions to `RunMode`.
   - **ExecScript**: Routes per contract (`exec` → `ExecScript`, and if `exec` points to `workflow.md` it is upgraded to `StartWorkflow`).
     - If `workflow/exec` points to classic `workflow.yaml/.yml` or `.xml`, return a structured error `NotSupportedClassicWorkflow` (do not silently proceed).
   - **RunAction**:
     - `menu.show` -> Returns `ShowMenu` event to UI.
     - `agent.dismiss` -> Clears `activeAgentId`, exits session.
     - `run.resume` -> Resumes the most recent run for the current `projectRoot` (or a specified `runId`, if supported).
     - `promptId` (e.g., `action="#greet"`) -> Injects prompt content as user message.
   - **Chat**: Sends input to `LLMAdapter` using the Agent's Persona (no workflow state; tool visibility should respect agent tool policy).
   - **Data Attachment**: If the matched menu item has `data`, include it as `dataRef` in the resolved command and load/parse it before dispatch (per contract).

4. **Chat Mode Integration**
   - **Given** the resolution is `Chat` or `StartWorkflow` (running)
   - **When** I send a message
   - **Then** it triggers the backend LLM loop (`AgentChat`/`LLMAdapter` for `Chat`; `ExecutionEngine` for active runs).
   - IPC must bridge UI `handleSend` to backend:
     - `chat:send` for Agent/Chat mode (no active run).
     - `runs:continue` (or equivalent) for RunMode (active run input).

5. **Input Mode Distinction**
   - **AgentMode** (No active run): Input routing prioritizes Menu > Chat.
   - **RunMode** (Active run): Input goes to ExecutionEngine *unless* it matches a Global Command (e.g., `/menu`, `/stop`, `*menu`, `*dismiss`).

## Technical Context

- **Contract**: `_bmad-output/tech-spec/agent-menu-command-contract.md` (Strict adherence required).
- **New Service**: `crewagent-runtime/electron/services/commandRouter.ts`.
- **Integration**:
  - `electron/main.ts`: Add IPC handlers (`agent:resolveCommand`, `agent:dispatch`, `chat:send`, `runs:continue`).
  - `ExecutionEngine`: Needs to handle "Chat" (stateless/session-state) vs "Workflow" (state-driven).
  - `RunManager`: For `StartWorkflow`.

## Design

### Summary

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-11-agent-first-session-menu-and-command-routing.md`
- Implement BMAD/Cursor-style Agent Session routing: menu → deterministic command resolution → dispatch.
- Add IPC bridges for Agent/Chat mode and Run mode: `agent:resolveCommand`, `agent:dispatch`, `chat:send`, `runs:continue`.
- Electron-only MVP: hardcode `targetSurface='electron'` and apply `ide-only/web-only` gating.

### UX / UI

- **Agent Session (WorksPage, AgentMode)**:
  - Render visible menu items as a **1-based numbered list**.
  - On send: resolve → dispatch. UI handles:
    - `SHOW_MENU`: re-render menu.
    - `CLARIFY_CHOICE`: show candidate list; next numeric input selects by index.
    - `RUN_STARTED/RUN_RESUMED`: set `activeRun` and transition to RunMode (`/runs` or equivalent).
    - `CHAT_RESPONSE`: append assistant message to the conversation.
    - `AGENT_DISMISSED`: clear `activeAgentId` and exit session.
- **RunMode (RunsPage / activeRun input)**:
  - Send user input via `runs:continue`; append `assistantText` as assistant message (or show in run chat panel).

### API / Contracts

- Contract: `_bmad-output/tech-spec/agent-menu-command-contract.md` (source of truth).
- IPC channels return `{ success: boolean, ... }` and never throw across the IPC boundary.
- IPC:
  - `agent:resolveCommand` → `ResolvedCommand` (numeric/trigger/alias/handler/fuzzy/clarify/chat).
  - `agent:dispatch` → UI event (starts run, resumes run, chat response, dismiss, show menu).
  - `chat:send` → non-streaming assistant response (MVP).
  - `runs:continue` → `EngineResult` (ExecutionEngine).

```ts
// Renderer -> Main
ipcRenderer.invoke('agent:resolveCommand', {
  projectRoot: string;
  packageId: string;
  agentId: string;
  input: string;
  context: { activeRunId?: string };
})
// Returns: AgentResolveCommandResult

ipcRenderer.invoke('agent:dispatch', {
  projectRoot: string;
  packageId: string;
  agentId: string;
  command: ResolvedCommand;
  conversationId?: string;
})
// Returns: AgentDispatchResult

ipcRenderer.invoke('chat:send', {
  projectRoot: string;
  packageId: string;
  agentId: string;
  conversationId?: string;
  input: string;
})
// Returns: ChatSendResult (MVP: non-streaming)

ipcRenderer.invoke('runs:continue', {
  projectRoot: string;
  packageId: string;
  workflowId: string;
  runId: string;
  activeAgentId: string;
  userInput: string;
})
// Returns: RunsContinueResult
```

- **Return shapes & error codes (recommended)**:

```ts
type IpcOk<T extends Record<string, unknown>> = { success: true } & T
type IpcErr<Code extends string> = { success: false; error: { code: Code; message: string; details?: unknown } }

type AgentSessionErrorCode = 'PROJECT_ROOT_INVALID' | 'PACKAGE_NOT_FOUND' | 'AGENT_NOT_FOUND' | 'UNKNOWN'

type ContractCommandErrorCode =
  | 'UNKNOWN_WORKFLOW'
  | 'NOT_SUPPORTED_CLASSIC_WORKFLOW'
  | 'UNKNOWN_PROMPT_ID'
  | 'DATA_LOAD_FAILED'
  | 'PERMISSION_DENIED'
  | 'VALIDATION_FAILED'

type RunManagerErrorCode =
  | 'PROJECT_ROOT_INVALID'
  | 'PACKAGE_NOT_FOUND'
  | 'MANIFEST_INVALID'
  | 'WORKFLOW_REF_REQUIRED'
  | 'WORKFLOW_REF_UNKNOWN'
  | 'WORKFLOW_REF_MISMATCH'
  | 'WORKFLOW_FILE_NOT_FOUND'
  | 'GRAPH_FILE_NOT_FOUND'
  | 'AGENT_NOT_FOUND'
  | 'STATE_INIT_FAILED'
  | 'RUNS_INDEX_INVALID'
  | 'RUNS_INDEX_WRITE_FAILED'
  | 'UNKNOWN'

type LlmErrorCode =
  | 'LLM_AUTH_FAILED'
  | 'LLM_TIMEOUT'
  | 'LLM_RATE_LIMITED'
  | 'LLM_HTTP_ERROR'
  | 'LLM_BAD_RESPONSE'
  | 'UNKNOWN'

type EngineErrorCode =
  | 'ENGINE_ABORTED'
  | 'ENGINE_MAX_TURNS_EXCEEDED'
  | 'ENGINE_LOOP_DETECTED'
  | 'ENGINE_PRECONDITION_FAILED'
  | 'LLM_ERROR'
  | 'TOOL_EXEC_FAILED'
  | 'UNKNOWN'

type AgentResolveCommandResult = IpcOk<{ command: ResolvedCommand }> | IpcErr<AgentSessionErrorCode>

type AgentDispatchEvent =
  | { type: 'SHOW_MENU' }
  | { type: 'CLARIFY_CHOICE'; command: Extract<ResolvedCommand, { kind: 'ClarifyChoice' }> }
  | { type: 'RUN_STARTED'; run: RunMetadata }
  | { type: 'RUN_RESUMED'; run: RunMetadata }
  | { type: 'AGENT_DISMISSED' }
  | { type: 'CHAT_RESPONSE'; assistant: { role: 'assistant'; content: string | null }; usage?: unknown }

type AgentDispatchResult =
  | IpcOk<{ event: AgentDispatchEvent }>
  | IpcErr<AgentSessionErrorCode | ContractCommandErrorCode | RunManagerErrorCode | LlmErrorCode | EngineErrorCode>

type ChatSendResult =
  | IpcOk<{ assistant: { role: 'assistant'; content: string | null }; usage?: unknown }>
  | IpcErr<AgentSessionErrorCode | ContractCommandErrorCode | LlmErrorCode>

type RunsContinueResult = {
  success: boolean
  phase: 'Running' | 'WaitingUser' | 'Completed' | 'Failed'
  assistantText?: string | null
  error?: { code: EngineErrorCode; message: string; details?: unknown }
}
```

### Data / Storage

- No new on-disk schema.
- Conversation messages remain persisted via existing conversation persistence (Story 5.8). UI appends:
  - user message before dispatch,
  - assistant message on `CHAT_RESPONSE` / `chat:send` / `runs:continue`.
- Run state/logs remain managed by RuntimeStore + RunManager (Story 4.8) + ToolHost logs under `@state/logs/`.
- ClarifyChoice does not require cross-turn server state in MVP (indices refer to visible menu item indices).

### Errors / Edge Cases

- `web-only` items are hidden in Electron (`targetSurface='electron'`); `ide-only` stays visible.
- Empty input → `ShowMenu`.
- Numeric selection out of range → `ClarifyChoice` (includes valid range/options).
- Unsupported classic refs (`.yaml/.yml/.xml`) → `NOT_SUPPORTED_CLASSIC_WORKFLOW`.
- Unknown workflow/prompt/data path failures → structured error (`UNKNOWN_WORKFLOW`, `UNKNOWN_PROMPT_ID`, `DATA_LOAD_FAILED`).
- LLM/Engine failures propagate as structured error codes; UI shows message and does not mutate `activeRun` on failure.

### Test Plan

- Unit (vitest): `commandRouter.test.ts`
  - Empty input → `ShowMenu`.
  - Numeric selection in/out-of-range.
  - trigger/cmd/alias/handler matching + fuzzy scoring preference.
  - `ide-only/web-only` gating affects candidates and numeric indexing.
  - Ambiguous match → `ClarifyChoice`.
- Integration (vitest): dispatch smoke
  - `StartWorkflow` calls `RunManager.createRun` and returns `RUN_STARTED`.
  - `run.resume` resolves latest run and returns `RUN_RESUMED`.
  - `chat:send` returns assistant message and LLM error codes.

## Tasks / Subtasks

- [ ] **M1: Resolve Only (CommandRouter)**
  - [ ] Implement `CommandRouter` service logic (Exact/Fuzzy/Numeric matching).
  - [ ] Add `ipcMain` handler `agent:resolveCommand` (Main Process).
  - [ ] Update `WorksPage.tsx`: Call `agent:resolveCommand` instead of appending text directly.
  - [ ] Verify: Clicking menu items logs resolved command in Console/UI.
- [ ] **M2: Dispatch & Smoke Test**
  - [ ] Implement `agent:dispatch` (or `resolveAndDispatch`) in Main.
  - [ ] Connect `StartWorkflow` to `RunManager.createRun`.
  - [ ] Connect `Chat` to `LLMAdapter` (basic echo or fast response).
  - [ ] Verify: "Start Workflow" triggers run creation; "Chat" returns response.
- [ ] **M3: Full Integration**
  - [ ] Connect `ExecutionEngine` loop.
  - [ ] Implement `ipcMain.handle('runs:continue')` to call `ExecutionEngine.start/continue` for RunMode input.
  - [ ] Finalize `ipcMain.handle('chat:send')` for Agent/Chat mode.


## Verification

- **Unit Test**: `commandRouter.test.ts`
  - Test empty input → `ShowMenu`.
  - Test numeric selection (in-range) → exact match.
  - Test numeric selection (out-of-range) → `ClarifyChoice` with candidates.
  - Test trigger/cmd matching (case-insensitive, leading `*` ignored for matching).
  - Test fuzzy matching preference (trigger/cmd beats description).
  - Test `ide-only` / `web-only` gating affects candidates and numeric indexing.
  - Test classic workflow refs (`.yaml/.yml/.xml`) return `NotSupportedClassicWorkflow` on dispatch.
- **Manual Verification**:
  - Select Agent -> Type "1" -> Verifies correct menu item triggered.
  - Type "hello" -> Verifies Chat fallback response.
  - Type "start workflow" (trigger) -> Verifies Run creation.

## References

- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-11-agent-first-session-menu-and-command-routing.md`
- Contract: `_bmad-output/tech-spec/agent-menu-command-contract.md`
