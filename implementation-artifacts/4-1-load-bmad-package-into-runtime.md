# Story 4.1: Load .bmad Package (v1.1) into Runtime

Status: complete

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime User**,
I want to **import a `.bmad` package (ZIP)** into the Runtime application,
so that I can access the workflows and agents defined in the package for execution.

## Acceptance Criteria

1. **Given** I am on the Start Page
   **When** I click "Import Package"
   **Then** a native file dialog opens allowing me to select `.bmad` or `.zip` files.

2. **Given** I select a valid `.bmad` package
   **When** the import process completes
   **Then** the package is extracted to the Runtime internal storage (`runtime-store/packages/<id>`).
   **And** the package metadata (name, version, schemaVersion) is read from `bmad.json`.
   **And** the list of Workflows is read from `bmad.json` or `workflow.graph.json`.
   **And** the list of Agents is read from `agents.json`.
   **And** the UI redirects to the **Package Page** showing this data.

3. **Given** I have imported a package
   **When** I view the **Package Page**
   **Then** I can see tabs for Overview, Workflows, and Agents populated with the imported data.
   **And** the **StatusBar** displays the active package info.

4. **Given** the package is invalid (missing `bmad.json` or critical files)
   **When** I try to import
   **Then** an error message is displayed and the import fails.

## Design

### Backend (Electron Main)
- **RuntimeStore**: Handles file operations.
  - `importPackage(zipPath)`:
    - Uses `adm-zip` to extract to user data directory.
    - Validates `bmad.json` existence and basic fields.
    - Generates unique ID.
    - Persists package registry in `packages.json`.
- **IPC**: Exposed `package:import` channel.

### Frontend
- **Actions**: `appStore.importPackage()` bridges to IPC.
- **UI**:
  - `StartPage`: Trigger import.
  - `PackagePage`: Bind to `activePackage`.

## Tasks / Subtasks (Dev)

- [x] Install dependencies (`adm-zip`, `ajv`)
- [x] Implement `RuntimeStore` backend logic (extract, validate, metadata)
- [x] Wire IPC handlers in `main.ts` and `preload.ts`
- [x] Update frontend `appStore` and types
- [x] Connect UI pages (`StartPage`, `PackagePage`) to real data
