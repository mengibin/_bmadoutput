# Design: Effective Agent, Run Mode, and Audit Alignment for Skills

**Story:** `13-6-effective-agent-run-mode-and-audit-alignment.md`  
**Design principle:** effective-agent truth source, no skill leakage, structured audit first

---

## Objectives

1. Align skill visibility with current `effectiveAgentId`.
2. Keep chat / agent / run behavior consistent.
3. Add structured audit coverage for the skill lifecycle.

---

## Scope

### In scope

- effective-agent skill filtering
- run mode parity
- audit event design
- future subworkflow-safe boundaries

### Out of scope

- hierarchical progress UI work
- package schema design

---

## Core Design

### 1. Effective Agent is the Only Skill Scope

Rules:

- skill registry is computed for the current `effectiveAgentId`
- skill activation is bound to that same agent
- switching effective agent invalidates any active skill state
- run mode tracks the owner agent of the current in-loop skill state and clears it before the next turn if `effectiveAgentId` changes

### 2. Run Mode Parity

- run mode must use the same skill registry / activation / narrowing rules as chat mode
- run mode may inject additional run context, but cannot bypass skill boundaries

### 3. Audit Event Schema

```ts
type SkillAuditEvent =
  | { kind: 'skill.discovered'; skillId: string; source: string }
  | { kind: 'skill.discovery_failed'; source: string; code: string; message: string }
  | { kind: 'skill.activated'; skillId: string }
  | { kind: 'skill.activation_failed'; skillId: string; code: string; message: string }
  | { kind: 'skill.resource_loaded'; skillId: string; relPath: string }
  | { kind: 'skill.resource_load_failed'; skillId: string; relPath: string; code: string; message: string }
  | { kind: 'skill.tools_narrowed'; skillId: string; toolNames: string[] }
```

Every event must include:

- `packageId`
- `workflowId`
- `runId`
- `agentId`
- `timestamp`
- discovery events are emitted where registry resolution happens; activation/resource events are emitted at tool execution time

### 4. Future Subworkflow Safety

Even before Epic 11.2-11.4 land fully:

- service interfaces must already accept `workflowId`, `runId`, and `agentId`
- no global singleton active skill
- any future subworkflow entry will recompute registry and active skill state

---

## Files to Modify

- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/knowledgeOpsService.ts` or equivalent audit consumer
- `crewagent-runtime/electron/stores/runtimeStore.ts`

---

## Test Plan

1. Effective agent switch changes visible skills.
2. Skill activation does not leak across agent changes.
3. Run mode writes audit events with correct identifiers.
4. Discovery / activation / resource load failures are audit-visible.
