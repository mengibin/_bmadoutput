# Tech Spec: Story 5-1 - View Current Workflow State (Graph UI)

**Story**: `5-1-view-current-workflow-state.md`  
**Status**: Draft

## 1. Objective
Provide a read-only **Workflow Graph** view inside **Works** so users can understand the static execution flow:
- Render nodes/edges from `workflow.graph.json` (React Flow).
- The graph is used as a **pre-start preview**; it should not expose per-step Markdown content to end users in this story.

## 2. Data Source & Contracts

### 2.1 Package Metadata
- Renderer receives `activePackage` from `packages:list` (Electron RuntimeStore).
- `activePackage.workflows[]` includes:
  - `id`
  - `name`
  - `graph` (relative path inside package)
  - `workflow` (relative path inside package)

### 2.2 Graph File
- Read via IPC:
  - `window.ipcRenderer.readFile({ path: "@pkg/<graphPath>", packageId })`
- Expected schema (v1.1):
  - `entryNodeId`
  - `nodes[]: { id, type, file, title?, agentId? }`
  - `edges[]: { from, to, label }`

## 3. UI Integration

### 3.1 Entry Point
- **Works → Conversation header** adds a **Graph** toggle button.
- In **Workflow mode before starting a run**, the left panel defaults to Graph view so users can understand the flow before clicking Start.

### 3.2 Layout
- Left: React Flow canvas (pan/zoom/minimap/controls).
- Right: Existing Conversation Details / Start controls (or RunDetails when a run exists).

### 3.3 Workflow Selection
Graph workflow selection is derived from:
1. `activeRun.workflowRef` (if present)
2. `selectedConversation.selectedWorkflowId`
3. `draftWorkflowId`
4. fallback to `activePackage.workflows[0].id`

## 4. Error Handling / Empty States
- Missing/invalid graph JSON → show error in graph view.
- If graphPath is missing → disable Graph button and show tooltip.

## 5. Implementation Notes
- New component:
  - `crewagent-runtime/src/components/workflow/WorkflowGraphView.tsx`
  - `crewagent-runtime/src/components/workflow/WorkflowGraphView.css`
- Dependencies:
  - `@xyflow/react`
- Works integration:
  - `crewagent-runtime/src/pages/WorksPage/WorksPage.tsx`

## 6. Testing Strategy

### 6.1 Manual
1. Open project with an active package.
2. Works → select a workflow conversation (before Start) → confirm graph is visible (or toggle **Graph**).
3. Click nodes and confirm it does not open per-step Markdown content.
4. Toggle back to chat and verify chat history is unchanged.

### 6.2 Build
- `npm -C crewagent-runtime run build:ci`
