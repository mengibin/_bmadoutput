# Design: Supporting Files Loading and Tool Narrowing

**Story:** `13-3-supporting-files-loading-and-tool-narrowing.md`  
**Design principle:** declared-resource-only access, deterministic narrowing, no privilege expansion

---

## Objectives

1. Load supporting files only on demand.
2. Restrict reads to resources explicitly declared by the skill.
3. Narrow tools deterministically using Claude `allowed-tools` semantics.

---

## Scope

### In scope

- `skill.load_resource`
- declared-resource path validation
- `@skill/<skillId>/...` alias support
- allowed-tools mapping
- visible-tool recomputation after activation

### Out of scope

- actual Python/Node execution
- package adaptation
- UI around resource browsing

---

## Core Design

### 1. Resource Contract

```ts
type SkillLoadResourceArgs = {
  skillId: string
  relPath: string
}
```

Rules:

- skill must be currently active
- `relPath` must match a declared supporting file
- resolved path must stay within skill root
- only text resources are readable via this tool
- generic `@skill/<skillId>/...` filesystem reads must also target declared supporting files; root-level browsing via `@skill/<skillId>` is not supported in this story

### 2. Alias Resolution

Add a new path root:

- `@skill/<skillId>/...`

Use cases:

- safe debug output
- future `fs.read` / execution bridge compatibility for declared supporting files
- script execution bridge

### 3. Allowed-Tools Mapping

Deterministic mapping table:

| Claude Tool | Runtime Tools |
|---|---|
| `Read` | `fs.read` |
| `Grep` | `fs.search` |
| `Glob` / `LS` | `fs.list` |
| `Edit` / `Write` | `fs.write`, `fs.apply_patch` if already allowed |
| `Bash` | `terminal.run`, `shell.exec`, `node.run`, `npm.install` |

Anything unmapped:

- logged as warning
- not fatal
- not added to visible tools

### 4. Narrowing Algorithm

```ts
visibleTools =
  basePolicyTools
  ∩ mappedAllowedTools
  ∩ activeSkillInternalTools
```

Important:

- if a skill declares no `allowed-tools`, base policy remains unchanged
- skill can reduce tool surface, never enlarge it

---

## Files to Modify

- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/skillRegistryService.ts`

---

## Error Model

- `SKILL_RESOURCE_NOT_DECLARED`
- `SKILL_RESOURCE_OUT_OF_SCOPE`
- `SKILL_RESOURCE_NOT_TEXT`
- `SKILL_NOT_ACTIVE`
- `SKILL_ALLOWED_TOOLS_INVALID`

---

## Test Plan

1. Active skill can load declared text resource.
2. Non-active skill cannot load resource.
3. Path traversal attempt is rejected.
4. Unmapped `allowed-tools` produces warning only.
5. Narrowed tool set never exceeds base tool set.
