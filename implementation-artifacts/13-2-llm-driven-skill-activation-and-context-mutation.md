# Story 13.2: LLM-Driven Skill Activation & Context Mutation

Status: done

<!-- Note: Code review completed. Follow-up fixes landed. See code-review-13-2-llm-driven-skill-activation-and-context-mutation.md -->

## Story

As a **Runtime User**,  
I want the main model to decide when to activate a skill during the conversation,  
So that skill usage follows model judgment rather than a separate routing step.

## Acceptance Criteria

### AC-1: LLM-triggered activation

**Given** the model sees a skill registry in context  
**When** it decides a skill should be used  
**Then** it can call a dedicated internal activation tool for an allowed skill  
**And** Runtime rejects activation for skills with `disable-model-invocation=true`

### AC-2: Context mutation after activation

**Given** skill activation succeeds  
**When** the tool loop continues to the next model turn  
**Then** Runtime injects the activated skill instructions as new system context  
**And** recomputes the visible tools for the next turn  
**And** keeps the activation state only within the current loop, without persisting it to conversation or workflow state

### AC-3: Multi-loop parity

**Given** Runtime is in chat or run mode  
**When** a skill is activated  
**Then** both loop implementations apply the same context mutation semantics  
**And** neither path requires a separate router model

## Technical Notes

- This story depends on Story 13.1 skill discovery/registry injection and Story 13.7 effective `user-global + session` source composition.
- `skill.activate` is an internal control tool, not a business tool.
- The main implementation risk is loop orchestration, not parsing.
- Design: `_bmad-output/implementation-artifacts/design-13-2-llm-driven-skill-activation-and-context-mutation.md`

## Dependencies

- Story 13.1 must be `done`.
- Story 13.7 must be `done`.

## BMAD Workflow Note

- The design artifact is intentionally written ahead of implementation so the loop-mutation interfaces are frozen early.
- Story 13.7 is `done`; this story reuses that shared `user-global / session` source-composition code path as its implementation base.
- Keep the implementation strictly within the activation/context-mutation slice; do not pull in supporting-file loading or execution-bridge scope from later stories.

## Tasks / Subtasks

- [x] Task 1: Internal tool contract (AC: 1,2)
  - [x] 1.1 Add `skill.activate` tool schema
  - [x] 1.2 Restrict available `skillId` values to current visible skills
  - [x] 1.3 Reject `disable-model-invocation=true`

- [x] Task 2: Context mutation result handling (AC: 2,3)
  - [x] 2.1 Extend tool execution path with `ContextMutationResult`
  - [x] 2.2 Add loop-local `SkillLoopState`
  - [x] 2.3 Rebuild system messages after activation
  - [x] 2.4 Recompute visible tools after activation

- [x] Task 3: Chat / run integration (AC: 2,3)
  - [x] 3.1 Update `chatToolLoop.ts`
  - [x] 3.2 Update `executionEngine.ts`
  - [x] 3.3 Add `extraSystemMessages` support to agent path as needed

- [x] Task 4: Tests (AC: 1,2,3)
  - [x] 4.1 Activation success path
  - [x] 4.2 Activation rejection path
  - [x] 4.3 Repeated activation behavior
  - [x] 4.4 Non-persistence across turns

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] Reuse runtime alias resolution for `@state/skills/global/...` so user-global skills can be activated after being discovered in the registry. [crewagent-runtime/electron/services/fileSystemToolHost.ts]
- [x] [AI-Review][MEDIUM] Narrow non-persistence to `skill.activate` protocol only so mixed turns still persist ordinary assistant/tool messages. [crewagent-runtime/electron/services/chatToolLoop.ts] [crewagent-runtime/electron/services/executionEngine.ts]
- [x] [AI-Review][MEDIUM] Add regression coverage for user-global activation and mixed `skill.activate + ordinary tool` turns in chat/run mode. [crewagent-runtime/electron/services/fileSystemToolHost.test.ts] [crewagent-runtime/electron/services/chatToolLoop.test.ts] [crewagent-runtime/electron/services/executionEngine.test.ts]

## File List

- `crewagent-runtime/electron/services/skillActivation.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/skillRegistryService.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/chatToolLoop.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `_bmad-output/implementation-artifacts/13-2-llm-driven-skill-activation-and-context-mutation.md`
- `_bmad-output/implementation-artifacts/design-13-2-llm-driven-skill-activation-and-context-mutation.md`

## Dev Notes (2026-03-19)

- Added a shared `skillActivation.ts` contract for `ContextMutationResult`, `SkillLoopState`, `ToolSkillContext`, and the dynamic `skill.activate` tool schema so chat/run loops interpret activation consistently.
- Extended `FileSystemToolHost` to expose `skill.activate`, validate visible `skillId` values, reject `disable-model-invocation=true`, and load the full `SKILL.md` body into an activated-skill system block on success.
- Updated `chatToolLoop` to treat activation as loop-local context mutation: rebuild the leading system messages, recompute the visible tools, and avoid persisting the activation tool protocol through the chat persistence callbacks.
- Updated `ExecutionEngine` to keep loop-local skill activation state in-memory for a single `start/continue` run loop, inject activated skill instructions on subsequent turns, and keep activation assistant/tool protocol out of persisted conversation history.
- Reused the existing Story 13.7 effective source composition and Story 13.1 registry discovery paths; this story does not yet implement supporting-file loading or allowed-tools narrowing.
- Review follow-up: `skill.activate` now resolves `@state/skills/global/...` through the runtime alias resolver so user-global skills can actually activate, and mixed activation/business-tool turns now persist only the non-activation assistant/tool protocol.

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts -t "skill activation"`
- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "skill.activate"`
- `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Review Follow-up Resolution (AI)

- [x] `skill.activate` now uses runtime-backed alias resolution for `@state/skills/global/...`, aligning activation with Story 13.7 global-skill discovery semantics.
- [x] Chat/run persistence logic now strips only activation protocol from history while preserving ordinary tool calls and tool results in mixed turns.
- [x] Added regression coverage for user-global skill activation plus mixed `skill.activate + fs.read` turns in both chat and run loops.
- [x] Follow-up verification:
  - `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "user-global skills"`
  - `cd crewagent-runtime && npx vitest run electron/services/chatToolLoop.test.ts -t "persists non-activation tool protocol when skill.activate shares a turn with ordinary tools"`
  - `cd crewagent-runtime && npx vitest run electron/services/executionEngine.test.ts -t "persists non-activation tool protocol in run mode when activation shares a turn with ordinary tools"`
  - `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.2.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.2 is blocked behind Story 13.1 completion.
- 2026-03-19: Story 13.1 re-review passed; Story 13.2 promoted from `blocked` to `ready-for-dev`.
- 2026-03-19: Story 13.7 inserted ahead of activation to formalize effective skill source layering; Story 13.2 returned to `blocked`.
- 2026-03-19: Completed `dev-story 13.2`; `skill.activate`, loop-local context mutation, chat/run parity, and non-persistent activation protocol handling landed. Status moved to `review`.
- 2026-03-19: Completed review follow-ups for user-global activation aliasing and mixed-turn persistence semantics; regression tests added; story remains in `review` pending re-review.
- 2026-03-19: Re-review passed with `0` blocking findings; Story 13.2 closed as `done`.
