# Design: Python / Node / Terminal Skill Execution Bridge

**Story:** `13-4-python-node-terminal-skill-execution-bridge.md`  
**Design principle:** first-class Node execution, isolated dependency install, bundled runtime tooling first

---

## Objectives

1. Make JS/Node skills executable as first-class runtime workloads.
2. Add `node.run` and `npm.install` instead of relying on `terminal.run node ...` as the primary path.
3. Keep dependency installs isolated from source skill directories.
4. Reuse existing Python and terminal bridges where appropriate.

---

## Scope

### In scope

- `python.run` gains `module`
- new `node.run`
- new `npm.install`
- per-skill isolated execution workspace
- npm auto-install-on-missing and retry-once flow
- reuse bundled Node.js/npm

### Out of scope

- pnpm / yarn support
- long-lived JS daemons
- TypeScript transpilation pipeline beyond what a skill explicitly brings itself

---

## Core Design

### 1. Tool Contracts

```ts
type PythonRunArgs = {
  code?: string
  file?: string
  module?: string
  args?: string[]
}

type NodeRunArgs = {
  code?: string
  file?: string
  module?: string
  args?: string[]
  cwd?: string
}

type NpmInstallArgs = {
  skillId: string
  packages?: string[]
  packageJson?: string
  dev?: boolean
}
```

Rules:

- `python.run`: exactly one of `code | file | module`
- `node.run`: exactly one of `code | file | module`
- `npm.install`: explicit package list or workspace `package.json`

### 2. Skill Workspace

Each resolved skill gets a persistent workspace under Runtime private state:

```text
runtime-store/skill-workspaces/<skillKey>/
  source-shadow/
  package.json
  node_modules/
  install-manifest.json
```

Decision:

- `skillKey = sha256(realpath(sourceRoot))`
- workspace is reused by `skillKey`, while `source-shadow/` is refreshed before execution so active skill files stay current
- installs never mutate the original source directory
- Node/npm child processes should reuse the same minimal execution-environment baseline as Story 5.19 terminal execution instead of inheriting full host `process.env`

### 3. Bundled Node.js/npm

Execution order:

1. use bundled Node.js from `nodeService`
2. use bundled npm from `nodeService`
3. only fall back to `terminal.run` when shell semantics are actually required

### 4. Auto-install Flow

For missing npm package during `node.run`:

1. detect module-not-found failure
2. resolve package name
3. current implementation auto-installs once for active skill execution:
   - call `npm.install`
   - retry `node.run` once
4. if still failing:
   - return structured error

### 5. When to use terminal.run

Use `terminal.run` only for:

- shell scripts
- external CLIs
- workflows where skill explicitly calls for shell behavior

Use `node.run` for:

- JS script files
- inline JS
- Node module entrypoints

---

## Expected Files

- `crewagent-runtime/electron/services/nodeService.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/shared/agentToolPolicy.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/skillWorkspaceService.ts` (new)
- `crewagent-runtime/electron/services/skillWorkspaceService.test.ts` (new)

---

## Error Model

- `NODE_NOT_FOUND`
- `NPM_NOT_FOUND`
- `NODE_MODULE_NOT_FOUND`
- `NODE_EXIT_CODE`
- `NODE_TIMEOUT`
- `NPM_INSTALL_FAILED`
- `SKILL_WORKSPACE_INIT_FAILED`
- `SKILL_SYSTEM_DEPENDENCY_MISSING`

---

## Test Plan

1. `node.run` with inline code.
2. `node.run` with file under `@skill`.
3. `node.run` with module entrypoint.
4. `npm.install` with explicit package.
5. missing npm dependency triggers install and single retry.
6. workspace install does not modify source skill directory.
7. `python.run module` regression.
