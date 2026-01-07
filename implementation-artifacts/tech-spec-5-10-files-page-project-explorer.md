# Tech Spec: Story 5.10 - Files Page (Project Explorer)

**Story**: [5-10-files-page-project-explorer.md](5-10-files-page-project-explorer.md)
**Status**: Implemented

## 1. Objective
Implement the Files page as a project-only explorer: browse ProjectRoot with default focus on `@project/artifacts/` when present, preview/edit Markdown (with Open in OS), and perform basic file operations via the RuntimeStore. A viewer registry enables future file type extensions.

## 2. Technical Architecture

### 2.1 Backend (RuntimeStore)
- **Existing Capabilities**:
  - `getFileTree`, `readFile`, `writeFile`, `createFolder`, `createFile`, `renamePath`, `deletePath`, `openInOs`
- **Requirements**:
  - All operations are scoped to ProjectRoot via `resolveInsideDir`.
  - `getFileTree` supports `maxDepth` (lazy loading optional for large trees).

### 2.2 IPC Layer (`preload.ts` / `main.ts`)
- Use existing handlers:
  - `files:getTree`, `files:read`, `files:write`, `files:createFolder`, `files:createFile`, `files:renamePath`, `files:deletePath`, `files:openInOs`
- Renderer calls pass `root: 'project'` and `projectRoot`.
- Do not expose real filesystem paths to the renderer.

### 2.3 Frontend (React)
- **FilesPanel**: Slide-out tree panel inside Project context.
- **FileTreeView**: Recursive tree with expand/collapse and selection highlight.
- **File Tabs**: Workspace tabs (after fixed Conversation tab) that use a `ViewerRegistry` keyed by file extension and support edit/save for project files.
  - Initial viewer: Markdown (`.md`).
  - Markdown viewer also exposes **Open in OS**.
  - Fallback viewer: "No preview available" + Open in OS.
- **FileActions**: Toolbar or context menu for create/rename/delete/open/save, defaulting new file/folder actions to `@project/artifacts/` when present.

## 3. Data Flow
1. Files panel opens -> `window.ipcRenderer.getFileTree({ root: 'project', projectRoot })`.
2. Tree renders ProjectRoot and focuses `@project/artifacts/` when present.
3. File click -> `window.ipcRenderer.readFile({ path, projectRoot })`.
4. File tab renders based on extension or fallback.
5. File edit -> `files:write` -> refresh tree if needed.
6. File operation -> IPC call -> refresh tree.

## 4. Testing Strategy (Crucial)

### 4.1 Automated Tests (Unit)
*Target: `electron/stores/runtimeStore.test.ts`*
- **File Tree Generation**
  - Setup: Create temp dir structure.
  - Action: Call `getFileTree`.
  - Assertion: Tree matches FS; node types correct.
- **Sandbox Security**
  - Action: Attempt read/write outside ProjectRoot.
  - Assertion: Reject with error "Path escapes destination dir".
- **File Operations**
  - Action: `createFile`, `createFolder`, `renamePath`, `deletePath`.
  - Assertion: FS state matches expectations.

### 4.2 Manual Verification (QA)
1. Open a project and navigate to **Files** (panel opens).
2. Expand folders; confirm child items render.
3. Click a `.md` file; confirm a file tab opens and **Open in OS** is available.
4. Edit and Save; confirm file updates on disk.
5. Click a non-md file; confirm fallback "No preview" + Open in OS.
6. Create, rename, and delete items under ProjectRoot; tree refreshes.

## 5. Implementation Plan
1. Update FilesPage to project-only tree (remove Package/State roots).
2. Introduce viewer registry + Markdown viewer.
3. Wire toolbar/context actions to IPC file ops (including write).
4. Add empty/error states and tree refresh.
5. Add runtime store tests for sandboxed file ops.
