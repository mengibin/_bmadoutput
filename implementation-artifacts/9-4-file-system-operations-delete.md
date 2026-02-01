# Story 9.4: File System Operations (Delete)

Status: done

<!-- Note: Code review complete. See code-review-9-4-file-system-operations-delete.md -->

## Overview

Enable users to delete files and directories directly from the Runtime file explorer context menu, with a confirmation dialog to prevent accidental data loss.

---

## User Story

As a **Consumer**,
I want to delete files and folders from the file explorer,
So that I can manage my project structure.

---

## Acceptance Criteria

### AC-1: Context Menu Delete
**Given** I right-click a file or folder in the project tree
**When** the context menu appears
**Then** I see a "Delete" option (highlighted in red or distinct)

### AC-2: Confirmation Dialog
**Given** I select "Delete"
**When** the action is triggered
**Then** a modal dialog appears asking "Are you sure you want to delete 'filename'?"
**And** "Cancel" is the default focus (safety)
**And** "Delete" button performs the action

### AC-3: Directory Deletion
**Given** I am deleting a folder
**When** I confirm deletion
**Then** the folder and all its contents are deleted recursively (`rm -rf`)

### AC-4: UI Update
**Given** the deletion is successful
**When** the operation completes
**Then** the item disappears from the file explorer immediately (handled by Story 9.3 watcher or manual callback)

---

## Validation Report

**Validated**: 2026-01-27
**Verdict**: ✅ **Specs & Design Approved**

### 1. Integrity Check
- **Safety**: Confirmation dialog and recursion support are confirmed.
- **Constraints**: Sandbox check is a hard requirement.

### 2. Implementation Risks
- **Path Traversal**: Malicious paths must be blocked.
- **System Files**: Should prevent deleting project root or `.git` (optional but recommended).

---

## Design Specification

### 1. Design Principles
- **Safety First**: "Confirm before destructive action".
- **Sandbox**: Restrict operations to `projectRoot`.
- **Feedback**: Immediate UI update.

### 2. Change Scope
| Component | Change | Description |
|-----------|--------|-------------|
| `Main Process` | MODIFY | Harden `files:delete` with project-root guard; make confirm dialogs default to Cancel. |
| `FilesPage` | MODIFY | Add right-click Context Menu "Delete" action in file tree. |
| `useFileExplorer` | MODIFY | Add `deletePath(path)` helper and reuse confirm dialog. |

### 3. Data Flow
1. **User**: Right-click -> "Delete".
2. **Renderer**: `ProjectExplorer` shows Context Menu.
3. **User**: Click "Delete" -> Show `ConfirmDialog`.
4. **User**: Click "Confirm".
5. **Renderer**: `window.ipcRenderer.deletePath({ path, projectRoot })`.
6. **Main**:
   - Validate `path` starts with `projectRoot`.
   - Call `fs.rm(path, { recursive: true })`.
7. **Result**: Success -> Renderer refresh / Fail -> Toast Error.

---

## Implementation Steps

1. **Main Process**: Implement `fs:delete-path` handler with sandbox check.
2. **Context Menu**: Add right-click menu to `FileTreeItem`.
3. **Dialog**: Create `ConfirmDialog` component.
4. **Integration**: Wire up the "Delete" action to the Dialog, then to the IPC call.

---

## Impact Analysis

| Component | Change | Risk |
|:----------|:-------|:-----|
| `Main Process` | Delete Capability | High (Data Loss) |
| `UI` | Context Menu | Low |

---

## Tasks / Subtasks

- [x] Implement `files:delete` guardrails (Main) <!-- id: 0 -->
- [x] Implement Sandbox validation (Main) <!-- id: 1 -->
- [x] Add Context Menu "Delete" item (Renderer) <!-- id: 2 -->
- [x] Implement Confirmation Dialog (Native `dialog:confirm`) <!-- id: 3 -->
- [x] Connect Logic <!-- id: 4 -->

---

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/hooks/useFileExplorer.ts`
- `crewagent-runtime/src/pages/FilesPage/FilesPage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
