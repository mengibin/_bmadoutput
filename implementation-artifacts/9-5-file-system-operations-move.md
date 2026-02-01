# Story 9.5: File System Operations (Move)

Status: done

<!-- Note: Implemented in RuntimeStore + FilesPanel drag-and-drop. Unit tests pass. -->

## Overview

Enable users to reorganize their project structure by moving files and folders via drag-and-drop within the Runtime file explorer.

---

## User Story

As a **Consumer**,
I want to move files and folders,
So that I can reorganize my project.

---

## Acceptance Criteria

### AC-1: Drag and Drop
**Given** I drag a file or folder in the project tree
**When** I drop it onto another folder
**Then** the item is moved to the target folder
**And** the file tree updates immediately

### AC-2: Conflict Handling
**Given** a file with the same name exists in the destination
**When** I drop the item
**Then** the operation is blocked
**And** an error message or conflict dialog is shown (e.g., "File 'x' already exists")

### AC-3: Recursive Move
**Given** I move a folder
**When** the operation completes
**Then** the folder and all its contents are moved to the new location

---

## Validation Report

**Validated**: 2026-01-27
**Verdict**: ✅ **Specs & Design Approved**

### 1. Integrity Check
- **User Needs**: Organization is a core feature.
- **Constraints**: Destination conflict handling is essential.

### 2. Implementation Risks
- **Data Loss**: Overwrite without warning is the biggest risk. `fs.rename` behavior needs careful checking across platforms.
- **Self-Move**: Moving a folder into itself must be blocked.

---

## Design Specification

### 1. Design Principles
- **Direct Manipulation**: Drag and Drop is intuitive.
- **Safety**: Prevent accidental overwrites and illegal moves (e.g., parent into child).
- **Feedback**: Highlight drop target.

### 2. Change Scope
| Component | Change | Description |
|-----------|--------|-------------|
| `Main Process` | MODIFY | Add `fs:move-path` handler. |
| `FileTreeItem` | MODIFY | Make draggable, handle drop events. |
| `drag-and-drop` | LOGIC | Handle `dataTransfer` (source path). |

### 3. Data Flow
1. **Renderer**: `onDragStart` -> Set `dataTransfer.setData('path', src)`.
2. **Renderer**: `onDragOver` -> Visual feedback (border).
3. **Renderer**: `onDrop` -> Get `src`, `dest`.
4. **Validation (Renderer)**: Verify basic rules (not moving into self).
5. **IPC**: `window.electron.movePath(src, dest)`.
6. **Main**:
   - Validate sandbox.
   - Check if `dest/basename(src)` exists -> Error if yes.
   - `fs.rename(src, dest/basename(src))`.
7. **Result**: Success -> Refresh / Fail -> Alert.

---

## Implementation Steps

1. **Main Process**: Implement `fs:move-path`.
2. **Renderer UI**:
   - Add `draggable` to items.
   - Implement `onDragStart`, `onDragOver` (preventDefault), `onDrop`.
   - Calculate correct destination path (folder vs file drop).
3. **Validation**: Check for conflicts before sending IPC or let IPC handle it.
4. **Feedback**: Add visual cues (CSS).

---

## Impact Analysis

| Component | Change | Risk |
|:----------|:-------|:-----|
| `Main Process` | Move Capability | Medium (File integrity) |
| `UI` | DnD Interactions | Low |

---

## Tasks / Subtasks

- [x] Implement `files:move` IPC (Main) <!-- id: 0 -->
- [x] Implement Sandbox validation (Main) <!-- id: 1 -->
- [x] Add Draggable/Droppable attributes to Tree Items <!-- id: 2 -->
- [x] Implement Drop Handler (Renderer) <!-- id: 3 -->
- [x] Error Reporting (Toast/Dialog) <!-- id: 4 -->

---

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/src/hooks/useFileExplorer.ts`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.css`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
