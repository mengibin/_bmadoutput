# Story 9.2: Enhanced Markdown Editor (Split View + Toolbar)

Status: done

<!-- Note: Code review complete. See code-review-9-2-enhanced-markdown-editor.md -->

## Overview

Enhance the Markdown editing experience in the Runtime to match the Builder's capabilities. This includes adding a formatting toolbar, establishing a split-view layout (Editor + Preview), and ensuring visual consistency with the Builder.

---

## User Story

As a **Consumer**,
I want to edit Markdown files in the Runtime Files view using an experience similar to the Builder,
So that I have a consistent and powerful editing environment.

---

## Acceptance Criteria

### AC-1: Toolbar
**Given** I open a `.md` file and switch to **Edit** mode
**When** the Markdown editor loads
**Then** I see a toolbar with formatting buttons:
  - Bold (`**txt**`)
  - Italic (`*txt*`)
  - Link (`[txt](url)`)
  - List (Bullet/Number)
  - Table (Table template)
  - Code Block
**And** clicking a button applies the formatting to the selection or inserts a template

### AC-2: Split View
**Given** I am editing a markdown file (Edit mode)
**Then** the default layout is Split View (Editor Left, Preview Right)
**And** I can toggle between "Split", "Editor Only", and "Preview Only" (optional enhancement)

### AC-3: Real-time Preview
**Given** I type in the editor
**Then** the preview pane updates in real-time
**And** the preview renders GFM (GitHub Flavored Markdown) including tables and code blocks

### AC-4: Visual Consistency
**Given** I use the Runtime Editor
**Then** the fonts, colors, and spacing match the Builder's `MarkdownEditorModal`

---

## Validation Report

**Validated**: 2026-01-27
**Verdict**: ✅ **Specs & Design Approved**

### 1. Integrity Check
- **Consistency**: High alignment with Builder requested by user.
- **Components**: Reuse strategy is sound.

### 2. Implementation Risks
- **Dependencies**: Need to ensure `react-markdown` is available or installed.
- **Styles**: Tailwind configuration differences between Builder/Runtime might require adjustments.

---

## Design Specification

### 1. Design Principles
- **Porting Strategy**: Reuse `MarkdownEditorModal` logic from `crewagent-builder-frontend`.
- **UX**: "WYSIWYG-ish" experience with instant preview.
- **Consistency**: Shared icons (Lucide) and color palette (Zinc).

### 2. Change Scope
| Component | Change | Description |
|-----------|--------|-------------|
| `package.json` | MODIFY | Add `react-markdown`, `remark-gfm`. |
| `MarkdownEditor` | NEW | Main editor component with state. |
| `MarkdownPreview` | NEW | Preview component (GFM renderer). |
| `Toolbar` | NEW | Formatting buttons (inside `MarkdownEditor`). |
| `WorkspacePage` | MODIFY | In Edit mode, render `MarkdownEditor` for `.md` files. |

### 3. Data Structures
```typescript
interface MarkdownEditorProps {
    value: string
    onChange: (value: string) => void
    readOnly?: boolean
    filePath?: string
}
```

### 4. Data Flow
```
WorkspacePage File Tab (Edit) -> MarkdownEditor
    ├── Toolbar (onClick -> insert markdown syntax)
    ├── Generic Textarea (onChange -> update state)
    └── MarkdownPreview (prop: value -> render HTML)
```

---

## Implementation Steps

1. **Dependencies**: `npm install react-markdown remark-gfm`.
2. **Port Components**:
   - Copy `MarkdownPreview.tsx` from Builder.
   - Re-implement `MarkdownEditor` logic (toolbar actions, wrapping selection).
3. **Styling**: Ensure split pane layout (Grid/Flex) works in Runtime container.
4. **Integration**: In `.md` file tab **Edit** mode, render `MarkdownEditor` instead of a plain textarea.

---

## Impact Analysis

| Component | Change | Risk |
|:----------|:-------|:-----|
| `WorkspacePage` | Component swap | Low |
| `CSS` | Global styles | Low (Scoped to component) |

---

## Tasks / Subtasks

- [x] Install `react-markdown`, `remark-gfm` <!-- id: 0 -->
- [x] Implement `MarkdownPreview` (Ported) <!-- id: 1 -->
- [x] Implement `MarkdownEditor` (State + Toolbar) <!-- id: 2 -->
- [x] Implement Layout (Split View) <!-- id: 3 -->
- [x] Integrate into Workspace file editor (Edit mode) <!-- id: 4 -->

---

## File List

- `crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.tsx`
- `crewagent-runtime/src/components/MarkdownEditor/MarkdownEditor.css`
- `crewagent-runtime/src/components/MarkdownEditor/MarkdownPreview.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
