# Story 5.3: Workflow Progress with Real Data

Status: done

## Story

As a **Consumer**,
I want the **Workflow Progress panel to display real-time status from `@state/workflow.md`**,
so that I can **see which steps are completed, which is in progress, and which are pending**.

## Background

The Progress UI already exists (Tab: Progress | Tool Calls | Artifacts | Logs).
This story connects it to real workflow state data from `@state/workflow.md` frontmatter.

## Acceptance Criteria

### 1. Read Step List from Graph

**Given** a Run is active and has a valid `workflow.graph.json`  
**When** I view the Progress panel  
**Then** the step list is populated from the graph nodes (in execution order)

### 2. Display Step Status

**Given** a Run with state file `@state/workflow.md`  
**When** I view the Progress panel  
**Then** each step shows the correct status:
- 🟢 **Completed**: `stepsCompleted[]` includes this nodeId
- 🔵 **In Progress**: `currentNodeId` equals this nodeId
- ⚪ **Pending**: Not in `stepsCompleted` and not `currentNodeId`

### 3. Real-Time Updates

**Given** the ExecutionEngine updates `@state/workflow.md`  
**When** the frontmatter changes (via `fs.apply_patch`)  
**Then** the Progress panel updates automatically to reflect the new state

### 4. Resume from Last State (on App Reopen)

**Given** a project with an incomplete Run (`currentNodeId` is not an end node)  
**When** I reopen the project/conversation  
**Then** the Progress panel shows the last known state

> Note: “Continue / Resume execution” is implemented as part of **Epic 4 (Execution Core)** once the ExecutionEngine exists.

## Design

### Data Sources

| Data | Source |
|------|--------|
| Step list (nodes) | `@pkg/workflow.graph.json` |
| Current step | `@state/workflow.md` frontmatter `currentNodeId` |
| Completed steps | `@state/workflow.md` frontmatter `stepsCompleted[]` |

### IPC Handlers (if needed)

- `runs:getProgress(runId)` → Returns `{ currentNodeId, stepsCompleted, graphNodes }`
- Or: Subscribe to state changes via existing IPC events

### UI Components

- Existing: `Progress` tab in Run Workspace (already implemented)
- Change: Connect to real data instead of mock/static data

### Tech Spec

Full technical design: `tech-spec-5-3-resume-execution-after-crash.md`

## Out of Scope

- Complex pause/resume state machine (Paused, Resumable, Crashed)
- Banner-style notifications for execution state
- Decision log visualization (future story)

## Tasks

- [x] **IPC**: Add `runs:getProgress` handler to read graph + state frontmatter
- [x] **Hook**: Add `useWorkflowProgress(...)` hook in Renderer
- [x] **Integration**: Replace mock steps in Progress panel with real data
- [x] **Subscription**: Emit/subscribe `runs:progressUpdated` when run state changes
- [x] **Run State**: Ensure run-scoped state folder exists (`runs/<runId>/state/`)

## Definition of Done

- [x] Progress panel shows step list from graph
- [x] Step status reflects actual `currentNodeId` and `stepsCompleted`
- [x] UI updates when state file changes during execution
- [x] On app reopen, last run's progress is shown correctly

## Implementation Notes

- **Graph source**: Package workflow graph (`workflow.graph.json`) resolved under package root (sandboxed path resolution).
- **State source**: Prefer `RuntimeStoreRoot/projects/<projectId>/runs/<runId>/state/workflow.md`; fallback to project-level `state/workflow.md` if present.
- **Frontmatter parsing**: Uses `gray-matter` for robust YAML parsing (supports list style for `stepsCompleted`).
- **Real-time updates**: Main process watches the run state directory and broadcasts `runs:progressUpdated`; renderer subscribes and updates the Progress UI.
- **Step naming**: UI displays `node.title` (fallback to `node.name`/`node.id`).

## File List

- `crewagent-runtime/src/pages/RunsPage/RunWorkspace.tsx` (Progress panel uses `useWorkflowProgress`)
- `crewagent-runtime/src/hooks/useWorkflowProgress.ts` (loads progress + subscribes to updates)
- `crewagent-runtime/electron/preload.ts` (exposes `getWorkflowProgress`)
- `crewagent-runtime/electron/main.ts` (`runs:getProgress` + file watcher + `runs:progressUpdated`)
- `crewagent-runtime/electron/stores/runtimeStore.ts` (`getWorkflowProgress`, run state directory helpers)
- `crewagent-runtime/electron/stores/runtimeStore.test.ts` (vitest coverage)

## Change Log

- 2026-01-01: Implemented Story 5.3 workflow progress with real graph/state data + realtime updates.
