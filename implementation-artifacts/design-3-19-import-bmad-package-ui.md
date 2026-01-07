# Design: Import `.bmad` Package UI (Builder)

**Story:** `_bmad-output/implementation-artifacts/3-19-import-bmad-package-into-builder.md`
**Tech Spec:** `_bmad-output/implementation-artifacts/tech-spec-3-19-import-bmad-package-into-builder.md`

## Goals

- Provide a simple, discoverable entry point to import `.bmad` packages.
- Show clear progress + actionable validation errors (file path + schema pointer).
- Handle name conflicts without losing the user’s selected file.

## Entry Point

- Location: Builder Dashboard (`/dashboard`)
- Primary CTA: **Import Package**

## Modal Layout

### 1) Upload State

- File picker restricted to `.bmad`
- Selected file name shown
- Actions:
  - **Cancel**
  - **Import** (disabled until file selected)

![Import Package Modal](import_package_modal_1767614046988.png)

### 2) Progress State

- Inline progress text:
  - Uploading…
  - Validating…
  - Importing…

### 3) Validation Error State

- Show:
  - Error message summary
  - A list of validation issues: `code`, `file`, `path`, `message`
- Copyable/scrollable area recommended.

![Validation Error Display](validation_error_display_1767614101820.png)

### 4) Name Conflict State

- When `bmad.json.name` conflicts with an existing project name:
  - Provide suggested name (e.g. `-imported`)
  - Allow user to confirm a new name and retry import
  - Allow cancel

![Conflict Resolution Dialog](conflict_resolution_dialog_1767614121698.png)

## UX Notes

- Keep validation errors grouped by file (if feasible) to reduce scanning cost.
- Always preserve the selected file while user resolves errors (name conflict / validation issues).
- Success → redirect to new project: `/builder/{projectId}`.

