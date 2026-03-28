# Design: Package v1.2 Skill Import Adapter

**Story:** `13-5-package-v1-2-skill-import-adapter.md`  
**Design principle:** adapter only, no duplicated runtime logic

---

## Objectives

1. Reuse the runtime-first skill mechanism for package-bound skills.
2. Keep package parsing separate from runtime discovery/activation/execution logic.
3. Make Epic 11.1 the only schema dependency for this story.

---

## Scope

### In scope

- read `agents.skills.imports`
- convert package declarations to package-layer `attachedSkills`
- preserve agent bindings

### Out of scope

- reimplementing discovery
- direct prompt injection logic
- activation protocol
- supporting file loading

---

## Core Design

### Adapter Contract

```ts
interface PackageSkillImportAdapter {
  toAttachedSkills(params: {
    packageId: string
    effectiveAgentId: string
    agentDefinition: AgentDefinitionV12
  }): AttachedSkillSource[]
}
```

Rules:

- only imports for the current effective agent are emitted
- each package import is resolved relative to `@pkg`
- adapter output must be identical in shape to host-provided `attachedSkills`
- adapted sources are fed into the shared effective skill source composition layer from Story 13.7

### Example Mapping

Package input:

```json
{
  "skills": {
    "imports": [
      { "source": "assets/skills/pptx-tools/" }
    ]
  }
}
```

Adapter output:

```ts
[
  {
    source: "@pkg/assets/skills/pptx-tools/",
    agentIds: ["designer-agent"]
  }
]
```

---

## Files to Modify

- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/electron/services/packageSkillImportAdapter.ts` (new)
- `crewagent-runtime/electron/services/packageSkillImportAdapter.test.ts` (new)
- `crewagent-runtime/electron/main.ts`

---

## Test Plan

1. Package v1.2 imports map to package-layer `attachedSkills`.
2. Relative package paths resolve under `@pkg`.
3. Agent binding is preserved.
4. Runtime-first and package-driven attachments converge on the same effective source composition and downstream discovery path.
