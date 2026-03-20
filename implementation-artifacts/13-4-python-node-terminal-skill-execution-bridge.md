# Story 13.4: Python/Node/Terminal Skill Execution Bridge

Status: done

## Story

As a **Runtime User**,  
I want activated skills to work through the existing execution tools,  
So that supporting scripts can actually run inside Runtime.

## Acceptance Criteria

### AC-1: Python and Node execution

**Given** a skill references Python or JavaScript supporting scripts or module-style entrypoints  
**When** the model calls language execution through the skill flow  
**Then** Runtime supports `python.run` with `code`, `file`, or `module`  
**And** Runtime supports `node.run` with `code`, `file`, or `module`  
**And** skill-relative file paths can resolve through a stable skill alias

### AC-2: npm dependency install

**Given** a JavaScript skill depends on npm packages  
**When** execution encounters missing npm dependencies  
**Then** Runtime can install them through `npm.install` in an isolated skill workspace  
**And** if auto-install is enabled it retries the failed `node.run` once after installation

### AC-3: Bundled runtime bridge

**Given** a skill relies on Python, Node, or shell execution  
**When** execution is attempted  
**Then** Runtime reuses current Python dependency handling, bundled Node.js/npm, and existing terminal bridge  
**And** returns structured errors for missing system dependencies instead of silent failure

## Technical Notes

- This story depends on Story 5.15, Story 5.16, and Story 5.19 as existing runtime execution foundations.
- JS/Node execution becomes first-class here; `terminal.run node ...` is no longer the intended primary path for skills.
- Design: `_bmad-output/implementation-artifacts/design-13-4-python-node-terminal-skill-execution-bridge.md`

## Dependencies

- Story 13.1 must be `done`.
- Story 13.7 must be `done`.
- Story 13.2 must be `done`.
- Story 13.3 must be `done`.
- Story 5.19 terminal bridge implementation must be landed; its remaining review-only cross-platform validation is tracked separately.

## BMAD Workflow Note

- The design artifact is drafted early so Python/Node/npm interfaces can be reviewed before implementation starts.
- Story 5.19 still carries a `review` status because of pending cross-platform manual validation, but its terminal bridge implementation is already the execution foundation reused here.
- Keep the implementation scoped to Python/Node/npm execution bridging and isolated dependency install; do not expand into package import adaptation or broader agent/run alignment work from later stories.

## Tasks / Subtasks

- [x] Task 1: Python bridge upgrade (AC: 1,3)
  - [x] 1.1 Extend `python.run` to support `module`
  - [x] 1.2 Keep existing auto-install behavior for Python packages
  - [x] 1.3 Add regression tests for code/file/module exclusivity

- [x] Task 2: Node execution bridge (AC: 1,3)
  - [x] 2.1 Add `node.run` tool contract and result type
  - [x] 2.2 Execute with bundled Node.js
  - [x] 2.3 Support `code`, `file`, and `module`
  - [x] 2.4 Resolve `@skill/<skillId>/...` files

- [x] Task 3: npm install bridge (AC: 2,3)
  - [x] 3.1 Add `npm.install` tool contract and result type
  - [x] 3.2 Create isolated skill workspace service
  - [x] 3.3 Install packages into workspace only
  - [x] 3.4 Retry failed `node.run` once after auto-install

- [x] Task 4: Terminal fallback and diagnostics (AC: 3)
  - [x] 4.1 Keep `terminal.run` for shell and external CLI scenarios
  - [x] 4.2 Return structured errors for missing system dependencies
  - [x] 4.3 Audit Node/npm execution and install events

- [x] Task 5: Tests (AC: 1,2,3)
  - [x] 5.1 `node.run` inline/file/module tests
  - [x] 5.2 `npm.install` workspace isolation tests
  - [x] 5.3 auto-install retry path tests
  - [x] 5.4 `python.run module` regression tests

### Review Follow-ups (AI)

- [x] [AI-Review][HIGH] Reuse the terminal execution env isolation for `node.run` / `npm.install` so skill scripts and npm lifecycle hooks cannot read arbitrary host secrets from `process.env`. [crewagent-runtime/electron/services/fileSystemToolHost.ts] [crewagent-runtime/electron/services/skillWorkspaceService.ts] [crewagent-runtime/electron/services/terminalService.ts]
- [x] [AI-Review][MEDIUM] Stop deleting the shared `source-shadow` in place on every ensure; build a fresh shadow path per ensure so concurrent runs of the same skill do not race and remove each other's execution files. [crewagent-runtime/electron/services/skillWorkspaceService.ts]

## File List

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/skillActivation.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/skillWorkspaceService.ts` (new)
- `crewagent-runtime/electron/services/skillWorkspaceService.test.ts` (new)
- `_bmad-output/architecture/runtime-claude-code-skills-architecture.md`
- `_bmad-output/implementation-artifacts/13-4-python-node-terminal-skill-execution-bridge.md`
- `_bmad-output/implementation-artifacts/code-review-13-4-python-node-terminal-skill-execution-bridge.md`
- `_bmad-output/implementation-artifacts/design-13-4-python-node-terminal-skill-execution-bridge.md`

## Verification

- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts electron/services/skillWorkspaceService.test.ts`
- `cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "node.run|npm.install|host environment|Story 13.4"`
- `cd crewagent-runtime && npx vitest run electron/services/skillWorkspaceService.test.ts`
- `cd crewagent-runtime && npx eslint electron/services/fileSystemToolHost.ts electron/services/fileSystemToolHost.test.ts electron/services/skillActivation.ts electron/services/toolHost.ts electron/services/skillWorkspaceService.ts electron/services/skillWorkspaceService.test.ts electron/services/terminalService.ts`
- `cd crewagent-runtime && npx tsc --noEmit --pretty false`

## Review Follow-up Resolution (AI)

- [x] `node.run` now uses `buildExecutionEnv(undefined, policy)` instead of inheriting full `process.env`, matching the terminal env-isolation baseline from Story 5.19.
- [x] `npm.install` now also builds its child process environment from the same minimal base env before applying npm registry/proxy overrides.
- [x] `SkillWorkspaceService` now creates a unique `source-shadow/<uuid>/` per ensure call rather than deleting and recreating a shared shadow path in place.
- [x] Added regression coverage for host env isolation in `node.run` and for repeated `ensureWorkspace()` calls preserving earlier shadow directories.

## Dev Notes (2026-03-19)

- `python.run` now supports `module` alongside the existing `code` and `file` modes, while preserving the Story 5.16 auto-install retry behavior for Python packages.
- Added first-class `node.run` and `npm.install` tool contracts on top of the existing terminal-enabled execution surface instead of routing JS skill execution through `terminal.run node ...`.
- Introduced `SkillWorkspaceService`, which creates runtime-managed workspaces under `runtime-store/skill-workspaces/<skillKey>/`, rebuilds a `source-shadow/`, and keeps npm installs isolated from the original skill source tree.
- `node.run` now executes inline code, declared `@skill/<skillId>/...` files, and module entrypoints with bundled Node.js; module-not-found failures trigger a single default-on `npm.install` retry path inside the skill workspace.
- Updated `allowed-tools: Bash` narrowing so activated skills can see the new `node.run` / `npm.install` capabilities once those runtime tools exist.
- Review follow-up: `node.run` and `npm.install` now reuse the same minimal execution-environment model as Story 5.19 terminal execution instead of inheriting full host `process.env`.
- Review follow-up: `SkillWorkspaceService.ensureWorkspace()` now creates a fresh `source-shadow/<uuid>/` per ensure call, so concurrent runs no longer delete and recreate the same shared shadow directory in place.

## Change Log

- 2026-03-19: Initial design and implementation artifact drafted for Epic 13 Story 13.4.
- 2026-03-19: Expanded execution bridge scope to include first-class `node.run` and `npm.install`.
- 2026-03-19: Converted Epic 13 to strict BMAD execution flow; Story 13.4 is blocked behind Stories 13.1-13.3 and Story 5.19.
- 2026-03-19: Story 13.7 inserted before activation/resource execution to stabilize effective skill source composition; dependencies updated.
- 2026-03-19: Completed `dev-story 13.4`; bundled Node/npm execution, isolated skill workspaces, `python.run module`, and npm auto-install retry landed. Status moved to `review`.
- 2026-03-19: Resolved 13.4 code-review findings for env isolation and workspace shadow concurrency; regression tests added and Story 13.4 closed as `done`.
