# Design: LLM-Driven Skill Activation and Context Mutation

**Story:** `13-2-llm-driven-skill-activation-and-context-mutation.md`  
**Design principle:** main-model self-activation, explicit control tool, ephemeral loop state

---

## Objectives

1. Let the main model decide when to activate a skill.
2. Make activation mutate the next LLM request context, not just produce a plain tool result.
3. Keep activation state loop-local and non-persistent.

---

## Scope

### In scope

- Internal control tool `skill.activate`.
- `ContextMutationResult` contract.
- Chat loop mutation handling.
- Run loop mutation handling.
- Agent mode support through shared context-build path.

### Out of scope

- Supporting file reads.
- Tool narrowing details.
- Package adaptation.

---

## Core Design

### 1. Internal Tool Contract

```ts
type SkillActivateArgs = {
  skillId: string
}
```

Schema behavior:

- only visible skills appear as selectable values
- skills with `disableModelInvocation=true` are excluded

### 2. Control Result

```ts
type ContextMutationResult = {
  ok: true
  mutationType: 'skill_activation'
  skillId: string
  injectedSystemBlocks: string[]
  narrowedToolNames?: string[]
}
```

This result must be interpreted by the loop controller, not just appended as a normal tool result.

### 3. Loop-local State

```ts
type SkillLoopState = {
  activeSkillId: string | null
  injectedSystemBlocks: string[]
  narrowedToolNames?: string[]
}
```

Rules:

- initialized empty at the start of each tool loop
- updated only by `skill.activate`
- discarded when the loop ends

### 4. Chat Loop Changes

`chatToolLoop.ts` must:

1. execute `skill.activate`
2. detect `ContextMutationResult`
3. update `SkillLoopState`
4. rebuild the next LLM request with new system blocks
5. recompute visible tools before the next turn

### 5. Run Loop Changes

`executionEngine.ts` must:

1. keep a run-local in-memory `SkillLoopState`
2. rebuild `requestMessages` after activation
3. recompute visible tools for the next LLM turn
4. avoid writing skill activation state into conversation history or workflow state

---

## Prompt Mutation Rules

Activated skill injection must produce:

```md
## Activated Skill
Skill: `pptx-tools`
Root: `@skill/pptx-tools/`

<full skill instructions here>
```

Order:

1. base rules
2. tool policy
3. persona/run context
4. activated skill blocks
5. conversation history

---

## Error Model

- `SKILL_ACTIVATION_NOT_ALLOWED`
- `SKILL_NOT_VISIBLE`
- `SKILL_ALREADY_ACTIVE`
- `SKILL_ACTIVATION_FAILED`

Repeated activation of the same skill in a single loop should return a benign result rather than duplicating system blocks.

---

## Files to Modify

- `crewagent-runtime/electron/services/toolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/agentContextBuilder.ts`

---

## Test Plan

1. Model can see `skill.activate` only when there are activatable skills.
2. Activation injects full skill instructions on the next turn.
3. Re-activating same skill does not duplicate injection blocks.
4. Activation does not persist across independent user turns.
5. Run mode and chat mode both rebuild context after activation.
