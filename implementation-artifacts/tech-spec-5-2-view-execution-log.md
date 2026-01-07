# Tech Spec: Story 5.2 Rich Execution Log Panel

## Overview

This feature provides a dedicated **Log Panel** (separate from Chat) that visualizes runtime events using "Rich Blocks" (Thoughts, Tool Calls) similar to modern chat interfaces. This keeps the main chat clean while providing deep debugging visibility.

## Architecture

### Frontend (React)

#### `LogViewer` Component
- **Path**: `src/components/logs/LogViewer.tsx`
- **Props**: `logs: LogEntry[]`
- **Render Logic**:
    - Iterate through logs.
    - Switch on `log.type`:
        - `'thought'`: Render `ThinkingBlock` (Collapsible, duration, raw content).
        - `'tool'`: Render `ToolBlock` (Collapsible, input/output).
        - `'text'`: Render `status` line (Info/Error).

#### `RuntimeStore` (Log Aggregation)
- **State**: `logs: LogEntry[]` (Transient, cleared on reload/project switch).
- **Actions**: `addLog(entry)`

### Data Model

```typescript
// electron/stores/runtimeStore.ts

export type LogType = 'text' | 'thought' | 'tool'
export type LogLevel = 'info' | 'warning' | 'error' | 'debug' | 'success'

export interface LogEntry {
    id: string
    timestamp: string
    type: LogType
    level: LogLevel
    message?: string // For text logs or fallback
    // Rich Data
    thoughtContent?: string
    toolName?: string
    toolInput?: Record<string, unknown>
    toolOutput?: unknown
    durationMs?: number
}
```

## Implementation Steps

1.  **Update Store**: Add `LogEntry` type and `logs` array to `RuntimeStore`.
2.  **Components**:
    - `ThinkingBlock`: Reusable component for thought logs.
    - `ToolBlock`: Reusable component for tool logs.
    - `LogViewer`: Main list container.
3.  **Integration**: Add `LogViewer` to the Workspace Layout (Bottom Panel).
4.  **Mocking**: Update "Simulate" to emit `thought` and `tool` log entries.

## Styling
- **Thinking**: Dark gray background, brain icon.
- **Tool**: Dark blue/green background, terminal icon.
- **Scroll**: Auto-scroll behavior on new entries.
