# Story 5.10: Files Page (Project Explorer)

Status: done

## Story

As a **User**,
I want to **view and manage files in my project**,
so that I can verify generated artifacts, edit configurations, and organize my workflow files.

## Acceptance Criteria

1. **Given** a loaded project
   **When** I navigate to **Files**
   **Then** a **slide-out panel** opens with a file tree rooted at **ProjectRoot** labeled with the project name.

2. **Given** the file tree
   **When** I expand/collapse a directory
   **Then** it reveals child items without blocking the UI (lazy load or depth-limited refresh is acceptable).

3. **Given** a file item
   **When** I click it
   **Then** a **new file tab** opens after the fixed Conversation tab  
   **And** the file tab shows **view/edit** content with a Save action:
   - `.md` renders as Markdown and offers an **Open in OS** action.
   - `.json`, `.yaml`, `.xml` render with syntax highlighting and offer an **Open in OS** action.
   - Unsupported types show a "No preview available" state with an **Open in OS** action.

4. **Given** the Files page
   **When** I use file actions
   **Then** I can **create folder/file**, **rename**, **delete**, and **open in OS** for items under ProjectRoot only.

5. **Given** a file extension
   **When** a viewer is registered for that extension
   **Then** the page uses that viewer; otherwise it falls back to the default "No preview" state.

## Validation Notes
- `RuntimeStore.getFileTree` already exists (see `runtimeStore.ts`).
- IPC channels for `files:getTree`, `files:read`, `files:createFolder`, `files:createFile`, `files:renamePath`, `files:deletePath`, `files:openInOs` are already wired.
- Files UI is **project-only** (no Package/State roots) to match the updated IA.

## Technical Context

- **Scope**: Project-only explorer (no Package/State roots); default focus on `@project/artifacts/` when present.
- **IPC**: All file ops are scoped to ProjectRoot via `RuntimeStore.resolveInsideDir`.
- **UI**: Files panel (tree) + workspace **file tabs** (after fixed Conversation tab).
- **State**: Tree data comes from `RuntimeStore.getFileTree({ root: 'project', projectRoot })`.
- **Tech Spec**: `_bmad-output/implementation-artifacts/tech-spec-5-10-files-page-project-explorer.md`

## Dependencies
- Story 5-0 (Shell/Navigation) - *Done*

## Design

### Summary
- Files is a **Project-context** explorer: tree in a slide-out panel, preview in a **file tab** (multiple tabs supported).
- **Markdown preview first**, with an **extensible viewer registry** for future file types.
- File operations are **read/write within ProjectRoot only**.

### UX / UI
- **Layout**: Slide-out panel (tree) + persistent workspace tabs (fixed Conversation + file tabs).
- **Tree**:
  - Root label = Project name.
  - Expand/collapse folders.
  - Default focus on `@project/artifacts/` when present.
  - Selected item highlight + icon by file type.
- **Preview/Edit**:
  - Markdown renderer or plain text editor for `.md`, with **Open in OS** action.
  - Syntax highlighting viewer for `.json`, `.yaml`, `.xml`.
  - Fallback "No preview available" + Open in OS.
  - Editable for project files; Save writes to `@project`.
- **Actions**:
  - Toolbar or context menu: New File/Folder, Rename, Delete, Open in OS, Save.
  - New File/Folder defaults to `@project/artifacts/` when present.
  - Disable destructive actions for invalid selection.
- **Empty/Error states**:
  - No project: show "Open Project".
  - File read error: show error message in the file tab.

### API / Contracts
- `files:getTree`, `files:read`, `files:write`, `files:createFolder`, `files:createFile`, `files:renamePath`, `files:deletePath`, `files:openInOs`
- All payloads use `root: 'project'` + `projectRoot`.

### Data / Storage
- All paths resolved under ProjectRoot.
- After create/rename/delete, refresh the tree and preserve selection when possible.

### Errors / Edge Cases
- Path escapes ProjectRoot -> surface error and abort.
- Deleted/renamed selected item -> clear selection and preview.
- Large directories -> use `maxDepth` or lazy refresh to keep UI responsive.

### Test Plan

#### Manual Verification
1. Open a project and navigate to **Files** (panel opens).
2. Expand folders; confirm child items render.
3. Click a `.md` file; confirm a file tab opens, shows Markdown content, and exposes **Open in OS**.
4. Edit the file and Save; confirm the file updates on disk.
5. Click a non-md file; confirm fallback "No preview" + Open in OS.
6. Create a folder and file under `artifacts/`; rename and delete them; tree refreshes.
7. Open a file in OS; confirm the system file viewer opens.

#### Automated Tests
- `RuntimeStore.getFileTree` returns correct structure and respects ProjectRoot.
- `createFile/createFolder/renamePath/deletePath` reject paths outside ProjectRoot.

## Tasks / Subtasks

- [x] Update Files page to **project-only** tree (remove Package/State tabs).
- [x] Implement viewer registry + Markdown viewer.
- [x] Implement viewers for JSON, YAML, and XML (syntax highlighting).
- [x] Support Mermaid diagrams in Markdown preview.
- [x] Implement syntax highlighting for standard code blocks in Markdown.
- [x] Wire toolbar/context actions to IPC file ops (including Save/Write).
- [x] Add empty/error states for file selection and read failures.
- [x] Add runtime store tests for sandboxed file operations.
