# UX Design: Story 5.2 - Rich Execution Log Panel

**Version**: 1.0  
**Date**: 2025-12-31  
**Status**: Ready for Review

---

## Overview

The **Rich Execution Log Panel** provides users with a real-time view into the Agent's execution process, showing "Thinking" phases, tool usage, and system events in a clean, structured format. This design keeps the main chat interface uncluttered while offering deep transparency for power users and debugging.

---

## Design Principles

1. **Separation of Concerns**: Chat shows final answers; Log shows process.
2. **Progressive Disclosure**: Collapsible blocks hide complexity by default.
3. **At-a-Glance Clarity**: Color-coded blocks and icons for instant recognition.
4. **Performance-Conscious**: Virtual scrolling and entry limits prevent UI lag.

---

## Layout & Hierarchy

### Panel Position
```
┌─────────────────────────────────────────────────┐
│  Workspace (Chat + Workflow View)              │
│                                                 │
│  [Main Content Area]                            │
│                                                 │
├─────────────────────────────────────────────────┤ ← Resize Handle
│  📋 Execution Log          [🔄][🗑️][⚙️]        │
│  ┌───────────────────────────────────────────┐ │
│  │ > Thought for 12s                         │ │
│  │ > Tool: Search Google                     │ │
│  │ ✓ Assistant Response                      │ │
│  └───────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

**Position**: Bottom Panel (collapsible)  
**Default Height**: 30% of viewport  
**Min Height**: 100px  
**Max Height**: 70% of viewport  
**Resize**: Draggable handle at top edge

---

## Component Breakdown

### 1. Panel Header
```
┌─────────────────────────────────────────────────┐
│ 📋 Execution Log          [🔄][🗑️][⚙️]        │
└─────────────────────────────────────────────────┘
```

**Elements**:
- **Title**: "Execution Log" (14px, medium weight)
- **Auto-scroll Toggle** 🔄: Active (blue) / Inactive (gray)
- **Clear Button** 🗑️: Removes all entries
- **Settings** ⚙️: Filter dropdown (All/Info/Warning/Error/Debug)

---

### 2. ThinkingBlock Component

```
┌─────────────────────────────────────────────────┐
│ › 🧠 Thought for 12s                  14:32:45  │
├─────────────────────────────────────────────────┤
│ I need to search for recent information         │
│ about this topic to provide an accurate answer. │
└─────────────────────────────────────────────────┘
```

**States**:
- **Collapsed** (default): Shows "› 🧠 Thought for {duration}" + timestamp
- **Expanded**: Shows full thought content

**Styling**:
- Background: `#2D2D3D` (dark gray/purple)
- Border-left: `3px solid #8B5CF6` (purple accent)
- Icon: 🧠 (or brain SVG)
- Hover: Lighten background by 5%

**Interaction**:
- Click anywhere to toggle expand/collapse
- Smooth transition (200ms)

---

### 3. ToolBlock Component

```
┌─────────────────────────────────────────────────┐
│ › 🔧 search_google                    14:32:47  │
├─────────────────────────────────────────────────┤
│ Input:                                          │
│   { query: "latest developments in AI" }        │
│                                                 │
│ Output:                                         │
│   [3 results found...]                          │
└─────────────────────────────────────────────────┘
```

**States**:
- **Collapsed** (default): Shows "› 🔧 {toolName}" + timestamp
- **Expanded**: Shows Input + Output (JSON formatted)

**Styling**:
- Background: `#1E3A3A` (dark teal)
- Border-left: `3px solid #10B981` (green accent)
- Icon: 🔧 (or wrench SVG)
- Code blocks: Monospace font, syntax highlighting

**Interaction**:
- Click to toggle
- Input/Output are syntax-highlighted JSON

---

### 4. TextLog Component (Info/Error)

```
┌─────────────────────────────────────────────────┐
│ ℹ️  Workflow started                   14:32:40  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ ⚠️  Rate limit approaching             14:33:10  │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ ❌ API call failed: Timeout           14:33:15  │
└─────────────────────────────────────────────────┘
```

**Styling**:
- **Info**: Default background, blue icon
- **Warning**: Yellow border + icon
- **Error**: Red border + icon
- Timestamp: right-aligned, muted gray

---

## Interaction Patterns

### Auto-Scroll Behavior
1. **Default**: Auto-scroll ON (new entries scroll into view)
2. **User Scrolls Up**: Auto-scroll disables automatically
3. **User Scrolls to Bottom**: Auto-scroll re-enables
4. **Manual Toggle**: Toolbar button overrides behavior

### Keyboard Shortcuts
- `Cmd/Ctrl + L`: Toggle Log Panel visibility
- `Cmd/Ctrl + K`: Clear logs

### Filter Dropdown
```
⚙️ All ▼
  ├─ All
  ├─ Info
  ├─ Warning
  ├─ Error
  └─ Debug
```
- Changes apply immediately
- Filtered entries fade out (don't remove)

---

## Visual Styling

### Color Palette
| Element | Color | Usage |
|---------|-------|-------|
| Panel BG | `#1A1A1A` | Main background |
| Thinking Block | `#2D2D3D` | Thought entries |
| Tool Block | `#1E3A3A` | Tool calls |
| Error Block | `#3D1E1E` | Errors |
| Border Accent (Thinking) | `#8B5CF6` | Purple |
| Border Accent (Tool) | `#10B981` | Green |
| Border Accent (Error) | `#EF4444` | Red |
| Text | `#E5E7EB` | Primary text |
| Muted Text | `#9CA3AF` | Timestamps |

### Typography
- **Headers**: 14px, Medium (500)
- **Body**: 13px, Regular (400)
- **Code**: 12px, Monospace (Fira Code / Consolas)
- **Timestamps**: 11px, Regular, Muted

### Spacing
- **Block Padding**: 12px vertical, 16px horizontal
- **Block Gap**: 8px between entries
- **Icon-Text Gap**: 8px

---

## Responsive Behavior

### Small Screens (< 768px)
- Panel becomes full-width overlay (slide-up from bottom)
- Toolbar moves to sticky top of panel
- Blocks remain full-width

### Large Screens (> 1024px)
- Optional: Side-by-side panel (right rail) instead of bottom
- User preference in Settings

---

## Performance Considerations

1. **Virtual Scrolling**: Use `react-window` if > 100 entries visible
2. **Entry Limit**: Max 1000 entries; auto-trim oldest
3. **Lazy Render**: Collapsed blocks don't render content until expanded
4. **Debounced Scroll**: 50ms debounce on scroll event listeners

---

## Empty States

### No Logs Yet
```
┌─────────────────────────────────────────────────┐
│              📋 No logs yet                      │
│    Start a workflow to see execution details    │
└─────────────────────────────────────────────────┘
```

### Filtered to Zero
```
┌─────────────────────────────────────────────────┐
│         🔍 No logs match this filter             │
│          Try changing the filter settings        │
└─────────────────────────────────────────────────┘
```

---

## Component File Structure

```
src/components/logs/
├── LogViewer.tsx          # Main container
├── LogViewer.css
├── ThinkingBlock.tsx      # Collapsible thought block
├── ThinkingBlock.css
├── ToolBlock.tsx          # Collapsible tool block
├── ToolBlock.css
├── TextLogEntry.tsx       # Simple text entry
├── TextLogEntry.css
└── index.ts               # Barrel export
```

---

## Design Decisions & Rationale

### Why bottom panel instead of sidebar?
- **Context**: Users read chat vertically; logs are secondary.
- **Efficiency**: Bottom panel maximizes horizontal space for chat.
- **Precedent**: VS Code, Chrome DevTools use bottom panels for logs.

### Why collapsible by default?
- **Scan-ability**: Collapsed = faster scanning of what happened.
- **Detail on Demand**: Expand only when debugging specific steps.

### Why color-coded borders instead of backgrounds?
- **Accessibility**: Borders + icons = multi-sensory cues.
- **Subtlety**: Full background colors can be visually overwhelming.

---

## Future Enhancements (Out of Scope for MVP)

1. **Search**: Cmd+F to search log content
2. **Export**: Download logs as JSON/CSV
3. **Timestamps Toggle**: Show relative ("2s ago") vs absolute ("14:32:45")
4. **Pinning**: Pin specific log entries to top
5. **Log Levels**: Custom log levels beyond Info/Warning/Error

---

## Related Artifacts
- Story: [`5-2-view-execution-log.md`](file:///Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/5-2-view-execution-log.md)
- Tech Spec: [`tech-spec-5-2-view-execution-log.md`](file:///Users/mengbin/code/GPT/CrewAgent/_bmad-output/implementation-artifacts/tech-spec-5-2-view-execution-log.md)
