# Story 5.0: Runtime UI Shell & Navigation (IA First)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Consumer**,  
I want a consistent Runtime application shell (navigation + core layout),  
so that all later features (import/run/progress/log/settings) fit into a coherent UI and don’t require rework.

## Acceptance Criteria

1. **Given** I open the Runtime app  
   **When** no Project is selected yet  
   **Then** the left sidebar shows **Start** and **Settings** only (Settings pinned at the bottom)  
   **And** Start lets me **New Project** or **Open Project**  
   **And** New Project creates `ProjectRoot/artifacts/` and `ProjectRoot/.crewagent.json`

2. **Given** I open a Project  
   **When** the Project loads  
   **Then** the left sidebar shows the **Project name/icon**  
   **And** below it shows **Files** and **Works**  
   **And** Start is no longer visible while a project is active

3. **Given** I open a Project  
   **When** the Project has no `activePackageId` in `.crewagent.json` **or** the referenced package is missing locally  
   **Then** the Runtime shows a **blocking error overlay**  
   **And** it prevents entering **Works/Execution** until the package is re-imported

4. **Given** I open **Files**  
   **When** I click a file  
   **Then** I can view and edit the file in a **file tab**  
   **And** I can save changes back to `@project`  
   **And** file viewing is **extensible** (future file types via plugins)

5. **Given** I open **Works**  
   **When** I create a conversation  
   **Then** I can choose entry type: **Agent / Workflow / Chat**  
   **And** within a conversation I can **switch type**  
   **And** switching type keeps the same conversation and history (no new conversation created)  
   **And** the UI updates to show type-specific controls (Agent selector / Workflow selector / Chat input)  
   **And** the active type is clearly indicated and persisted for the conversation  
   **And** I can see how many workflows were started from that conversation  
   **And** I can select a workflow to continue its execution

6. **Given** I open **Settings**  
   **When** I configure Runtime  
   **Then** I can manage:
   - **Package info & cache** (active package summary, list/remove/clear, re-validate, import `.bmad`)
   - **LLM Provider** (OpenAI / OpenAI-compatible like DeepSeek / Azure / Ollama), model, endpoint, API key, test connection
   - **Theme** (system/light/dark; optional high-contrast)

7. **Given** I trigger an execution-related action inside a conversation  
   **When** the UI updates  
   **Then** it reflects canonical state derived from `@state/workflow.md` + `@pkg/workflow.graph.json` (not from chat memory)

8. **Given** I am inside a Project  
   **When** I click **Files** or **Works** in the left nav  
   **Then** a **slide-out panel** opens from the left  
   **And** the main workspace stays visible on the right as **tabs**  
   **And** the **first tab is fixed** as **Conversation/Chat** (cannot be closed)  
   **And** opening files creates **additional tabs** for file viewing/editing (multiple tabs allowed)

9. **Given** I have a conversation in the Works list  
   **When** I delete it  
   **Then** the UI asks for confirmation  
   **And** the conversation is removed from the list  
   **And** its persisted data is removed from RuntimeStore

## Design Notes (Guidance)

- Primary references:
  - UX spec: `_bmad-output/ux-design-specification.md`
  - Runtime entrypoints: `_bmad-output/architecture/entrypoints-agent-vs-workflow.md`
  - Runtime architecture: `_bmad-output/architecture/runtime-architecture.md`
  - ToolCalls protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- MVP should prioritize “information architecture + navigation + state visibility” over visual polish.

## Design

### Summary

- This story defines **Runtime’s UI shell + information architecture (IA)** so that Epic 4 (execution) and Epic 5 (observability/settings) features land consistently.
- The UI must make the “Project / Package / Conversation / Run” mental model explicit and keep **state visible** (Document-as-State) while preserving **Project-First + Private RuntimeStore** constraints.
- When Project ↔ Package binding is invalid, the UI must show a **blocking error overlay** with clear recovery actions (re-import package / return to Start).
- Initial state shows **Start + Settings only**; once inside a Project, the left nav pivots to **Project header + Files + Works** with Settings pinned at bottom.
- Package information is surfaced in **Settings** (no standalone Package page).
- **Works** is the primary execution entry: conversations can start via Agent/Workflow/Chat and switch types mid-stream.
- **Conversation type switching** is explicit in the Conversation tab header (segmented control). Switching updates the **activeType** of the same conversation and swaps the controls (Agent selector / Workflow selector / Chat input) without creating a new conversation.
- **Main workspace** is a **tabbed surface**: the **Conversation/Chat** tab is fixed first, and **file tabs** open after it (multiple allowed). **Files/Works** open as **slide-out panels** without replacing the workspace.

### Navigation Model (IA)

**Top-level pages (left sidebar / nav rail):**

- **Start** (only when no project selected): New/Open Project, Recent Projects
- **Settings** (always pinned at bottom): Package info/import/cache, LLM Provider, Theme

**Project Context (shown after a Project is opened):**

- **Project Header** (icon + project name)
- **Files** (Project file tree + previews) → **slide-out panel**
- **Works** (Conversation history + entry creation) → **slide-out panel**

**Workspace (right side, persistent):**

- **Conversation/Chat** tab (fixed first tab; create/switch type, workflow runs, resume)
  - Header includes **Agent / Workflow / Chat** segmented control; switching updates the activeType and the visible controls without changing the conversation.
- **File Viewer/Edit** tabs (open per file; multiple tabs supported; save writes to `@project`)

**Constraints:**
- **1-Project-1-Package**: Each project is bound to a single active package.  
  - If `.crewagent.json` has no `activePackageId`, or the referenced package is missing locally, **show a blocking error** and prevent entry into Works/Execution.
- **Settings placement**: Settings is always visible at the bottom of the sidebar in both global and project contexts.

**Global status bar (always visible):**
- Active Project (name/path), Package (name/version), Run (runId + phase)
- Primary controls: Import Package, Open Project, Settings
- Run status indicator (phase + runId)

### Blocking Error States (Package Binding)

**Trigger Conditions (either):**
- `.crewagent.json` missing `activePackageId`
- `activePackageId` exists but package cache does not exist locally

**UX Response:**
- Show a **blocking overlay** (centered card, red/error tone, overlay backdrop) on:
  - Project context (Files/Works area)
  - Works conversation view
- Disable navigation into Works/Execution until resolved
- Provide actions:
  - **Re-import Package** (primary)
  - **Return to Start** (secondary)

### Key Flows

#### Flow A — New/Open Project

1) User clicks **New Project** → choose parent folder + project name → create folder  
2) Runtime ensures `ProjectRoot/artifacts/` exists (Project-First)  
3) Runtime creates a project-level config file (non-secret) at `ProjectRoot/.crewagent.json`  
4) Runtime stores project in **recent projects** (in RuntimeStore)
5) On **Open Project**, if `.crewagent.json` has no `activePackageId` or the package is missing locally → show a **blocking error** and require re-import before entering Works.

> “New Project” creates only user-auditable files in ProjectRoot: `artifacts/` + `.crewagent.json`.  
> Secrets (LLM API keys), local caches, and run state/logs remain in RuntimeStore.

**Suggested `.crewagent.json` (v1):**

```json
{
  "schemaVersion": "1",
  "projectName": "My Project",
  "artifactsDir": "artifacts/",
  "createdAt": "2025-12-28T00:00:00.000Z"
}
```

#### Flow B — Import/Open `.bmad` Package (in Settings)

1) User opens **Settings → Packages**  
2) Click **Import Package** → selects `.bmad` zip  
3) Runtime validates (schema + referenced files), then extracts to RuntimeStore cache and assigns `packageId`  
4) Settings shows:
   - validation result (pass/fail + actionable errors with file path)
   - active package summary (`bmad.json` name/version; workflows/agents counts)
   - cache list (remove/clear)

#### Flow C — Works: Create Conversation (entry selection)

1) User opens **Works**  
2) Click **New Conversation**  
3) Choose entry type:
   - **Agent** (select agent → menu view)
   - **Workflow** (select workflow → start run)
   - **Chat** (lightweight conversation without workflow)
4) Conversation appears in history list with type + timestamps

#### Flow D — Works: Switch Type & Continue Workflows

1) Inside a conversation, user can **switch type** (Agent ↔ Workflow ↔ Chat)  
2) Conversation shows **workflows started count**  
3) User can pick a workflow run to continue execution  
   - Switching to a run shows **run details** (right side) without replacing conversation history/messages

**Type Switch UI Details (explicit)**

- Provide a **segmented control** in the conversation header: **Agent / Workflow / Chat**.
- Switching types **does not** create a new conversation; it only updates `activeType`.
- When type = **Workflow**: show workflow selector + “Start Workflow” action.
- When type = **Agent**: show agent selector + “Start Agent” action; render the agent menu as **trigger buttons** (description on hover).
- When type = **Chat**:
  - show a simple chat input (no workflow start required)
  - chat is “generic” (no explicit agent persona injected), but still **tool-enabled** via ToolPolicy + ToolCalls (e.g., `fs.*` for `@project/@state`)
- The selected workflow/agent is **remembered** per conversation.

#### Flow E — Files View & Preview

1) Files opens as a **slide-out panel** (tree rooted at `@project`)  
2) Clicking a file opens a **new file tab** (or focuses an existing one)  
3) File tab supports **view/edit + save** (Markdown first)  
4) Future file types are handled by **viewer plugins** (extensible registry)

### Files & Directory Management (Project-first Tree)

**Principles:**

- Files tree is rooted at **ProjectRoot** (default focus on `@project/artifacts/`)
- Markdown preview is supported first; other file types are handled via **viewer plugins**
- File actions (create/rename/delete) apply only to `@project` paths

**MVP capabilities:**

- Tree view of **Project** files (expand/collapse)
- Markdown preview with “Open in OS” action
- Create folder/file (default to `@project/artifacts/`, but can operate anywhere under `@project`)
- Rename/delete under `@project` (confirm dialog; recommended: “move to trash”)
- Viewer registry: map file extension → viewer component (extensible later)

### Settings (must exist in the shell)

**Settings → Packages**

- Active package summary (name/version/workflows/agents)
- Import `.bmad` package
- Cache list (name/version/packageId/import time)
- Actions: remove cache, clear all, re-validate, re-import
- If `projectPackageIssue` exists, show a blocking warning banner even when an active package is visible

**Settings → LLM Provider**

- Provider presets:
  - OpenAI
  - OpenAI-compatible (e.g., DeepSeek) — same ToolCalls protocol (`chat.completions`)
  - Azure OpenAI
  - Ollama / local endpoint
- Fields:
  - Base URL / Endpoint
  - Model
  - API key (if needed)
  - Request options (timeout, retries; optional headers)
- Actions:
  - Test connection
  - Save & apply

**Settings → Theme**

- System / Light / Dark
- Optional: High-contrast toggle
- Persist to RuntimeStore settings (no dependency on project)

### State & Persistence (RuntimeStore-backed)

- `recentProjects[]`: `{ projectId, name, projectRoot, lastOpenedAt }`
- `settings`: `{ theme, llmProviderConfig, uiPrefs }`
- `projectBindings`: `{ projectId -> activePackageId, lastRunId }`
- `conversationsIndex`: `{ projectId -> conversations[] }`  
  - Conversation metadata: `{ conversationId, title?, entryType, createdAt, lastActiveAt, workflowRuns[] }`
  - `workflowRuns[]`: `{ runId, workflowId, status, startedAt, lastUpdatedAt }`
- `runsIndex`: `{ projectId -> runs[] }` (metadata only; logs/state are files)

### Out of Scope (explicit)

- Pixel-perfect visual design; focus on IA and flows.
- Advanced approvals UI (“Explain-Plan-Approve”) beyond a placeholder panel; detailed policy comes in later stories.

## Test Plan (Manual)

- Start: New/Open Project works; Settings pinned bottom; Start hidden once project is active
- Settings → Packages: import `.bmad`, show active package summary; cache list updates after remove/clear
- Project: sidebar shows project name/icon + Files + Works
- Files: tree view shows Project files; file tabs open with view/edit + save; file actions operate under `@project`
- Works: create conversation (Agent/Workflow/Chat); switch type; show workflow count and resume a workflow
  - Verify switching type keeps the same conversation and updates the visible controls (Agent/Workflow/Chat)
- Files/Works: clicking in the sidebar opens **slide-out panels** without replacing workspace; Conversation tab stays fixed, file tabs open as needed
- Blocking: missing package shows overlay and blocks Works/Execution until re-imported
- Settings: LLM config saved/persisted; theme change applies immediately

## Tasks / Subtasks (Dev)

- [x] Update shell navigation: Start only when no project; Settings pinned bottom; Project header + Files + Works
- [x] Move Package view into Settings (active package summary + cache list + import/re-validate)
- [x] Implement Works: conversation history list + create (Agent/Workflow/Chat)
- [x] Support conversation type switching + workflow count + workflow resume entry
- [x] Agent menu: trigger buttons in Agent mode; send supports command-only (empty input)
- [x] Conversation management: delete conversation with confirmation (and remove persisted data)
- [x] Files: project tree + markdown preview; viewer plugin registry for future types
- [x] Blocking overlay for missing package (Project context + Works)
- [x] Add workspace tabs (fixed Conversation + multi file tabs) and slide-out panels for Files/Works

## Implementation Notes (Resolved)

- Navigation model matches updated IA: Start + Settings (no project), Project header + Files/Works + Settings (project).
- Files: project-only explorer with file tabs (Conversation fixed + multiple file tabs), markdown preview/edit/save, extensible viewer registry.
- Works: conversation list + create (Agent/Workflow/Chat), type switching (single conversation + shared history), run list + run selector, run details without replacing chat history.
- Agent mode: menu rendered as trigger buttons; optional detail text appended; empty input sends command-only.
- Package binding issues show blocking overlay; Works is blocked until package is resolved.

## Testing Steps

> 目标：验证 Story 5.0 的 UI Shell + Project/Files/Works/Settings 主流程。

### Local Checks

1) `npm -C crewagent-runtime run lint`
2) `npm -C crewagent-runtime run build:ci`

### Manual UX Flow

1) 启动 Runtime（开发或打包模式均可）
2) Start 页点击 **New Project** → 选择目录  
   - 确认生成 `ProjectRoot/artifacts/` 与 `ProjectRoot/.crewagent.json`
3) 点击 **Open Project** → 选择刚创建的项目  
   - 左侧显示 Project 名称 + Files + Works  
   - Start 不再显示，Settings 仍在底部
4) Settings → Packages：导入 `.bmad` 包  
   - 显示 active package summary + cache list
5) Works → **New Conversation**  
   - 选择 Agent / Workflow / Chat  
   - 在会话中切换类型，查看 workflow 启动数量
6) 选择某个 workflow run 继续执行（右侧显示 run details；聊天区保持同一份历史消息）
7) Files：树形结构浏览 Project  
   - 点击文件打开新的文件 Tab（可多开）  
   - Markdown 文件可预览并可编辑保存  
   - 创建/重命名/删除仅作用于 `@project`
8) Settings：修改 LLM Provider/Model/Endpoint → Save  
   - 重启应用后设置仍保持
9) Settings：Cached Packages 执行 Remove/Clear  
   - 若 activePackageId 绑定但 cache 缺失，应出现阻断提示
10) Works：删除 conversation  
   - 确认弹窗出现；删除后列表与持久化数据同步移除

### Optional Verification

- 切换主题（System/Light/Dark），确认全局主题即时生效

### Build/Quality Notes

- 已验证 `crewagent-runtime` 可通过 `eslint`。
- `RuntimeStore.importPackage` 仍是“基础校验”；完整 schema 校验应在 Epic 4 的导入/校验故事中补齐。
