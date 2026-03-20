# Story 13.1: Runtime Skill Discovery & Registry Injection

Status: done

<!-- Note: Code review completed. Follow-up fixes landed. See code-review-13-1-runtime-skill-discovery-and-registry-injection.md -->

## Story

As a **Runtime User**,  
I want Runtime to discover attached Claude Code skills and expose a compact registry to the LLM,  
So that the model knows which skills are available without preloading all skill content.

## Acceptance Criteria

### AC-1: Runtime skill discovery

**Given** the host attaches one or more runtime-accessible skill sources  
**When** Runtime initializes the session context  
**Then** it discovers valid Claude Code style `SKILL.md` skills  
**And** parses each skill into a normalized internal model with `skillId`, `description`, root path, and supporting file index  
**And** excludes invalid or out-of-scope skill sources with structured diagnostics

### AC-2: Registry injection

**Given** valid skills are discovered  
**When** the initial system prompt is composed  
**Then** Runtime injects a skill registry block listing available skills and one-line descriptions  
**And** it does not inject the full skill body at this stage  
**And** the registry is filtered by the current effective agent scope

### AC-3: Multi-mode consistency

**Given** Runtime is operating in `chat`, `agent`, or `run` mode  
**When** the system prompt is built  
**Then** the same discovery result is available for injection  
**And** mode-specific prompt sections continue to work without being replaced by skill registry logic

## Technical Notes

- Delivery pattern: runtime vertical slice across Main process, context builders, and tests.
- This story establishes the host-to-runtime skill attachment path through `attachedSkills`.
- No skill activation is in scope here; only discovery and registry injection.
- Design and review follow-ups are complete; this story is closed and now acts as the foundation for downstream Epic 13 slices.
- Design: `_bmad-output/implementation-artifacts/design-13-1-runtime-skill-discovery-and-registry-injection.md`

## Dependencies

- None. This is the Epic 13 entry story and the first implementation slice.

## BMAD Workflow Note

- This foundation slice is complete.
- Story 13.7 is now the next Epic 13 story allowed to move into implementation.
- If the discovery or prompt-injection interface changes during implementation, update the downstream design artifacts before promoting the next story.

## Tasks / Subtasks

- [x] Task 1: Skill discovery service (AC: 1,2)
  - [x] 1.1 Add `attachedSkills` input contract to session entrypoints
  - [x] 1.2 Implement `SkillRegistryService` for directory and direct `SKILL.md` discovery
  - [x] 1.3 Parse frontmatter/body into normalized skill summaries
  - [x] 1.4 Build supporting-file index from explicit relative links only

- [x] Task 2: Prompt injection (AC: 2,3)
  - [x] 2.1 Generate a compact `Skill Registry` system block
  - [x] 2.2 Reuse `extraSystemMessages` in chat mode
  - [x] 2.3 Add `extraSystemMessages` support to agent mode
  - [x] 2.4 Add dynamic system block support in run mode

- [x] Task 3: Effective-agent scoping (AC: 2,3)
  - [x] 3.1 Filter visible skills by current `effectiveAgentId`
  - [x] 3.2 Keep discovery diagnostics separate from visible registry output

- [x] Task 4: Tests and diagnostics (AC: 1,2,3)
  - [x] 4.1 Unit tests for valid/invalid discovery paths
  - [x] 4.2 Unit tests for agent-scoped filtering
  - [x] 4.3 Integration tests for chat / agent / run prompt injection order

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] Move skill deduplication to happen after effective-agent visibility filtering so duplicate attachments for different agents do not hide valid skills. [crewagent-runtime/electron/services/skillRegistryService.ts]
- [x] [AI-Review][MEDIUM] Normalize `attachedSkills` in `agent:dispatch` using the same payload parser as chat/run mode. [crewagent-runtime/electron/main.ts]
- [x] [AI-Review][MEDIUM] Add regression coverage for duplicate-source multi-agent discovery and shared attached-skills payload normalization. [crewagent-runtime/electron/services/skillRegistryService.test.ts] [crewagent-runtime/electron/services/attachedSkillSource.test.ts]

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.ts`
- `crewagent-runtime/electron/services/agentContextBuilder.ts`
- `crewagent-runtime/electron/services/systemMessageInjection.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/attachedSkillSource.ts` (new)
- `crewagent-runtime/electron/services/attachedSkillSource.test.ts` (new)
- `crewagent-runtime/electron/services/skillRegistryService.ts` (new)
- `crewagent-runtime/electron/services/skillRegistryService.test.ts` (new)
- `crewagent-runtime/electron/services/agentSessionContract.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/electron/services/agentContextBuilder.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `_bmad-output/implementation-artifacts/13-1-runtime-skill-discovery-and-registry-injection.md`
- `_bmad-output/implementation-artifacts/design-13-1-runtime-skill-discovery-and-registry-injection.md`

## Dev Notes (2026-03-19)

- Added `SkillRegistryService` to discover Claude Code style skills from `attachedSkills`, parse `SKILL.md` frontmatter, derive normalized summaries, and index only explicit relative supporting-file links.
- Introduced shared `systemMessageInjection` helper so chat, agent, and run flows inject registry blocks in the same place in the system-message prefix.
- Extended chat / agent / run entrypoints with optional `attachedSkills` plumbing while keeping existing callers backward-compatible.
- Added debug-time and runtime diagnostics for invalid or out-of-scope skill sources without failing the whole session when at least one visible skill remains.
- Review follow-up: moved dedupe behind effective-agent visibility, extracted shared `attachedSkills` payload normalization, and added regression coverage for duplicate-source multi-agent attachments.

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/attachedSkillSource.test.ts electron/services/skillRegistryService.test.ts`
- `cd crewagent-runtime && npx vitest run electron/services/skillRegistryService.test.ts`
- `cd crewagent-runtime && npx vitest run electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/executionEngine.test.ts -t "skill registry"`
- `npm --prefix crewagent-runtime test -- electron/services/attachedSkillSource.test.ts electron/services/skillRegistryService.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/executionEngine.test.ts`
  - Passed: `5` files, `43` tests.
- `cd crewagent-runtime && npx tsc --noEmit`
  - Passed with `0` errors.

## Review Follow-up Resolution (AI)

- [x] Fixed the effective-agent scoping bug by deduplicating only after a skill attachment is confirmed visible to the current agent.
- [x] Unified `attachedSkills` payload normalization across `chat:send`, `agent:dispatch`, and `runs:continue` via a shared helper.
- [x] Added regression tests for duplicate-source multi-agent discovery and payload normalization trimming/filtering.
- [x] Follow-up verification:
  - `npm --prefix crewagent-runtime test -- electron/services/attachedSkillSource.test.ts electron/services/skillRegistryService.test.ts electron/services/chatContextBuilder.test.ts electron/services/agentContextBuilder.test.ts electron/services/executionEngine.test.ts`
  - `cd crewagent-runtime && npx tsc --noEmit`

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.1.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.1 is now the only `ready-for-dev` entry point.
- 2026-03-19: Completed `dev-story 13.1`; discovery service, chat/agent/run registry injection, and validation tests landed. Status moved to `review`.
- 2026-03-19: Senior Developer Review completed; H1/M1 issues logged in `code-review-13-1-runtime-skill-discovery-and-registry-injection.md`; story temporarily returned to `in-progress`.
- 2026-03-19: Completed review follow-ups for duplicate-source agent scoping and shared payload normalization; regression tests added; status moved back to `review`.
- 2026-03-19: Re-review passed with `0` blocking findings; Story 13.1 closed as `done` and Epic 13 advanced to Story 13.7.
