# Story 13.5: Package v1.2 Skill Import Adapter

Status: blocked

## Story

As a **Package Author**,  
I want future package-bound skill imports to reuse the runtime-first skill mechanism,  
So that package support does not fork the activation and execution architecture.

## Acceptance Criteria

### AC-1: Adapter path from package imports

**Given** Epic 11.1 introduces package `skills.imports`  
**When** Runtime loads a package-bound skill declaration  
**Then** it adapts the package declaration into the same internal skill source structure used by runtime-first attached skills  
**And** it feeds the package layer into the same effective skill source composition used by user-global and session skills  
**And** it does not duplicate skill parsing, activation, resource loading, or tool narrowing logic

### AC-2: Shared downstream pipeline

**Given** the same skill can be attached runtime-first or declared in package v1.2  
**When** Runtime resolves it  
**Then** both paths converge on the same effective skill source composition, skill registry, and activation pipeline

## Technical Notes

- This story depends on Epic 11.1 schema upgrade.
- The implementation must remain an adapter only; source composition/discovery/activation stay in Epic 13.7 and Epic 13.1-13.4 code paths.
- Design: `_bmad-output/implementation-artifacts/design-13-5-package-v1-2-skill-import-adapter.md`

## Dependencies

- Epic 11.1 must deliver package `skills.imports`.
- Story 13.1 must be `done`.
- Story 13.7 must be `done`.
- Story 13.2 must be `done`.
- Story 13.3 must be `done`.
- Story 13.4 must be `done`.
- Story 13.6 must be `done`.

## BMAD Workflow Note

- This is intentionally the last Epic 13 story so package adaptation lands on top of a stabilized runtime-first skill core.
- This story remains `blocked` until Epic 11.1 and Stories 13.1, 13.7, 13.2, 13.3, 13.4, plus 13.6 all reach `done`.
- Once unblocked, promote the story to `ready-for-dev` before implementation starts.

## Tasks / Subtasks

- [ ] Task 1: Package adapter service (AC: 1,2)
  - [ ] 1.1 Add `PackageSkillImportAdapter`
  - [ ] 1.2 Resolve package-relative sources under `@pkg`
  - [ ] 1.3 Preserve agent binding in adapter output

- [ ] Task 2: RuntimeStore integration (AC: 1,2)
  - [ ] 2.1 Read `skills.imports` from v1.2 agent definitions
  - [ ] 2.2 Feed adapted sources into the effective skill source composition service

- [ ] Task 3: Tests (AC: 1,2)
  - [ ] 3.1 Adapter mapping tests
  - [ ] 3.2 Convergence tests against runtime-first attached skills

## File List

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/packageSkillImportAdapter.ts` (new)
- `crewagent-runtime/electron/services/packageSkillImportAdapter.test.ts` (new)
- `crewagent-runtime/electron/main.ts`
- `_bmad-output/implementation-artifacts/13-5-package-v1-2-skill-import-adapter.md`
- `_bmad-output/implementation-artifacts/design-13-5-package-v1-2-skill-import-adapter.md`

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/packageSkillImportAdapter.test.ts`
- `cd crewagent-runtime && npx vitest run electron/stores/runtimeStore.test.ts -t "skills.imports"`

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.5.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.5 is blocked as the final adapter slice behind Epic 11.1 and prior Epic 13 runtime slices.
- 2026-03-19: Story 13.7 inserted to define shared source layering before package imports join the runtime skill pipeline.
