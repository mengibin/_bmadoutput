# Design: Runtime Skill Discovery and Registry Injection

**Story:** `13-1-runtime-skill-discovery-and-registry-injection.md`  
**Design principle:** runtime-first source attachment, lazy prompt injection, agent-scoped visibility

---

## Objectives

1. Discover Claude Code style skills from runtime-accessible sources supplied by the host.
2. Normalize each skill into a stable internal model before any activation occurs.
3. Inject a compact skill registry into chat, agent, and run system prompts.
4. Keep initial context light by avoiding full `SKILL.md` body injection.

---

## Scope

### In scope

- Host-provided `attachedSkills` input.
- Directory or direct `SKILL.md` discovery.
- Frontmatter + body parsing for registry metadata.
- Supporting file link index extraction.
- Effective-agent filtering.
- Skill registry system message injection for chat / agent / run.

### Out of scope

- Skill activation.
- Supporting file content loading.
- Tool narrowing.
- JS/Python execution bridge.
- Package v1.2 import adaptation.

---

## Core Design

### 1. Input Contract

```ts
type AttachedSkillSource = {
  source: string
  agentIds?: string[]
}
```

- `source` may point to:
  - a directory containing `SKILL.md`
  - a direct `SKILL.md` path
- `agentIds` is optional; omitted means visible to any effective agent in the session scope.

### 2. SkillRegistryService

New main-process service:

```ts
interface SkillRegistryService {
  discover(params: {
    attachedSkills: AttachedSkillSource[]
    packageId?: string
    workflowId: string
    runId: string
    agentId: string
  }): Promise<SkillDiscoveryResult>
}
```

Responsibilities:

- resolve and validate source paths
- find `SKILL.md`
- parse frontmatter and markdown body
- derive `skillId`, `displayName`, `description`
- build a supporting-file index from explicit relative links
- filter skills by current `agentId`

### 3. Normalized Models

```ts
type ResolvedSkillSummary = {
  skillId: string
  displayName: string
  description: string
  rootPath: string
  skillMdPath: string
  supportingFiles: Array<{
    relPath: string
    kind: 'text' | 'script' | 'other'
  }>
  allowedTools?: string[]
  disableModelInvocation: boolean
  agentIds?: string[]
}

type SkillDiscoveryResult = {
  visibleSkills: ResolvedSkillSummary[]
  errors: Array<{
    source: string
    code: string
    message: string
  }>
}
```

Decision:

- `description` is required for a valid visible skill.
- `name` may fall back to directory name.
- invalid skills are excluded from registry but surfaced in diagnostics.

### 4. Supporting File Index

- Parse only local relative links from `SKILL.md`.
- Classify each linked file:
  - `text`: `.md`, `.txt`, `.json`, `.yaml`, `.yml`
  - `script`: `.py`, `.js`, `.mjs`, `.cjs`, `.ts`, `.sh`
  - `other`: everything else
- Do not read file contents at discovery time.

### 5. Prompt Injection

All three modes must receive a `Skill Registry` system block:

```md
## Skill Registry
- `pptx-tools`: Build and edit slide decks. Model activation allowed. Resources: `editing.md`, `pptxgenjs.md`
- `finance-helper`: Validate finance calculations. Model activation disabled.
```

Rules:

- Only visible skills for the current effective agent are listed.
- Only summary metadata is injected.
- No full skill instructions in initial prompt.

---

## Architecture Impact

### Main process

- `electron/main.ts`
  - accept `attachedSkills` in chat / agent / run session entrypoints
  - call `SkillRegistryService.discover(...)`
  - pass registry block via `extraSystemMessages`

### Context builders

- `chatContextBuilder.ts`
  - already supports `extraSystemMessages`; reuse
- `agentContextBuilder.ts`
  - add `extraSystemMessages`
- `executionEngine.ts`
  - add dynamic system block insertion for run mode

### New files

- `crewagent-runtime/electron/services/skillRegistryService.ts`
- `crewagent-runtime/electron/services/skillRegistryService.test.ts`

---

## Error Model

- `SKILL_SOURCE_NOT_FOUND`
- `SKILL_MD_NOT_FOUND`
- `SKILL_FRONTMATTER_INVALID`
- `SKILL_DESCRIPTION_MISSING`
- `SKILL_OUT_OF_SCOPE`

Errors do not fail the whole session unless all declared skills are invalid and the caller explicitly required skills to be present.

---

## Test Plan

### Unit

1. Discover directory-based skill.
2. Discover direct `SKILL.md` path.
3. Exclude invalid skill without `description`.
4. Build supporting-file index from relative links only.
5. Filter visible skills by `agentIds`.

### Integration

1. Chat mode injects skill registry.
2. Agent mode injects skill registry after persona.
3. Run mode injects skill registry together with run context.

---

## Files to Modify

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/chatContextBuilder.ts`
- `crewagent-runtime/electron/services/agentContextBuilder.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/skillRegistryService.ts` (new)
- `crewagent-runtime/electron/services/skillRegistryService.test.ts` (new)

---

## Manual Verification Checklist

1. Attach two valid skills and one invalid skill; only valid ones appear in registry.
2. Restrict one skill to a specific agent; switch effective agent and confirm registry changes.
3. Confirm initial prompt includes registry summary but not full skill body.
