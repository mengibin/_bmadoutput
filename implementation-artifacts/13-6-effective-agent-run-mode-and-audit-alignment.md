# Story 13.6: Effective Agent / Run Mode / Audit Alignment

Status: done

## Story

As a **Runtime Developer**,  
I want skill visibility and audit behavior to align with effective agent selection and run mode execution,  
So that skill behavior remains predictable across chat, agent, and run loops.

## Acceptance Criteria

### AC-1: Effective-agent skill scope

**Given** Runtime is operating in chat, agent, or run mode  
**When** skills are resolved for the current turn  
**Then** visible skills are scoped to the current effective agent  
**And** skill activation does not leak across agent boundaries

### AC-2: Structured audit events

**Given** a skill is discovered, activated, loads a resource, or fails  
**When** Runtime records diagnostics  
**Then** it writes structured audit events for discovery, activation, resource loading, and errors  
**And** each event includes enough identifiers to trace package, workflow, run, agent, and skill context

### AC-3: Future-safe run context

**Given** subworkflow and per-workflow state will land later through Epic 11  
**When** this story is implemented  
**Then** skill services and audit contracts already accept `workflowId`, `runId`, and `agentId`  
**And** no global skill activation state is introduced

## Technical Notes

- This story ties together the earlier Epic 13 slices and prevents behavior drift across modes.
- Audit visibility is part of the runtime contract, not an optional debug extra.
- Design: `_bmad-output/implementation-artifacts/design-13-6-effective-agent-run-mode-and-audit-alignment.md`

## Dependencies

- Story 13.1 must be `done`.
- Story 13.7 must be `done`.
- Story 13.2 must be `done`.
- Story 13.3 must be `done`.
- Story 13.4 must be `done`.

## BMAD Workflow Note

- This story is scheduled after the core discovery, activation, resource, and execution slices so audit and effective-agent alignment can validate the settled contracts.
- Stories 13.1, 13.7, 13.2, 13.3, and 13.4 are now `done`, so this story is the next Epic 13 slice to implement.
- Keep the scope focused on effective-agent alignment and structured audit contracts; package import adaptation remains deferred to Story 13.5.

## Tasks / Subtasks

- [x] Task 1: Effective-agent alignment (AC: 1,3)
  - [x] 1.1 Ensure all skill queries take `effectiveAgentId`
  - [x] 1.2 Invalidate active skill state when effective agent changes
  - [x] 1.3 Keep run mode aligned with chat/agent scoping

- [x] Task 2: Audit contract (AC: 2,3)
  - [x] 2.1 Add structured skill audit event schema
  - [x] 2.2 Emit discovery / activation / resource / error events
  - [x] 2.3 Include package/workflow/run/agent identifiers on every event

- [x] Task 3: Tests (AC: 1,2,3)
  - [x] 3.1 Effective-agent visibility tests
  - [x] 3.2 No-leakage tests across agent changes
  - [x] 3.3 Audit event payload tests

## File List

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `_bmad-output/architecture/runtime-claude-code-skills-architecture.md`
- `_bmad-output/implementation-artifacts/code-review-13-6-effective-agent-run-mode-and-audit-alignment.md`
- `_bmad-output/implementation-artifacts/13-6-effective-agent-run-mode-and-audit-alignment.md`
- `_bmad-output/implementation-artifacts/design-13-6-effective-agent-run-mode-and-audit-alignment.md`

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/executionEngine.test.ts electron/services/fileSystemToolHost.test.ts`
- `cd crewagent-runtime && npx vitest run electron/services/executionEngine.test.ts -t "effectiveAgentId|skill discovery audit|clears active skill state"`
- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "skill activation and resource audit|skill failure audit"`
- `cd crewagent-runtime && npx eslint electron/services/executionEngine.ts electron/services/executionEngine.test.ts electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/main.ts`
- `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Dev Notes (2026-03-19)

- `ExecutionEngine` now treats `effectiveAgentId` as the only valid owner of in-loop skill activation state; if the workflow node switches to another effective agent, the prior active skill state is cleared before the next turn is composed.
- Run-mode skill discovery audit now emits structured `skill.discovered` / `skill.discovery_failed` events into `execution.jsonl` with `packageId`, `workflowId`, `runId`, and `agentId`.
- Main-process chat/agent skill-registry resolution now mirrors the same discovery audit contract by writing skill discovery events into the per-run audit log for `chat-*` runs.
- `FileSystemToolHost` now emits structured `skill.activated`, `skill.activation_failed`, `skill.resource_loaded`, `skill.resource_load_failed`, and `skill.tools_narrowed` events with full run context identifiers.

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.6.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.6 is blocked behind Stories 13.1-13.4.
- 2026-03-19: Story 13.7 inserted before activation/audit alignment to stabilize effective skill source composition; dependencies updated.
- 2026-03-19: Completed `dev-story 13.6`; effective-agent-owned skill state invalidation and structured skill audit events landed for run/chat/agent flows. Status moved to `review`.
- 2026-03-19: Code review passed with no remaining blocking findings; Story 13.6 closed as `done`.
