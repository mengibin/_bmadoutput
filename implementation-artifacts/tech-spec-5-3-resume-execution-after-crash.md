# Tech Spec: Story 5.3 - Workflow Progress with Real Data

## 1. Problem Statement

The Progress panel in `RunWorkspace.tsx` currently displays **hardcoded mock data** for workflow steps. Users cannot see the actual execution progress when a workflow is running.

## 2. Solution Overview

Connect the Progress panel to real workflow state data from:
- `@pkg/workflow.graph.json` → Step list
- `@state/workflow.md` frontmatter → `currentNodeId`, `stepsCompleted[]`

## 3. Scope

### In Scope
- IPC handler to read workflow progress
- React hook for consuming progress data
- Real-time updates when state changes
- UI integration in existing Progress panel

### Out of Scope
- Resume/Continue execution logic (Epic 4 dependency)
- Decision log visualization
- Complex state machine (Paused, Crashed, Resumable)

## 4. Technical Design

### 4.1 Data Flow

```
Main Process                    Renderer
─────────────────────────────────────────────────────
@state/workflow.md    →    IPC Handler    →    useWorkflowProgress hook
@pkg/workflow.graph.json        ↓                      ↓
                        runs:getProgress        RunDetails component
                                                       ↓
                                              Progress panel UI
```

### 4.2 IPC Handler (Main Process)

**File**: `electron/main.ts`

```typescript
// New handler: runs:getProgress
ipcMain.handle('runs:getProgress', async (_event, payload: { runId: string, projectRoot: string }) => {
  // 1. Read workflow.graph.json from package
  const graphPath = resolvePackagePath(packageId, 'workflow.graph.json')
  const graph = JSON.parse(await fs.readFile(graphPath, 'utf-8'))
  
  // 2. Read @state/workflow.md frontmatter
  const statePath = resolveStatePath(runId, 'workflow.md')
  const stateContent = await fs.readFile(statePath, 'utf-8')
  const frontmatter = parseFrontmatter(stateContent)
  
  // 3. Build step list with status
  const steps = graph.nodes.map(node => ({
    id: node.id,
    name: node.name || node.id,
    status: frontmatter.stepsCompleted.includes(node.id) 
      ? 'completed' 
      : node.id === frontmatter.currentNodeId 
        ? 'current' 
        : 'pending'
  }))
  
  return { steps, currentNodeId: frontmatter.currentNodeId }
})
```

### 4.3 Preload Bridge

**File**: `electron/preload.ts`

```typescript
getWorkflowProgress: (payload: { runId: string, projectRoot: string }) => 
  ipcRenderer.invoke('runs:getProgress', payload),
```

### 4.4 React Hook

**File**: `src/hooks/useWorkflowProgress.ts`

```typescript
export interface StepProgress {
  id: string
  name: string
  status: 'completed' | 'current' | 'pending'
}

export function useWorkflowProgress(runId: string | null) {
  const [steps, setSteps] = useState<StepProgress[]>([])
  const [currentNodeId, setCurrentNodeId] = useState<string | null>(null)
  const [loading, setLoading] = useState(true)

  useEffect(() => {
    if (!runId) return
    
    const fetchProgress = async () => {
      const result = await window.ipcRenderer.getWorkflowProgress({ runId, projectRoot })
      setSteps(result.steps)
      setCurrentNodeId(result.currentNodeId)
      setLoading(false)
    }
    
    fetchProgress()
    
    // Subscribe to state changes
    const unsubscribe = window.ipcRenderer.on('runs:progressUpdated', (data) => {
      if (data.runId === runId) {
        setSteps(data.steps)
        setCurrentNodeId(data.currentNodeId)
      }
    })
    
    return () => unsubscribe()
  }, [runId])

  return { steps, currentNodeId, loading }
}
```

### 4.5 UI Integration

**File**: `src/pages/RunsPage/RunWorkspace.tsx`

Replace hardcoded steps (lines 49-70) with dynamic data:

```tsx
function RunDetails({ run }: { run: Run }) {
  const { steps, loading } = useWorkflowProgress(run.runId)
  
  // ... tab logic ...
  
  {activeTab === 'progress' && (
    <div className="progress-content">
      <h3>Workflow Progress</h3>
      {loading ? (
        <div className="loading">Loading...</div>
      ) : (
        <div className="step-list">
          {steps.map((step) => (
            <div key={step.id} className={`step ${step.status}`}>
              <span className="step-indicator" />
              <span className="step-name">{step.name}</span>
              <span className="step-status">
                {step.status === 'completed' ? 'Completed' : 
                 step.status === 'current' ? 'In Progress' : 'Pending'}
              </span>
            </div>
          ))}
        </div>
      )}
    </div>
  )}
}
```

### 4.6 Real-Time Updates

When `ToolHost` writes to `@state/workflow.md`, emit event:

**File**: `electron/main.ts` (in fs.apply_patch handler)

```typescript
// After successful write to @state/workflow.md
if (path === '@state/workflow.md') {
  const progress = await getWorkflowProgress(runId)
  win?.webContents.send('runs:progressUpdated', { runId, ...progress })
}
```

## 5. Files to Modify

| File | Change |
|------|--------|
| `electron/main.ts` | Add `runs:getProgress` handler |
| `electron/preload.ts` | Add `getWorkflowProgress` bridge |
| `src/hooks/useWorkflowProgress.ts` | **NEW** - Progress hook |
| `src/pages/RunsPage/RunWorkspace.tsx` | Replace mock data with hook |

## 6. Testing Strategy

### Manual Testing
1. Open a project with an active run
2. Verify step list matches `workflow.graph.json` nodes
3. Verify current step highlights correctly
4. (If ExecutionEngine exists) Verify updates when step completes

### Build Verification
- `npm run build` passes
- No TypeScript errors

## 7. Dependencies

- **Blocker**: None for UI implementation
- **Partial**: Real-time updates depend on ExecutionEngine existing (can mock/skip for MVP)

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| `@state/workflow.md` not existing | Return empty steps with helpful message |
| Graph has complex branching | Only show nodes in execution path (or show all with skipped state) |
