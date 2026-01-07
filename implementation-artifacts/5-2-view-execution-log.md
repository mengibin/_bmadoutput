# Story 5.2: Rich Execution Log Panel

Status: done

## Story

As a **User**,
I want to **view a real-time execution log in a separate panel**,
so that I can see the Agent's detailed "Thinking" and "Tool Calls" without cluttering the main chat window.

## Acceptance Criteria

1. **Given** the Workspace
   **Then** a **Log Panel** is available (e.g., in a bottom drawer or side tab).

2. **Given** the Log Panel
   **When** the Agent generates a "Thought"
   **Then** a **"Thinking Block"** appears in the log list.
   - Style: Collapsible, shows duration, icon, and raw content (like the screenshot).

3. **Given** the Log Panel
   **When** the Agent calls a tool
   **Then** a **"Tool Block"** appears in the log list.
   - Style: Clear separation of Input and Output.

4. **Given** new log entries are added
   **When** I have not manually scrolled up
   **Then** the log panel auto-scrolls to the bottom to show the latest entry.

5. **Given** standard system events
   **When** an error or info message occurs
   **Then** it appears as a standard log line (timestamp + message).

6. **Given** the Chat Window
   **Then** it remains clean, showing only the final Assistant responses (and user messages), while the heavy lifting is visible in the Log Panel.

## Validation Notes
- **Mocking**: Simulate `User Message` -> `Log: Thought` -> `Log: Tool` -> `Chat: Assistant Response`.
- **Reference**: The UI style should match the provided screenshot (clean blocks), but placed in a dedicated log container.

## Technical Context
- **UI Component**: `LogViewer` (renders a list of `LogEntry` items).
- **Data Model**: `LogEntry` needs structured types (`thought`, `tool`, `text`) to render different blocks.
- **IPC (Future)**: Backend will stream logs via `execution:log` IPC channel.
- **Overflow Handling**: Limit to 1000 entries; trim oldest when exceeded.

## Dependencies
- Story 5-0 (Shell) - Panel container.

## Design

### Summary
The Rich Execution Log Panel provides real-time visibility into Agent execution using a bottom collapsible panel with color-coded, collapsible blocks for different log types (Thinking, Tools, Events).

### Layout & Hierarchy

#### Panel Structure
```
┌─────────────────────────────────────────────────┐
│  Workspace (Chat + Workflow View)              │
│  [Main Content Area]                            │
├─────────────────────────────────────────────────┤ ← Resize Handle
│  📋 Execution Log          [🔄][🗑️][⚙️]        │
│  ┌───────────────────────────────────────────┐ │
│  │ > Thought for 12s                         │ │
│  │ > Tool: Search Google                     │ │
│  │ ✓ Assistant Response                      │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Panel Position**: Bottom collapsible panel (similar to VS Code terminal)
- **Default Height**: 30% of viewport
- **Min/Max Height**: 100px / 70% viewport
- **Resize**: Draggable handle at top edge
- **Toggle**: Toolbar button or `Cmd/Ctrl+L`

#### Panel Header
```
┌─────────────────────────────────────────────────┐
│ 📋 Execution Log          [🔄][🗑️][⚙️]        │
└─────────────────────────────────────────────────┘
```

**Controls**:
- **Auto-scroll Toggle** 🔄: Active (blue) / Inactive (gray)
- **Clear Button** 🗑️: Removes all entries (`Cmd/Ctrl+K`)
- **Filter** ⚙️: Dropdown (All/Info/Warning/Error/Debug)

### UX / UI

#### ThinkingBlock Component
```
┌─────────────────────────────────────────────────┐
│ › 🧠 Thought for 12s                  14:32:45  │
├─────────────────────────────────────────────────┤
│ I need to search for recent information         │
│ about this topic to provide an accurate answer. │
└─────────────────────────────────────────────────┘
```

**States**:
-  **Collapsed** (default): `› 🧠 Thought for {duration}` + timestamp
- **Expanded**: Full thought content

**Styling**:
- Background: `#2D2D3D` (dark gray/purple)
- Border-left: `3px solid #8B5CF6` (purple)
- Hover: Lighten 5%
- Transition: 200ms smooth

#### ToolBlock Component
```
┌─────────────────────────────────────────────────┐
│ › 🔧 search_google                    14:32:47  │
├─────────────────────────────────────────────────┤
│ Input: { query: "latest AI developments" }      │
│ Output: [3 results found...]                    │
└─────────────────────────────────────────────────┘
```

**Styling**:
- Background: `#1E3A3A` (dark teal)
- Border-left: `3px solid #10B981` (green)
- Code: Monospace, syntax-highlighted JSON

#### TextLog (Info/Warning/Error)
```
ℹ️  Workflow started                   14:32:40
⚠️  Rate limit approaching             14:33:10
❌ API call failed: Timeout           14:33:15
```

**Color Coding**:
- Info: Blue icon, default background
- Warning: Yellow border + icon
- Error: Red border + icon

### Color Palette
| Element | Color | Usage |
|---------|-------|-------|
| Panel BG | `#1A1A1A` | Main background |
| Thinking | `#2D2D3D` + `#8B5CF6` border | Purple |
| Tool | `#1E3A3A` + `#10B981` border | Green |
| Error | `#3D1E1E` + `#EF4444` border | Red |
| Text | `#E5E7EB` | Primary |
| Muted | `#9CA3AF` | Timestamps |

### Typography
- **Headers**: 14px Medium (500)
- **Body**: 13px Regular (400)
- **Code**: 12px Monospace (Fira Code/Consolas)
- **Timestamps**: 11px Muted

### Spacing
- Block Padding: 12px vertical, 16px horizontal
- Block Gap: 8px
- Icon-Text Gap: 8px

### Interaction Patterns

#### Auto-Scroll Behavior
1. **Default**: ON (new entries scroll into view)
2. **Auto-Disable**: When user scrolls up manually
3. **Re-Enable**: When user scrolls to bottom OR clicks toggle

#### Keyboard Shortcuts
- `Cmd/Ctrl + L`: Toggle panel visibility
- `Cmd/Ctrl + K`: Clear all logs

#### Collapsible Blocks
- **Click anywhere**: Toggle expand/collapse
- **Smooth transition**: 200ms

### Empty States

**No Logs**:
```
┌─────────────────────────────────────────────────┐
│              📋 No logs yet                      │
│    Start a workflow to see execution details    │
└─────────────────────────────────────────────────┘
```

**Filtered Empty**:
```
┌─────────────────────────────────────────────────┐
│         🔍 No logs match this filter             │
│          Try changing the filter settings        │
└─────────────────────────────────────────────────┘
```

### Component File Structure
```
src/components/logs/
├── LogViewer.tsx          # Main container
├── LogViewer.css
├── ThinkingBlock.tsx      # Collapsible thought
├── ThinkingBlock.css
├── ToolBlock.tsx          # Collapsible tool
├── ToolBlock.css
├── TextLogEntry.tsx       # Simple text entry
├── TextLogEntry.css
└── index.ts
```

### Performance
- **Virtual Scrolling**: Use `react-window` if > 100 visible
- **Entry Limit**: Max 1000; trim oldest
- **Lazy Render**: Collapsed blocks defer content render
- **Scroll Debounce**: 50ms

### Accessibility
- **ARIA**: Buttons have labels, panel has role="region"
- **Keyboard Navigation**: Tab through toolbar, Enter to expand blocks
- **Color Independence**: Icons + borders (not just color)

### Reference
- **Layout**: VS Code Integrated Terminal
- **Block Style**: Provided user screenshot (collapsible, clean blocks)

## Tasks / Subtasks

- [x] Update `RuntimeStore` to add `logs: LogEntry[]` and `addLog` action.
- [x] Implement `ThinkingBlock.tsx` component (collapsible with duration).
- [x] Implement `ToolBlock.tsx` component (collapsible with input/output).
- [x] Implement `LogViewer.tsx` container with auto-scroll logic.
- [x] Add toolbar (Clear, Auto-scroll toggle, Filter dropdown).
- [x] Integrate LogViewer into Workspace Layout (Bottom Panel).
- [x] Add "Simulate Logs" button for testing/mocking.
- [x] Implement log overflow handling (max 1000 entries).

## Dev Agent Record

### Implementation Plan
Story 5.2 implements a rich execution log panel with collapsible blocks for thinking/tool calls. Core components:
- **RuntimeStore**: Added LogEntry types (LogType, LogLevel, LogEntry interface) + state array + 3 methods
- **UI Components**: ThinkingBlock (purple), ToolBlock (green), TextLogEntry (color-coded)
- **LogViewer**: Auto-scroll, toolbar (clear/filter/auto-scroll toggle), empty states

### Completion Notes
- ✅ All 8 tasks completed
- ✅ TypeScript types defined and exported
- ✅ Overflow handling (max 1000) implemented in RuntimeStore.addLog()
- ✅ Rich UI components with color-coded themes matching design spec
- ✅ Auto-scroll logic with manual scroll detection
- ✅ Filter dropdown (All/Success/Info/Warning/Error/Debug)
- ✅ Empty states for no logs / filtered results
- ✅ Dev-only simulate logs button (🧪) for rapid UI validation
- ✅ Real-time updates via IPC push (`execution:log`, `execution:log:clear`)
- ✅ Panel UX: resize handle + shortcuts (Cmd/Ctrl+L toggle, Cmd/Ctrl+K clear)
- ✅ Logs cleared on project switch

**Next Steps**:
1. [Optional] Persist logs per run/project (align with `RuntimeStoreRoot/projects/<projectId>/runs/<runId>/...` in Story 4-8).
2. [Optional] Replace dev simulation with real ExecutionEngine log emission once the agent loop is integrated (Epic 4).

**IPC Integration (2025-12-31 - Complete)**:
- ✅ Added 3 IPC handlers in `main.ts`
- ✅ Exposed API methods in `preload.ts`
- ✅ Created `useLogs()` React hook for frontend integration
- ✅ Added IPC push events for real-time log streaming (`execution:log`, `execution:log:clear`)

## File List
- electron/stores/runtimeStore.ts (modified: added LogEntry types + logs state + methods)
- electron/main.ts (modified: added logs IPC handlers)
- electron/preload.ts (modified: exposed log API)
- src/hooks/useLogs.ts (new)
- src/components/logs/ThinkingBlock.tsx (new)
- src/components/logs/ThinkingBlock.css (new)
- src/components/logs/ToolBlock.tsx (new)
- src/components/logs/ToolBlock.css (new)
- src/components/logs/TextLogEntry.tsx (new)
- src/components/logs/TextLogEntry.css (new)
- src/components/logs/LogViewer.tsx (new)
- src/components/logs/LogViewer.css (new)
- src/components/logs/index.ts (new)

## Change Log
- 2025-12-31: Implemented Story 5.2 - Rich Execution Log Panel (13 files created/modified)
- 2025-12-31: Added IPC integration for log panel (main.ts, preload.ts, useLogs hook)
