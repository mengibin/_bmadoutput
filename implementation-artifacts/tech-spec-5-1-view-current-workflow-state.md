# Tech Spec: Story 5-1 – View Current Workflow State

## Summary
Implement a read-only React Flow visualization for the active workflow.

## 1. Frontend Architecture

### 1.1 Dependencies
- Install `@xyflow/react` (React Flow v12)
- `npm install @xyflow/react`

### 1.2 Components
- `WorkflowGraphPanel`: Main container, handles data fetching.
- `WorkflowGraph`: Wrapper for `<ReactFlow>`.
- `GraphNode`: Custom node component for "Step" (title, icon).

### 1.3 State Management (`appStore`)
- Use `activePackage` to find the current workflow ref.
- Use `activeConversation.selectedWorkflowId` (if available) or default workflow.
- Fetch graph JSON via `files:read`.

### 1.4 Data Flow
1. **Mount**: `WorkflowGraphPanel` checks `activePackage`.
2. **Identify Graph Path**: 
   ```typescript
   const wf = activePackage.workflows.find(w => w.id === selectedWorkflowId)
   const graphPath = wf.graph // e.g. "graphs/my-flow.graph.json"
   ```
3. **Fetch Data**:
   ```typescript
   const result = await window.ipcRenderer.readFile({
     aliasPath: `@pkg/${graphPath}`,
     options: { packageId: activePackage.id }
   })
   const graphData = JSON.parse(result.content)
   ```
4. **Transform**: Convert `graphData` (Bmad V1.1 Graph) to React Flow `nodes` and `edges`.
   - **Nodes**: Map `id` -> `id`. Map `type` -> `type`. Position comes from `metadata.position` or defaults.
   - **Edges**: Map `from`/`to` -> `source`/`target`.

## 2. UI/UX Design

### 2.1 Placement
- Inside `WorksPage`.
- Add a Toggle/Tab in the Header: `[ Chat | Graph ]`.
- **Graph Mode**: Hides Chat (or minimizes it), shows full-screen Graph.

### 2.2 Node Styling
- **Standard Node**: Rounded rectangle, 1px border.
- **Start Node**: Green border/accent.
- **End Node**: Red/Dark border/accent.
- **Selection**: Thick blue border.

### 2.3 Interaction
- **Click Node**: Update local state `selectedNodeId`.
- **Side Panel**: If `selectedNodeId` is set, show a floating "Node Details" panel with the step's raw definition (or Markdown if available).
    - Note: To show Markdown, we need the *workflow definition* file too (`wf.workflow`), map the node ID to the step file, and read that file. *Complexity Alert*.
    - *Scope Reduction*: For this story, just show the JSON properties of the node in the side panel. Reading step markdown involves parsing the `workflow.yaml` which mapping steps to files. This might be too heavy for 5-1.
    - *Revised Plan*: Show Node ID, Type, and any immediate metadata in the JSON.

## 3. Testing

### 3.1 Verification Steps
1. **Load Project**: Open a project with a valid package imported.
2. **Enter Works**: Go to Works page.
3. **Toggle Graph**: Click "Graph" view.
4. **Verify Render**: graph nodes appear.
5. **Verify Layout**: Nodes are not all at (0,0) (assuming graph JSON has positions from Builder).
    - *Risk*: If Builder didn't save positions, auto-layout might be needed. `dagre` is common for this.
    - *Mitigation*: Check if `reactflow` handles partial layout, or assume input has positions. If not, simple grid or warn user.
6. **Switch Workflow**: If package has multiple workflows (Settings?), ensure Graph updates.

## 4. Risks & Edge Cases
- **Missing Positions**: If JSON lacks x/y, graph looks piled up. -> *Validation*: Add simple auto-layout if x/y missing, or just accept it's a "viewer" for Builder-generated graphs.
- **Large Graphs**: Performance check.
- **Invalid JSON**: Handle parse errors gracefully.

