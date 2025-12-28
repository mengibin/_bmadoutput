# Story 5.0: Runtime UI Shell & Navigation (IA First)

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Consumer**,  
I want a consistent Runtime application shell (navigation + core layout),  
so that all later features (import/run/progress/log/settings) fit into a coherent UI and don’t require rework.

## Acceptance Criteria

1. **Given** I open the Runtime app  
   **When** no Project is selected yet  
   **Then** I can **New Project** (create a folder) or **Open Project** (pick an existing folder)  
   **And** New Project creates `ProjectRoot/artifacts/` and a non-secret config file `ProjectRoot/.crewagent.json`  
   **And** the app clearly shows the active ProjectRoot once selected

2. **Given** a Project is selected  
   **When** I need to run a workflow  
   **Then** I can **Import/Open `.bmad` Package** (with validation feedback)  
   **And** I can choose entry mode: **Workflow-First** or **Agent-First**

3. **Given** I have selected a Project and a Package  
   **When** I enter the main Runtime shell  
   **Then** the shell provides (at minimum):  
   - persistent navigation (Project / Files / Package / Workflows / Agents / Runs / Settings)  
   - a unified Run workspace surface (Chat + Progress + Tool Calls + Logs + Artifacts)  
   - a global status area showing Project/Package/Run and RunPhase (Running/WaitingUser/Completed/Failed)

4. **Given** I am inside a Project  
   **When** I open “Files”  
   **Then** I can browse and manage the full ProjectRoot directory tree (create/open/rename/delete files and folders, and open in OS file explorer)  
   **And** I can quickly locate: generated artifacts, current run state/logs (read-only), and package files (read-only)

5. **Given** the Package contains workflows and agents  
   **When** I open “Workflows” or “Agents”  
   **Then** I can view workflow list/preview and agent list/persona/menu  
   **And** I can start a run either by selecting a workflow (Workflow-First) or selecting an agent then choosing a menu item (Agent-First)

6. **Given** I open “Settings”  
   **When** I configure Runtime  
   **Then** I can manage:
   - Package import/cache (list, remove/clear, re-validate)
   - LLM Provider (OpenAI / OpenAI-compatible like DeepSeek / Azure / Ollama), model, endpoint, API key, test connection
   - Theme (system/light/dark; optional high-contrast)

7. **Given** I trigger an execution-related action (Run/Pause/Resume/Stop)  
   **When** the UI updates  
   **Then** it reflects canonical state derived from `@state/workflow.md` + `@pkg/workflow.graph.json` (not from chat memory)  
   **And** both entry modes converge into the same Run workspace (same panes + same stop-point semantics)

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
- The UI must make the “Project / Package / Run” mental model explicit and keep **state visible** (Document-as-State) while preserving **Project-First + Private RuntimeStore** constraints.

### Navigation Model (IA)

**Top-level pages (left sidebar / nav rail):**

- **Start** (only when no project selected): New/Open Project, Import Package, pick entry mode
- **Project**
  - Overview (active ProjectRoot, recent projects, quick actions)
- **Files**
  - Project tree (`@project`, read/write)
  - Package tree (`@pkg`, read-only)
  - Run state/logs (`@state`, read-only for browsing)
- **Package**
  - Current package summary (name/version/schemaVersion, validation status)
- **Workflows**
  - List workflows; per-workflow preview derived from `workflow.graph.json` + `workflow.md`
- **Agents**
  - List agents; per-agent persona and menu preview; “Start Agent Session”
- **Runs**
  - Current run workspace
  - Run history list (per project)
- **Settings**
  - LLM Provider
  - Package cache/import
  - Theme & UI preferences

**Global status bar (always visible):**

- Active Project (name/path), Package (name/version), Run (runId + phase)
- Primary controls: Import Package, Open Project, Settings
- Run controls (enabled only when a run exists): Run / Pause / Resume / Stop

### Key Flows

#### Flow A — New/Open Project

1) User clicks **New Project** → choose parent folder + project name → create folder  
2) Runtime ensures `ProjectRoot/artifacts/` exists (Project-First)  
3) Runtime creates a project-level config file (non-secret) at `ProjectRoot/.crewagent.json`  
4) Runtime stores project in **recent projects** (in RuntimeStore)

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

#### Flow B — Import/Open `.bmad` Package

1) User clicks **Import Package** → selects `.bmad` zip  
2) Runtime validates (schema + referenced files), then extracts to RuntimeStore package cache and assigns `packageId`  
3) UI shows:
   - validation result (pass/fail + actionable errors with file path)
   - package summary (`bmad.json` name/version; workflows list; agents list)
   - “Set as Active Package for this Project”

#### Flow C — Choose Entry Mode

- **Workflow-First**:
  - choose workflow (from `bmad.json.entry` default or `workflows[]`)
  - choose/confirm default `activeAgentId` (if needed)
  - click **Start Run** → enters Run workspace
- **Agent-First**:
  - choose agent → enters Agent Session (menu view)
  - user selects menu item (trigger/number/text)
  - Runtime routes command via CommandRouter → starts run (if workflow/exec) → enters Run workspace

#### Flow D — Run Workspace (unified)

**Layout suggestion:**

- Left: **Chat** (assistant output stream + user input box)
- Right: tabbed panels (default tab: **Progress**)
  - **Progress**: graph/step list, current node highlight, stepsCompleted, variables, decisionLog count (default)
  - **Tool Calls**: tool call list + details (args/result; approval placeholder)
  - **Artifacts**: list from `@state/workflow.md.artifacts` → open in Files
  - **Logs**: execution.jsonl viewer (filters: tool/error/llm)

**Stop-point semantics (must match architecture):**

- if assistant returns toolCalls → Runtime executes and continues internal loop
- if no toolCalls:
  - workflow complete → Completed
  - otherwise → WaitingUser (input box focused)

### Files & Directory Management (Project / Package / State)

**Principles:**

- Default safe working area: `@project/artifacts/` (auto-created), but user can manage the full ProjectRoot
- `@pkg` is read-only (UI shows lock icon; no create/rename/delete)
- `@state` is private but inspectable (read-only browse), since it’s essential for debugging

**MVP capabilities:**

- Tree view with three roots: **Project**, **Package**, **Run State**
- File preview (text) + “Open in OS” action
- Create folder/file (default to `@project/artifacts/`, but can operate anywhere under `@project`)
- Rename/delete under `@project` (confirm dialog; recommended: “move to trash”)
- Quick filters: show only `artifacts/`, show modified recently, search by filename

### Settings (must exist in the shell)

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

**Settings → Packages**

- List cached/imported packages (name/version/packageId/source filename/import time)
- Actions: remove cache, clear all, re-validate

**Settings → Theme**

- System / Light / Dark
- Optional: High-contrast toggle
- Persist to RuntimeStore settings (no dependency on project)

### State & Persistence (RuntimeStore-backed)

- `recentProjects[]`: `{ projectId, name, projectRoot, lastOpenedAt }`
- `settings`: `{ theme, llmProviderConfig, uiPrefs }`
- `projectBindings`: `{ projectId -> activePackageId, lastRunId }`
- `runsIndex`: `{ projectId -> runs[] }` (metadata only; logs/state are files)

### Out of Scope (explicit)

- Pixel-perfect visual design; focus on IA and flows.
- Advanced approvals UI (“Explain-Plan-Approve”) beyond a placeholder panel; detailed policy comes in later stories.

## Test Plan (Manual)

- Start: New Project creates folder + artifacts; Open Project loads existing
- Package import: valid package shows summary; invalid package shows actionable errors and blocks “Set Active”
- Entry modes: Workflow-First starts run; Agent-First enters agent session and can start a run via menu
- Run workspace: on WaitingUser, input focuses; on Running, tool calls stream and panels update
- Files: can create under artifacts; package/state roots are read-only; open in OS works
- Settings: LLM config saved/persisted; theme change applies immediately; package cache list updates after import/remove

## Tasks / Subtasks (Dev)

- [ ] Implement Runtime shell layout (nav + status bar + page routing)
- [ ] Implement Start page (New/Open Project, create `artifacts/` + `.crewagent.json`, Import Package, choose entry mode)
- [ ] Implement Files page (tree + preview + full `@project` management + read-only `@pkg/@state`)
- [ ] Implement Package/Workflows/Agents pages (list + preview + start entry)
- [ ] Implement Run workspace scaffold (Chat + tabs; stop-point state machine wiring placeholder)
- [ ] Implement Settings pages (LLM Provider, Packages, Theme) + persistence in RuntimeStore
