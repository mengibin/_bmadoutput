# Runtime Development Roadmap: UI-First Approach

This document outlines the development sequence for the CrewAgent Runtime environment, following the **"Interface First, Backend Last"** principle. The goal is to establish a functional visual shell immediately, ensuring user interaction models are validated before implementing heavy execution logic.

## Principles
1.  **UI First**: Prioritize building the visual structure (Shell, Navigation, Views).
2.  **Mock Integration**: Use mock data in UI components initially to validate UX flow.
3.  **Backend Integration**: Connect existing backend logic (e.g., `RuntimeStore` logic from Story 4.2/4.3) only after the corresponding specific UI capability is ready.
4.  **Execution Last**: The complex "Agent Loop" (LLM calls, tool execution) is implemented last, once the control surface is complete.

---

## Phase 1: UI Foundation & Configuration
*Goal: A running application that looks complete, even if it doesn't "run" workflows yet.*

### 1. Story 5-0: Runtime UI Shell and Navigation
- **Scope**: Start/Settings shell, Project context (Project header + Files/Works), slide-out panels, fixed Conversation tab, blocking package error overlay.
- **Why**: Establishes the application skeleton.
- **Status**: *Done*

### 2. Story 5-10: Files Page (Project Explorer)
- **Scope**: Project-only file tree, file preview/edit in tabs (after fixed Conversation), Open in OS, basic file operations (default to `@project/artifacts/` when present).
- **Why**: Essential for users to verify project files and generated artifacts.
- **Dependencies**: Basic IPC for file system (partly ready).
- **Status**: *Done*

### 3. Story 5-5 & 5-6: LLM & Settings Configuration
- **Scope**: Settings UI for package info/cache, LLM provider config (model/endpoint/API key/test), and theme.
- **Why**: Independent module; "easy win" to provide necessary configuration capabilities early.
- **Dependencies**: `RuntimeStore` settings methods (Ready).

---

## Phase 2: Visualization & Read-Only Integration
*Goal: The user can see the "Static" state of a workflow (Graph, Current Step).*

### 4. Story 5-1: View Current Workflow State (Graph UI)
- **Scope**:
    - **Graph View**: Visual rendering of nodes and edges (React Flow).
    - **Pre-Start Preview**: Shown before starting a run so users understand the flow.
    - **No Step Detail Exposure**: Do not surface per-step Markdown content/details in this phase.
- **Integration**:
    - Use **Package workflow.graph.json** to populate the graph.
    - *Future*: After **Story 4-8 / 4-3**, overlay run progress and highlight `currentNodeId`.
- **Note**: This is the "Monitor" of the system.
- **Status**: *Done*

### 5. Story 4-11: Agent Session Menu (The "Start" Button)
- **Scope**: UI for the "Agent" tab or "Chat" panel start screen.
- **Why**: Provides the entry point for user interaction (e.g., "Start Auto-Pilot").

---

## Phase 3: Interaction UI (The Console)
*Goal: The user can see *dynamic* changes and history.*

### 6. Story 5-8: Persist Conversations & Messages
- **Scope**: Persist conversation metadata + message history per project in RuntimeStore (load on reopen, append on new messages).
- **Why**: Makes **Works** durable and allows continuing past chats.
- **Status**: *Done*

### 7. Story 5-2: View Execution Log
- **Scope**: A scrolling log panel (terminal-like or structured list).
- **Why**: Users need feedback when the Execution Core starts running.
- **Mocking**: Initially populate with static dummy logs to style the component.

### 8. Story 5-3: Workflow Progress with Real Data
- **Scope**: Connect Progress UI to real `@state/workflow.md` data (currentNodeId, stepsCompleted).
- **Why**: Users need to see actual execution progress, not mock data.

---

## Phase 4: Execution Core (The Brain)
*Goal: Make it actually work. Inject the "Ghost" into the machine.*

*Note: These are Backend-heavy stories.*

### 9. Story 4-4 to 4-7: The Agent Loop
- **Components**:
    - Prompt Composition (Template + Context).
    - LLM API Client (OpenAI/Anthropic/Local).
    - Tool Execution Sandbox (File System ops).
    - State Transition Logic (Updating `workflow.md`).
- **Validation**:
    - Use Phase 2 (Graph UI) and Phase 3 (Logs) to verify execution progress in real-time.

### 10. Story 4-8: Run Folder & Run-Scoped State
- **Scope**: Create `RuntimeStoreRoot/projects/<projectId>/runs/<runId>/state` and map `@state` to the run; persist run metadata in a project-scoped `runsIndex`.
- **Why**: Required for accurate Resume and per-run isolation.

### 11. Story 4-9: Project Persistence (Artifacts)
- **Scope**: Ensure execution artifacts are saved to the user's project folder correctly and appended into `@state/workflow.md`.

---

## Phase 5: Advanced Features
*Goal: Production readiness.*

### 12. Story 4-10: MCP Tool Support
- **Scope**: Advanced tool drivers for MCP servers.

### 13. Story 5-4 & 5-7: Advanced Configuration
- **Scope**: Tool timeouts, MCP server management UI.

---

## Summary of Immediate Next Steps
1. **Create/Design** Story 4-8 (Run Folder & Run-Scoped State) - *Enables per-run isolation and resume correctness*.
2. **Design/Implement** Story 4-4 (Prompt Composition) - *Required before Story 4-5 (LLM calls) can run safely*.
