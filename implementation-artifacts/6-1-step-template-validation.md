# Story 6.1: Step File Template Validation & UI Display

Status: done

## Story

As a **Package Author**,
I want the import process to validate that my step files follow the standard template format,
so that I can catch format errors early and ensure my workflows will work correctly in the Runtime.

As a **Builder User**,
I want the Step Editor UI to display step content organized by sections (Goal, Instructions, Completion),
so that I can easily understand and edit each part of the step.

## Background

BMAD Package Spec v1.1 defines a standard template for step files (`templates/step.md`) with required sections:
- YAML Frontmatter (schemaVersion, nodeId, type)
- `## Goal` section
- `## Instructions` section
- `## Completion` section (optional)

Currently, the import service stores step files as raw markdown without validating their structure. This leads to inconsistent step files and potential runtime errors. Additionally, the UI displays step content as a single text block instead of organized sections.

## Acceptance Criteria

### Backend Validation

1. **[AC-1] Frontmatter Validation**
   - GIVEN a step file being imported
   - WHEN the YAML frontmatter is parsed
   - THEN it MUST contain required fields: `schemaVersion`, `nodeId`, `type`
   - AND validation errors are reported if any required field is missing

2. **[AC-2] Required Section Validation**
   - GIVEN a step file with valid frontmatter
   - WHEN the markdown body is parsed
   - THEN it MUST contain `## Goal` section
   - AND it MUST contain `## Instructions` section
   - AND validation errors are reported if any required section is missing

3. **[AC-3] Section Content Parsing**
   - GIVEN a step file with valid structure
   - WHEN sections are parsed
   - THEN each section's content is extracted (between header and next header or EOF)
   - AND empty sections generate warnings

4. **[AC-4] Import Failure on Validation Error**
   - GIVEN a package with invalid step files
   - WHEN import is attempted
   - THEN the import FAILS with detailed error messages listing each validation error
   - AND errors include file path, line number (if possible), and description

5. **[AC-5] Spec Documentation Update**
   - The v1.1 README.md MUST document the required step file structure
   - Including YAML frontmatter required fields
   - Including required markdown sections (Goal, Instructions)

### Frontend UI Display

6. **[AC-6] Section-Based Display**
   - GIVEN a step is selected in the Builder UI
   - WHEN the step editor panel loads
   - THEN it displays content organized by sections:
     - **Goal** section content (collapsible)
     - **Instructions** section content (main editable area)
     - **Completion** section content (collapsible, if exists)

7. **[AC-7] Frontmatter Display**
   - GIVEN a step with valid frontmatter
   - WHEN displayed in the editor
   - THEN show frontmatter fields in a structured form:
     - nodeId (read-only)
     - title (editable)
     - type (dropdown: step, decision, merge, end)
     - agentId (dropdown from available agents)
     - inputs/outputs (tag input)

8. **[AC-8] Section Editing**
   - GIVEN the user edits a section
   - WHEN changes are saved
   - THEN the markdown file is reconstructed with updated section content
   - AND original formatting is preserved where possible

## Design

> Required before development. Run `design-story` to complete this section and set status to `ready-for-dev`.

### API / Contracts

**Backend:**
- **New function**: `validate_step_file(content: str, file_path: str) -> list[ValidationError]`
- **New function**: `parse_step_sections(content: str) -> StepSections`
- Located in: `app/services/package_import_service.py`

**Frontend:**
- **Parser function**: `parseStepMarkdown(content: string) -> StepData`
- Located in: `src/lib/step-parser.ts`
- Returns: `{ frontmatter: {...}, goal: string, instructions: string, completion?: string }`

### Data / Storage

- No schema changes - step files still stored as complete markdown in `step_files_json`
- Sections parsed at display time by frontend

### UI Components

| Component | Description |
|:---|:---|
| `StepEditorPanel` | Main editor for step files, replaces current textarea |
| `FrontmatterForm` | Structured form for editing frontmatter fields |
| `SectionEditor` | Markdown editor for each section (Goal, Instructions, Completion) |
| `SectionCollapse` | Collapsible section wrapper |

### Errors / Edge Cases

| Error Code | Description |
|:---|:---|
| `STEP_MISSING_FRONTMATTER` | Step file has no YAML frontmatter |
| `STEP_INVALID_FRONTMATTER` | Frontmatter missing required fields |
| `STEP_MISSING_GOAL` | No `## Goal` section found |
| `STEP_MISSING_INSTRUCTIONS` | No `## Instructions` section found |
| `STEP_EMPTY_SECTION` | Warning: section header exists but content is empty |

### Test Plan

- Unit tests for `validate_step_file()` and `parseStepMarkdown()`
- Integration test: import package with valid step files → success
- Integration test: import package with missing frontmatter → error
- UI test: step editor correctly displays sections

## Tasks / Subtasks

### Phase 1: Spec & Validation

- [x] **Task 1**: Update BMAD Package Spec v1.1 README.md (AC: #5)
  - [x] Add "Step File Structure" section
  - [x] Document YAML frontmatter required fields
  - [x] Document required markdown sections

- [x] **Task 2**: Implement step file validation (AC: #1, #2, #3)
  - [x] Add `validate_step_file()` function
  - [x] Parse YAML frontmatter and check required fields
  - [x] Parse markdown sections
  - [x] Return list of validation errors

- [x] **Task 3**: Integrate validation into import flow (AC: #4)
  - [x] Call `validate_step_file()` for each step file
  - [x] Aggregate errors and include in PackageImportError

### Phase 2: Example Update

- [x] **Task 4**: Update finance-operations example (AC: #1-4)
  - [x] Rewrite all 9 step files to follow template format
  - [x] Add Goal section to each step
  - [x] Repackage .bmad file

### Phase 3: Frontend UI

- [x] **Task 5**: Create step markdown parser (AC: #6)
  - [x] Implement `parseStepMarkdown()` in `src/lib/step-parser.ts`
  - [x] Parse frontmatter, Goal, Instructions, Completion sections
  - [x] Add unit tests

- [x] **Task 6**: Create StepEditorPanel component (AC: #6, #7, #8)
  - [x] Create `StepEditorPanel.tsx` component
  - [x] Implement FrontmatterForm for structured editing
  - [x] Implement SectionEditor for each markdown section
  - [x] Add save functionality to reconstruct markdown

- [x] **Task 7**: Integrate into Builder page (AC: #6)
  - [x] Replace current step textarea with StepEditorPanel
  - [x] Connect to existing step save API

### Phase 4: Test & Verify

- [x] **Task 8**: End-to-end testing
  - [x] Test import with valid/invalid packages
  - [x] Test UI editing and saving
  - [x] Verify error messages are clear

## Dev Notes

### Relevant Files

**Backend:**
- `crewagent-builder-backend/app/services/package_import_service.py`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/README.md`
- `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/step.md`

**Frontend:**
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/lib/step-parser.ts` (new)
- `crewagent-builder-frontend/src/components/StepEditorPanel.tsx` (new)

### Implementation Notes

- Use `yaml` library (backend) or `js-yaml` (frontend) to parse frontmatter
- Regex to find section headers: `^##\s+(Goal|Instructions|Completion)`
- Frontend should gracefully handle legacy step files without sections

### References

- [Source: spec/bmad-package-spec/v1.1/templates/step.md] - Standard step template
- [Source: spec/bmad-package-spec/v1.1/schemas/step-frontmatter.schema.json] - Frontmatter schema
