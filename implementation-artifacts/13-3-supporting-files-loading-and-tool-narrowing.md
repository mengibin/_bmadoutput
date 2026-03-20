# Story 13.3: Supporting Files Loading & Tool Narrowing

Status: done

## Story

As a **Runtime User**,  
I want the model to load supporting files only when needed and to obey the skill's allowed tool scope,  
So that context remains efficient and the skill cannot widen permissions.

## Acceptance Criteria

### AC-1: On-demand supporting file loading

**Given** a skill is active and declares supporting files  
**When** the model requests a supporting file  
**Then** Runtime allows loading only files explicitly linked by the skill and only within that skill's root directory  
**And** returns structured errors for undeclared or out-of-scope paths

### AC-2: Deterministic tool narrowing

**Given** a skill defines `allowed-tools`  
**When** the skill becomes active  
**Then** Runtime narrows the visible tool set based on a deterministic mapping to existing runtime tools  
**And** never grants tools that are not already allowed by runtime and agent policy

### AC-3: Alias boundary

**Given** supporting files and future execution bridges need stable addressing  
**When** Runtime resolves skill-local paths  
**Then** it exposes them through a stable `@skill/<skillId>/...` alias  
**And** alias resolution remains bounded to the active skill root

## Technical Notes

- This story depends on Story 13.7 source composition and Story 13.2 activation semantics on top of Story 13.1 discovery.
- `skill.load_resource` is internal runtime tooling, not a user-facing file browser.
- Design: `_bmad-output/implementation-artifacts/design-13-3-supporting-files-loading-and-tool-narrowing.md`

## Dependencies

- Story 13.1 must be `done`.
- Story 13.7 must be `done`.
- Story 13.2 must be `done`.

## BMAD Workflow Note

- The design artifact is drafted early to lock the `@skill` alias and `allowed-tools` narrowing contract.
- Story 13.7 is `done`, and the shared `user-global / session` source-composition path is the implementation base for this story.
- Keep the implementation scoped to declared-resource loading, alias boundaries, and narrowing semantics; execution-bridge behavior stays in Story 13.4.

## Tasks / Subtasks

- [x] Task 1: Resource loading tool (AC: 1,3)
  - [x] 1.1 Add `skill.load_resource`
  - [x] 1.2 Validate active skill and declared-resource membership
  - [x] 1.3 Enforce text-only resource reads
  - [x] 1.4 Add `@skill/<skillId>/...` alias resolution

- [x] Task 2: Allowed-tools mapper (AC: 2)
  - [x] 2.1 Implement deterministic Claude-to-runtime mapping
  - [x] 2.2 Emit warnings for unmapped entries
  - [x] 2.3 Keep narrowing as intersection only

- [x] Task 3: Loop integration (AC: 2,3)
  - [x] 3.1 Recompute visible tools after activation and after resource-related mutations
  - [x] 3.2 Ensure run/chat loops consume the narrowed tool list consistently

- [x] Task 4: Tests (AC: 1,2,3)
  - [x] 4.1 Declared resource read success
  - [x] 4.2 Out-of-scope resource rejection
  - [x] 4.3 Narrowed tool surface regression tests

## File List

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/skillActivation.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/chatToolLoop.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `_bmad-output/implementation-artifacts/13-3-supporting-files-loading-and-tool-narrowing.md`
- `_bmad-output/implementation-artifacts/design-13-3-supporting-files-loading-and-tool-narrowing.md`

## Dev Notes (2026-03-19)

- Added deterministic `allowed-tools` mapping in `skillActivation.ts`, carrying `narrowedToolNames` and warning metadata through the existing `ContextMutationResult`.
- Extended `FileSystemToolHost` with `skill.load_resource`, active-skill-only `@skill/<skillId>/...` alias resolution, text-only declared resource enforcement, and dynamic `skill.load_resource` tool exposure when the active skill has text supporting files.
- Updated `getVisibleTools()` so built-in runtime tools are intersected with `narrowedToolNames` while internal tools (`ui.ask_user`, `skill.activate`, `skill.load_resource`) remain available for loop control and resource loading.
- Reused the existing Story 13.2 loop mutation flow; chat/run tests now verify that post-activation tool recomputation exposes the narrowed tool set plus `skill.load_resource`.
- Review follow-up: generic `@skill/<skillId>/...` file access now requires a declared supporting file target instead of exposing the whole skill root through `fs.read`/`fs.search`/`fs.list`, and `allowed-tools: Bash` tracks the currently implemented local execution tools (`terminal.run`, `shell.exec`, `node.run`, `npm.install`).

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "skill"`
- `cd crewagent-runtime && npx vitest run electron/services/chatToolLoop.test.ts -t "loop-local context mutation"`
- `cd crewagent-runtime && npx vitest run electron/services/executionEngine.test.ts -t "applies skill activation only within the current run loop"`
- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts -t "skill|loop-local context mutation|current run loop"`
- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts electron/services/skillRegistryService.test.ts`
- `cd crewagent-runtime && npx eslint electron/services/skillActivation.ts electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/services/chatToolLoop.ts electron/services/executionEngine.ts`
- `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.3.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.3 is blocked behind Stories 13.1 and 13.2.
- 2026-03-19: Story 13.7 inserted before activation and resource loading to stabilize effective skill source composition; dependencies updated.
- 2026-03-19: Completed `dev-story 13.3`; `skill.load_resource`, active-skill `@skill` alias resolution, deterministic `allowed-tools` narrowing, and chat/run regression coverage landed. Status moved to `review`.
- 2026-03-19: Review follow-up tightened generic `@skill` alias reads to declared supporting files only and aligned `allowed-tools: Bash` with currently implemented runtime tools; Story 13.3 moved to `done`.
