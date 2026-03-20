# Story 13.7: Effective Skill Source Composition & User-Global Scope

Status: done

## Story

As a **Runtime User**,  
I want Runtime to resolve `user-global` and `session` skill sources into one effective set for each turn,  
So that my default reusable skills are available across projects while session inputs can extend or override them predictably.

## Acceptance Criteria

### AC-1: User-global skill availability

**Given** Runtime has a configured user-global skills directory under its private state  
**When** chat, agent, or run mode initializes session context  
**Then** Runtime auto-discovers valid Claude Code skills from that directory  
**And** exposes them to all projects by default without requiring per-session attachment  
**And** excludes invalid global skills with structured diagnostics rather than failing the session

### AC-2: Deterministic source layering

**Given** user-global and session-attached skill sources are both present  
**When** Runtime builds the effective skill source list  
**Then** it composes them with deterministic precedence `session > user-global`  
**And** deduplicates identical resolved sources  
**And** emits structured diagnostics for `skillId` or source collisions instead of silently changing precedence

### AC-3: Shared downstream pipeline with future package hook

**Given** Story 13.1 already defines the skill discovery and registry injection path  
**When** the effective skill source list is produced  
**Then** Runtime reuses the Story 13.1 discovery and registry pipeline without introducing a parallel skill loading path  
**And** chat, agent, and run mode all consume the same source-composition service  
**And** Runtime leaves a package-layer extension hook for Story 13.5 without implementing package imports in this story

### AC-4: User-global skill import and management UI

**Given** Runtime supports a user-global skills directory under private state  
**When** the user opens Runtime settings  
**Then** the UI lists current imported skills in Runtime settings  
**And** the user can import a valid skill directory or direct `SKILL.md`-based skill into that root through a file/folder picker  
**And** the UI can remove imported user-global skills  
**And** imported skills are copied into Runtime-managed storage rather than referenced from arbitrary external paths

## Technical Notes

- This story is inserted after Story 13.1 so source layering and precedence are stabilized before activation and resource loading expand the skill surface.
- Story 13.7 only delivers `user-global` and `session` composition; package integration stays deferred to Story 13.5.
- User-global skills are runtime-managed defaults, not project-owned assets.
- Story 13.7 now includes the first end-user management surface for user-global skills so the feature is not hidden behind manual filesystem edits.
- Design: `_bmad-output/implementation-artifacts/design-13-7-effective-skill-source-composition-and-user-global-scope.md`

## Dependencies

- Story 13.1 must be `done`.

## BMAD Workflow Note

- This story is the required Epic 13 bridge between Story 13.1 and Story 13.2.
- Story 13.2 returns to `blocked` until this source-composition slice reaches `done`.
- Keep the implementation focused on `user-global + session` source layering plus the minimal Runtime UI/IPC needed to import and manage user-global skills; activation, resource loading, and package schema parsing stay in downstream stories.

## Tasks / Subtasks

- [x] Task 1: Runtime-managed user-global source layer (AC: 1,3)
  - [x] 1.1 Define the user-global skills root under Runtime private state
  - [x] 1.2 Discover valid candidate skill directories or direct `SKILL.md` entries from that root
  - [x] 1.3 Treat a missing global root as an empty layer rather than a fatal error

- [x] Task 2: Effective skill source provider (AC: 2,3)
  - [x] 2.1 Add a shared source-composition service for `user-global` and `session`
  - [x] 2.2 Enforce precedence `session > user-global`
  - [x] 2.3 Deduplicate exact source duplicates and emit structured collision diagnostics
  - [x] 2.4 Preserve source-layer metadata needed for later registry collision handling and audit

- [x] Task 3: Multi-mode integration (AC: 1,2,3)
  - [x] 3.1 Use the source provider in chat entrypoints
  - [x] 3.2 Use the source provider in agent entrypoints
  - [x] 3.3 Use the source provider in run and prompt-dump entrypoints
  - [x] 3.4 Keep Story 13.1 discovery and registry injection as the only downstream skill loading path

- [x] Task 4: Future package extension hook (AC: 3)
  - [x] 4.1 Reserve a package-layer input slot for Story 13.5 adapter output
  - [x] 4.2 Keep package-layer input inactive and optional until `skills.imports` lands
  - [x] 4.3 Document how Story 13.5 will plug package sources into the same provider without bypassing it

- [x] Task 5: Tests (AC: 1,2,3)
  - [x] 5.1 Global-only skill discovery without session attachment
  - [x] 5.2 Session-over-global precedence regression tests
  - [x] 5.3 Duplicate-source and `skillId` collision diagnostics tests
  - [x] 5.4 Chat / agent / run parity tests for effective source resolution

- [x] Task 6: User-global skill import and management UI (AC: 4)
  - [x] 6.1 Add IPC to list the user-global skills root and current imported entries
  - [x] 6.2 Add import flow to copy a selected skill directory or direct `SKILL.md` skill into Runtime-managed global storage
  - [x] 6.3 Add remove actions for imported user-global skills
  - [x] 6.4 Add a Runtime settings UI section for global skills with list/import/remove actions
  - [x] 6.5 Add verification for import copy semantics and renderer state refresh

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/services/globalSkillService.ts` (new)
- `crewagent-runtime/electron/services/globalSkillService.test.ts` (new)
- `crewagent-runtime/electron/services/skillSourceProvider.ts` (new)
- `crewagent-runtime/electron/services/skillSourceProvider.test.ts` (new)
- `crewagent-runtime/electron/services/skillRegistryService.ts`
- `crewagent-runtime/electron/services/skillRegistryService.test.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`
- `_bmad-output/implementation-artifacts/13-7-effective-skill-source-composition-and-user-global-scope.md`

## Dev Notes (2026-03-19)

- Added a runtime-managed global skills root at `runtime-store/skills/global/` and extended `RuntimeStore.resolveAbsolutePath()` so `@state/skills/global/...` resolves there instead of project-scoped state.
- Introduced `SkillSourceProvider` as the shared composition layer for `session + user-global`, with deterministic precedence `session > user-global`, stable user-global enumeration order, duplicate-source diagnostics, and an inactive package hook reserved for Story 13.5.
- Reused the existing Story 13.1 discovery/registry pipeline by resolving effective attached skills before registry generation in chat, agent, run, and debug prompt-dump flows instead of creating a parallel loader.
- Extended `SkillRegistryService` to preserve source-layer metadata and to emit `SKILL_ID_COLLISION` diagnostics when different sources expose the same visible `skillId`.
- There is still no dedicated main-process IPC test harness for `main.ts`; parity across chat / agent / run is enforced through the shared provider path and covered by provider/registry/runtime-store regression tests plus targeted linting.
- Added `GlobalSkillService` plus main/preload/store wiring so Runtime can list, import, and remove user-global skills through IPC while keeping all imports copied into runtime-managed storage.
- Added a `Global Skills` section to `SettingsPage` that focuses on imported entry status plus import/remove actions without exposing skill-library filesystem controls.

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/globalSkillService.test.ts electron/services/skillSourceProvider.test.ts electron/services/skillRegistryService.test.ts electron/stores/runtimeStore.test.ts`
- `cd crewagent-runtime && npx eslint electron/main.ts electron/preload.ts electron/electron-env.d.ts electron/services/globalSkillService.ts electron/services/globalSkillService.test.ts electron/services/skillSourceProvider.ts electron/services/skillRegistryService.ts electron/stores/runtimeStore.ts src/stores/appStore.ts src/pages/SettingsPage/SettingsPage.tsx`
- `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Change Log

- 2026-03-19: Added Story 13.7 to formalize `user-global` and `session` skill source composition before downstream Epic 13 activation work proceeds, while reserving a package extension hook for Story 13.5.
- 2026-03-19: Completed `dev-story 13-7`; user-global source composition, deterministic precedence, registry collision diagnostics, and chat/agent/run/prompt-dump integration landed. Status moved to `review`.
- 2026-03-19: Expanded Story 13.7 scope to include user-global skill import/management UI in Runtime settings; story returned to `in-progress` until the UI/IPC slice lands.
- 2026-03-19: Completed the user-global skill import/management UI slice with Runtime settings actions and IPC-backed copy semantics; story returned to `review`.
- 2026-03-19: Code review accepted with no remaining blocking issues; Story 13.7 moved to `done`.
