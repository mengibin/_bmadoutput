# Story 5.21: Collapsible Workspace Context Panels

Status: review

## Story

As a **CrewAgent Runtime user**,
I want to **hide and reopen the middle context panel for Files, Knowledge, and Works**,
so that I can give the main workspace more room without losing the context I was working in.

## Acceptance Criteria

### AC-1: Unified toggle for all three panels

**Given** I am on **Files**, **Knowledge**, or **Works**
**When** the context panel is visible
**Then** a shared collapse control is displayed as the first control at the far left of the main workspace top row
**And** it appears before the Conversation tab for Files and Works, with an equivalent leading position in the Knowledge main header
**And** its tooltip and accessible label describe the hide action in the active UI language.

### AC-2: Collapse expands the main workspace

**Given** the context panel is visible
**When** I activate the collapse control
**Then** the panel is hidden with a short transition
**And** the main workspace consumes the released width
**And** a compact, discoverable restore control remains visible.

### AC-3: Reopen preserves panel context

**Given** the context panel is hidden
**When** I activate the restore control
**Then** the same panel opens again
**And** the mounted panel keeps its existing in-memory context, including file-tree selection/expansion and active Knowledge or Works state.

### AC-4: Navigation opens the destination panel

**Given** one context panel is hidden
**When** I navigate to another one of Files, Knowledge, or Works
**Then** the destination panel opens by default
**And** its main content remains usable.

### AC-5: Accessibility, localization, and responsive behavior

**Given** I use keyboard navigation, Simplified Chinese, English, light theme, dark theme, or a narrow window
**When** I use the panel toggle
**Then** the control remains keyboard reachable and exposes `aria-expanded`/`aria-controls`
**And** hidden panel controls are not keyboard focusable
**And** the toggle remains aligned with the actual panel edge.

## UX / Design Direction

- Preserve the existing restrained, utility-first Runtime visual language.
- Use a compact top-aligned utility button inside the main workspace rather than floating over the panel boundary.
- Place it before the Conversation tab for Files and Works, and before the main Knowledge heading for the equivalent no-tab layout.
- Use `PanelLeftClose` / `PanelLeftOpen` icons and existing surface, border, primary, shadow, and focus tokens.
- Keep the transition short and spatially clear; do not unmount the panel during collapse.
- Hide the handle while Conversation fullscreen mode is active.

## Technical Notes

- `WorkspacePage` is the common shell for `/files`, `/knowledge`, and `/works`; implement shared visibility state there.
- Reset visibility to open when the route changes to a different context panel.
- Keep `FilesPanel`, `KnowledgePanel`, and `WorksPanel` mounted while hidden so local state is retained.
- Add bilingual strings under `workspace` and rely on the existing resource-parity test.
- The feature is renderer-only; no Electron IPC or persisted data schema changes are required.

## Tasks / Subtasks

- [x] Add a reusable accessible workspace panel toggle.
- [x] Add shared open/collapsed state for Files, Knowledge, and Works.
- [x] Apply collapse/reopen behavior to both the Knowledge layout and the tabbed Files/Works layout.
- [x] Add responsive, theme-compatible boundary-handle styling.
- [x] Add Simplified Chinese and English labels.
- [x] Add automated coverage for open/collapsed toggle semantics and translation parity.
- [x] Run focused tests, full test suite, lint, and production renderer build.
- [x] Update this Story with completion notes and final file list.

## Test Plan

### Automated

- Toggle renders correct open/closed icon, `aria-expanded`, label, and controlled panel id.
- Translation catalogs remain in parity and contain no empty values.
- Runtime TypeScript/build, lint, and test suites pass.

### Manual

1. Open Files, collapse the file tree, and confirm the file viewer expands.
2. Reopen Files and confirm selection/tree state remains.
3. Repeat collapse/reopen for Knowledge and Works.
4. Collapse one panel, navigate to another, and confirm the destination opens.
5. Verify Chinese/English labels, light/dark themes, keyboard focus, and narrow-window alignment.

## File List

- `_bmad-output/epics.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `_bmad-output/implementation-artifacts/5-21-collapsible-workspace-context-panels.md`
- `_bmad-output/implementation-artifacts/validation-report-story-5-21.md`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePage.css`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePanelToggle.tsx`
- `crewagent-runtime/src/pages/WorkspacePage/WorkspacePanelToggle.test.tsx`
- `crewagent-runtime/src/i18n/resources/en-US.ts`
- `crewagent-runtime/src/i18n/resources/zh-CN.ts`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
- `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.css`

## Change Log

- 2026-07-20: Story created and moved to `in-progress` for unified Files/Knowledge/Works panel collapse support.
- 2026-07-20: Implemented shared collapse/reopen behavior, localization, accessibility semantics, responsive styling, and regression tests; moved Story to `review`.
- 2026-07-20: User review requested a Codex-style top utility control instead of the mid-height boundary handle; Story returned to `in-progress`.
- 2026-07-20: Replaced the mid-height handle with a compact 32×32 top utility control, added open/collapsed content clearance, revalidated the change, and returned the Story to `review`.
- 2026-07-20: User review requested moving the control fully into the right main panel, immediately before the Conversation tab; Story returned to `in-progress`.
- 2026-07-20: Embedded the toggle as the first Files/Works Tab-bar control and the leading Knowledge header control, removed boundary-overlay positioning, revalidated, and returned the Story to `review`.

## Dev Agent Record

### Implementation Summary

- Added one shared top-aligned utility toggle used as the first main-panel control by Files, Knowledge, and Works.
- Files and Works render it immediately before Conversation; Knowledge renders it before the main heading because that view has no Tab bar.
- Kept panels mounted while collapsed so local UI state is retained.
- Reopens the destination panel automatically when navigating between context-panel routes.
- Added hidden-state focus isolation, fullscreen suppression, reduced-motion support, and responsive panel-edge alignment.

### Verification

- Focused toggle, Knowledge, and i18n tests: passed (3 files / 10 tests).
- Full Runtime test suite: 761/762 passed under default parallel execution; the unrelated `shell.exec` case passed both targeted rerun and its complete 97-test file rerun.
- `npm run lint`: passed with zero warnings.
- `npx tsc --noEmit`: passed.
- `npm run build:ci`: passed.
- `git diff --check`: passed.

### Review Note

- Automated acceptance coverage is complete. The user-requested main-panel placement is ready for desktop-window review before moving the Story from `review` to `done`; the documented parallel-only `shell.exec` test flake is outside this renderer layout change.
