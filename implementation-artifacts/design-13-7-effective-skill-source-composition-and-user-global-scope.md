# Design: Effective Skill Source Composition and User-Global Scope

**Story:** `13-7-effective-skill-source-composition-and-user-global-scope.md`  
**Design principle:** runtime-managed global defaults, deterministic precedence, single downstream pipeline

---

## Objectives

1. Make user-global skills available by default across projects.
2. Compose `user-global` and `session` sources into one effective list.
3. Keep Story 13.1 discovery and registry injection as the only downstream skill-loading path while reserving a later package hook.
4. Add a minimal Runtime UI to import and manage user-global skills without manual filesystem edits.

---

## Scope

### In scope

- Runtime-managed user-global skill root.
- Shared source-composition service for chat / agent / run.
- Deterministic precedence, dedupe, and collision diagnostics.
- Runtime settings UI and IPC for listing, importing, and removing user-global skills.
- Reserved package-layer extension hook that Story 13.5 can plug into later.

### Out of scope

- Skill activation.
- Supporting file reads.
- Tool narrowing.
- Python / Node / terminal execution bridge.
- Parsing `agents.skills.imports` itself.
- Activating package-layer precedence in this story.

---

## Core Design

### 1. User-Global Skill Root

- Runtime-managed user-global skills live under:

```text
runtime-store/skills/global/
```

- Physical location resolves under `app.getPath('userData')/runtime-store/skills/global/`.
- Runtime aliases these sources under `@state/skills/global/...`.
- Missing root is benign and resolves to an empty `user-global` layer.

### 2. Source Provider Contract

```ts
type SkillSourceLayer = 'user-global' | 'package' | 'session'

type LayeredAttachedSkillSource = AttachedSkillSource & {
  layer: SkillSourceLayer
  sourceKey: string
}

interface EffectiveSkillSourceProvider {
  resolve(params: {
    projectRoot?: string
    packageId?: string
    sessionAttachedSkills?: AttachedSkillSource[]
    packageAttachedSkills?: AttachedSkillSource[]
  }): {
    effectiveSources: LayeredAttachedSkillSource[]
    diagnostics: Array<{
      code: 'SKILL_SOURCE_COLLISION' | 'SKILL_ID_COLLISION' | 'GLOBAL_SKILL_ROOT_UNREADABLE'
      layer?: SkillSourceLayer
      source?: string
      winner?: string
      message: string
    }>
  }
}
```

Responsibilities:

- scan `@state/skills/global`
- accept session-layer inputs
- reserve an optional package-layer input for Story 13.5
- normalize and tag each source with a layer
- enforce precedence `session > user-global` in Story 13.7
- emit structured diagnostics for skipped or colliding sources

### 3. Precedence and Collision Rules

Rules:

- Story 13.7 precedence is `session > user-global`.
- Exact same resolved source keeps only one effective entry.
- Package layer is reserved but inactive until Story 13.5 lands.
- `skillId` collisions across different roots are resolved after discovery:
  - higher-precedence active layer remains visible
  - lower-precedence layer is suppressed
  - a structured diagnostic is recorded

This extends the normalized discovery result with source metadata:

```ts
type ResolvedSkillSummary = {
  skillId: string
  displayName: string
  description: string
  rootPath: string
  skillMdPath: string
  sourceLayer: SkillSourceLayer
  sourceKey: string
  supportingFiles: SkillSupportingFile[]
  allowedTools?: string[]
  disableModelInvocation: boolean
  agentIds?: string[]
}
```

### 4. Integration with Story 13.1

Entry points:

- `chat:send`
- `agent:dispatch`
- `runs:continue`
- prompt dump / debug composition entrypoints

Flow:

1. Resolve user-global sources from Runtime private state.
2. Merge session attachments from payload.
3. Produce `effectiveSources`.
4. Pass `effectiveSources` into the Story 13.1 discovery / registry pipeline.

No parallel discovery path is introduced.

### 5. Package Adapter Hook

Story 13.5 will provide package-layer input to this provider, not bypass it.

Contract:

- adapter output remains `AttachedSkillSource`-shaped
- provider assigns `layer = 'package'`
- Story 13.5 extends precedence to `session > package > user-global`
- all precedence, dedupe, and collision rules stay centralized here

### 6. User-Global Skill Management UI

Placement:

- Add a `Global Skills` section under Runtime settings.
- This story does not introduce a separate skills page; the first management surface lives in `SettingsPage`.

UI capabilities:

- list imported user-global skills with display name / source folder name / validation status where available
- import a skill folder or direct `SKILL.md` source via OS picker
- remove an imported skill entry

Import semantics:

- imports always copy content into `runtime-store/skills/global/`
- the UI does not register arbitrary external absolute paths as durable global skills
- directory imports land under `runtime-store/skills/global/<folderName>/`
- direct `SKILL.md` imports copy the selected file's parent directory so the imported skill keeps its full supporting-file root
- name collisions at import time use deterministic non-destructive folder suffixing instead of overwrite

Suggested IPC surface:

```ts
type GlobalSkillEntry = {
  id: string
  displayName: string
  source: string
  skillMdPath: string
  status: 'ready' | 'invalid'
  error?: string
}

interface GlobalSkillManagementApi {
  listGlobalSkills(): Promise<{
    rootPath: string
    entries: GlobalSkillEntry[]
  }>
  importGlobalSkill(): Promise<{
    imported?: GlobalSkillEntry
    copiedPaths?: string[]
  }>
  removeGlobalSkill(id: string): Promise<{ success: true }>
}
```

Renderer flow:

1. Settings page loads current imported entries from IPC.
2. User clicks `Import Skill`.
3. Main process opens file/folder picker, validates the selected source, copies it into Runtime-managed global storage, and returns the refreshed entry.
4. Renderer refreshes the list and shows success/error feedback.
5. All subsequent chat / agent / run sessions pick up the imported skill automatically through the existing provider.

---

## Error Model

- `GLOBAL_SKILL_ROOT_UNREADABLE`
- `SKILL_SOURCE_COLLISION`
- `SKILL_ID_COLLISION`
- `GLOBAL_SKILL_IMPORT_INVALID`
- `GLOBAL_SKILL_IMPORT_CONFLICT`

Notes:

- missing global root is not an error
- invalid skills inside the root continue to flow through Story 13.1 discovery diagnostics
- UI import errors must not corrupt existing imported skills; failed import attempts should roll back partial copied content

---

## Files to Modify

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`
- `crewagent-runtime/electron/services/globalSkillService.ts` (new)
- `crewagent-runtime/electron/services/globalSkillService.test.ts` (new)
- `crewagent-runtime/electron/services/skillSourceProvider.ts` (new)
- `crewagent-runtime/electron/services/skillSourceProvider.test.ts` (new)
- `crewagent-runtime/electron/services/skillRegistryService.ts`
- `crewagent-runtime/electron/preload.ts`
- `crewagent-runtime/electron/electron-env.d.ts`
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.css`

---

## Test Plan

1. No global root produces an empty `user-global` layer without failing the session.
2. Valid user-global skills appear in chat / agent / run without session attachment.
3. Session source overrides conflicting global skill.
4. Duplicate real paths collapse to one effective source.
5. `skillId` collisions emit diagnostics and keep only the highest-precedence active winner.
6. All entrypoints call the same provider before building the skill registry.
7. Provider accepts a reserved package-layer parameter but does not require or activate it in Story 13.7.
8. Importing a skill copies it into Runtime-managed global storage and refreshes the settings list.
9. Removing an imported global skill removes it from the list and from subsequent auto-discovery.
10. Direct `SKILL.md` imports copy the parent directory, preserving supporting files.
11. Invalid imports roll back copied content instead of leaving partial global entries behind.
