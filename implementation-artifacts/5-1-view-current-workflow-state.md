# Story 5-1: View Current Workflow State (Graph View)

## Overview
**Epic**: 5 – Runtime Frontend  
**Priority**: Phase 2 (Visualization)  
**Status**: `done`

## Goal
Visualize the currently selected workflow definition as an interactive node graph, serving as the "map" for the user to understand execution flow.

## Business Value
- **Transparency**: Users can see exactly what steps the Agent will execute.
- **Monitoring**: In future stories (Phase 3/4), this view will highlight the *active* step, acting as the primary execution monitor.

## Acceptance Criteria
1. **Graph Rendering**: Render the selected workflow's graph using React Flow.
    - Nodes: Steps, Start, End.
    - Edges: Connections between steps.
2. **Pre-Start Preview**: In Workflow mode (before starting a run), users can preview the graph to understand what will happen after they click Start.
3. **No Step Detail Exposure**: The graph is for understanding the flow; it should not expose per-step Markdown files/content to end users in this story.
4. **Data Source**: Load graph data from the active Package's `workflow.graph.json` file.
5. **Layout**:
    - "Works" page should have a split view or tab to show the Graph.
    - *Decision*: Since "Works" is the conversation view, the Graph should be a "Third Column" or a "Drawer" compliant with the UI shell, OR a toggle view in the main area updates.
    - *Refined Criteria*: Add a "View Graph" toggle/tab in the Conversation header. When active, show the Graph in the main content area (replacing or split with chat).

## Dependencies
- **Package Management (Story 5-5)**: Needs an active package to be loaded.
- **RuntimeStore**: Needs to read file content from the package (`@pkg/`).
- **External Lib**: `@xyflow/react` (React Flow).

## Technical Context
- **Data Source**: `activePackage.workflows[current].graph` contains the relative path.
- **File Access**: Use `files:read` with alias `@pkg/<path>` to fetch the JSON.
- **React Flow**: Use custom node types for "Step" to display name/id.

## Related Artifacts
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-5-1-view-current-workflow-state.md`

## Implementation Notes
- UI entry: **Works → Conversation header → Graph** toggle.
- Graph source: `activePackage.workflows[workflowId].graph` (read via `files:read` + `@pkg/`).
- Before starting a run, the main panel defaults to Graph view for workflow conversations, while the right panel still shows the Start controls.

## Test Plan

### Manual Verification
1. Open a project with an active package (e.g. `create-story-micro`).
2. Go to **Works**, select a workflow conversation.
3. Confirm the workflow graph is visible before clicking Start (or toggle **Graph** in the conversation header).
4. Confirm the workflow graph renders and is navigable (pan/zoom).
5. Click nodes; confirm it does not open per-step Markdown content.
6. Toggle Graph off; confirm chat history remains unchanged.

### Build Verification
- `npm -C crewagent-runtime run build:ci`
