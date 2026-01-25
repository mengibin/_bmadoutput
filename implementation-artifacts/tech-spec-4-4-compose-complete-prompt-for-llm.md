# Tech Spec: Story 4-4 - Compose Complete Prompt for LLM

## Overview

The `PromptComposer` is a critical service in the Execution Engine that transforms static workflow definitions and dynamic runtime state into a structured `messages[]` array for the LLM. It employs a **Layered Prompt** strategy to ensure the model receives clear boundaries, correct tool policies, and precise navigation instructions.

## Architecture

### Prompt Layering Strategy

The prompt is constructed in a fixed order of layers to establish context hierarchy.

**By mode:**

- `mode=run` (workflow execution; workflow 未完成/存在 active step):
  1. System: BaseRuntimeRules (workflow/run)
  2. System: ToolPolicy
  3. System: Persona
  4. User: RunDirective
  5. User: NodeBrief
  6. User: UserInput (Optional, includes `forNodeId`)

- `mode=run` (Post-Completion Profile; workflow 已 completed):
  1. System: BaseRuntimeRules (workflow/run)
  2. System: ToolPolicy
  3. System: Persona
  4. User: RunDirective (includes Post-Run Protocol; omits `currentNodeId`)
  5. User: UserInput (Optional, no `forNodeId`)
  - No `NodeBrief` and no step markdown injection in this profile. See Story 7-11.

- `mode=agent` (agent session):
  1. System: BaseRuntimeRules (conversation)
  2. System: ToolPolicy
  3. System: Persona
  4. User: UserInput (Optional)

- `mode=chat` (free chat, tool-enabled):
  1. System: BaseRuntimeRules (conversation)
  2. System: ToolPolicy
  3. User: UserInput (Optional)

Notes:
- `mode=chat` intentionally omits Persona so users can “just chat”, but tools can still be used via ToolPolicy.
- ToolPolicy is still derived from `agentDefinition.tools` (choose a default agent purely for tool policy if needed).

The layer semantics remain:

1.  **System Layer 1: BaseRuntimeRules**
    -   Immutable rules for the Runtime environment.
    -   Defines mount points (`@project`, `@pkg`, `@state`).
    -   Enforces `workflow.graph.json` as the source of truth.
2.  **System Layer 2: ToolPolicy**
    -   Derived from the active Agent's configuration.
    -   Explicitly enables/disables tools (fs, mcp) and sets limits (e.g., `maxReadBytes`).
3.  **System Layer 3: Persona**
    -   Defines the agent's identity, role, and communication style.
    -   Prioritizes `systemPrompt` (compiled) > `persona` (schema-rendered).
4.  **User Layer 4: RunDirective**
    -   Standardized block for specific instructions (Start/Continue/Resume).
    -   Points to aliased paths for Graph and State.
5.  **User Layer 5: NodeBrief**
    -   Context for the `currentNodeId`.
    -   Lists specific validation rules, inputs, outputs, and allowed transitions.
6.  **User Layer 6: UserInput (Optional)**
    -   Wraps raw user text to prevent Prompt Injection or Context Confusion.
    -   Clearly associates input with the current Step/Node.

### Module: `PromptComposer`

Located in: `electron/services/promptComposer.ts`

#### Input Interface: `ComposeInput`
```typescript
interface ComposeInput {
    mode: 'run' | 'agent' | 'chat'
    // ... path contexts (projectRoot, packageId...)
    agentDefinition: AgentDefinition
    // ... runtime state (currentNodeId, variables...)
    nodeBrief?: {
        nodeId: string
        stepFilePathInPkg: string
        transitions: Edge[]
        // ...
    }
    // ... user interaction (intent, userInput)
}
```

#### Output: `Message[]`
```typescript
[
  { role: 'system', content: "..." },
  { role: 'user', content: "..." }
]
```

## Implementation Details

### RuntimeStore Integration
-   **New Method**: `getAgentDefinition(packageId, agentId)`
-   **Logic**: Reads `agents.json` from the package, validating against `v1.1` schema.
-   **Purpose**: Provides the detailed `tools` and `persona` configuration required by the Composer.

### Path Aliasing
-   **Security**: Real filesystem paths are never exposed to the LLM.
-   **Mounts**:
    -   `@project`: User's local project root.
    -   `@pkg`: The BMAD package content (read-only).
    -   `@state`: Running state directory (hidden).

### Edge Cases
-   **Missing System Prompt**: Fallback to rendering `persona` object (Identity + Principles).
-   **Missing User Template**: Fallback to generic "Task" format.
-   **Empty User Input**: Do not append the User Input layer.
-   **Chat Mode Persona**: `mode=chat` MUST NOT inject Persona (chat stays generic); ToolPolicy still applies.
-   **User Input Binding**: `forNodeId` is only emitted in `mode=run` when workflow is not completed (active step exists). Post-Completion Profile omits `forNodeId`. (See Story 7-11.)

## Testing Strategy
-   **Unit Tests**: Verify string inclusion of critical instructions (Mounts, Tool Policy).
-   **Scenario Tests**:
    -   **Start Run**: Verify `intent: start` and entry node brief.
    -   **Continue Run**: Verify `intent: continue` and updated node brief.
    -   **Override**: Verify `systemPrompt` field overrides schema rendering.
