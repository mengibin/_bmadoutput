# Story 4.4: Compose Complete Prompt for LLM

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Runtime**,
I want to assemble the complete prompt (Agent Persona + Step Instructions + Context + Tool Policy),
so that the LLM receives full context and stays within runtime guardrails.

## Acceptance Criteria

1. **Prompt Assembly**
   - Agent persona/prompt template from `agents.json` (by `effectiveAgentId = node.agentId ?? run.activeAgentId`)
   - Current node step instructions are discoverable via `@pkg/<node.file>` (step `.md`) and the prompt explicitly instructs the model to `fs.read` it (avoid inlining large step files)
   - Context injection (Progressive Disclosure):
     - Inject minimal summaries (do not paste large files)
     - Encourage `fs.search → fs.read(window)` for large files
     - Inject `variables` from `@state/workflow.md` (structured JSON or placeholder substitution)
   - Tool policy derived from `agents.json.tools` (fs/mcp enabled + limits like `maxReadBytes/maxWriteBytes`)
   - Mounts & rules:
     - Define `@project`, `@pkg`, `@state` mount points
     - Enforce `@pkg` as read-only
     - Default artifact output to `@project/artifacts/`

2. **Machine-Parseable User Shells**
   - Emit strict blocks aligned with `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`:
     - `RUN_DIRECTIVE` (start/continue/resume)
     - `NODE_BRIEF` (当 workflow 未完成且存在 active step 时；至少包含 currentNodeId, stepFile, outputsMap, allowedNext)
     - `USER_INPUT` (when applicable；Post-Completion Profile 不包含 `forNodeId`)

3. **Security & Privacy**
   - NEVER expose real filesystem paths (e.g., `/Users/...`) to the LLM; only mount alias paths (`@project/...`, `@pkg/...`, `@state/...`)
   - Sanitized injection: injected variables and brief fields must not leak internal absolute paths (RuntimeStoreRoot, temp paths, etc.)

## Technical Context

- **Module**: `crewagent-runtime/electron/services/promptComposer.ts`
- **Integration**: Called by ExecutionEngine before sending `messages[]` to LLMAdapter (Story 4.5)
- **Data Sources**:
  - `RuntimeStore.getAgentDefinition(packageId, agentId)` (agents.json, schema validated)
  - `RuntimeStore.getWorkflowState(packageId, workflowId, runId, projectRoot)` (`@state/workflow.md` → `currentNodeId/variables/...`)
  - `RuntimeStore.loadWorkflow(packageId, workflowId)` (graph + step files)
- **Reference (must follow)**: `_bmad-output/tech-spec/prompt-composer-examples.md`

## Design

### Summary

- Compose prompts in fixed layers/order:
  - `mode=run`: Base Rules → Tool Policy → Persona → RUN_DIRECTIVE → NODE_BRIEF → USER_INPUT (optional).
  - `mode=run`（Post-Completion Profile，workflow 已 completed）：Base Rules → Tool Policy → Persona → RUN_DIRECTIVE（含 Post-Run Protocol；不含 currentNodeId）→ USER_INPUT（无 `forNodeId`；可选）。不发送 `NODE_BRIEF`，不注入 step 内容。详见 Story 7-11。
  - `mode=agent`: Base Rules → Tool Policy → Persona → USER_INPUT (optional).
  - `mode=chat`: Base Rules → Tool Policy → USER_INPUT (optional). (No explicit agent persona, but tools still follow Tool Policy.)
- `RUN_DIRECTIVE.graph` MUST use `@pkg/<graphRelPath>` from the selected workflow record (no hardcoded `@pkg/workflow.graph.json`).
- Step instructions are referenced by `@pkg/<node.file>`; do not inline large files—LLM must `fs.read` them on demand.
- All model-visible paths are mount aliases only (`@project/@pkg/@state`); absolute-path leakage is blocked by tests.
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-4-compose-complete-prompt-for-llm.md`

### UX / UI
- N/A (backend service; surfaced indirectly via Logs panel).

### API / Contracts
- Service API:
  - `PromptComposer.compose(input: ComposeInput): Message[]` (OpenAI-compatible roles: `system|user`)
- Required input data (ExecutionEngine responsibility):
  - `effectiveAgentId = node.agentId ?? run.activeAgentId`
  - `graphRelPath` from selected workflow record → emit as `@pkg/<graphRelPath>`
  - `stepRelPath` from graph node → emit as `@pkg/<node.file>`
  - `currentNodeId` + `variables` from `@state/workflow.md` (via `RuntimeStore.getWorkflowState(...)`)
- Text shells (must be machine-parseable, aligned with `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`):
  - `RUN_DIRECTIVE` (example shape)
    - `- runType: bmad-micro`
    - `- intent: start|continue|resume`
    - `- workflow: <workflowId>`
    - `- state: @state/workflow.md`
    - `- graph: @pkg/<graphRelPath>`
    - `- artifactsRoot: @project/artifacts/`
    - `- currentNodeId: <currentNodeId>`（仅 workflow 未完成/存在 active step 时）
    - `- effectiveAgentId: <effectiveAgentId>`
    - `- autopilot: true|false`
  - `NODE_BRIEF` (minimum fields)
    - `- currentNodeId: <id> (type=<nodeType>)`
    - `- stepFile: @pkg/<node.file>`
    - `- outputsMap:` (project-relative → alias)
    - `- allowedNext:` (each edge: `to/label/isDefault/conditionText`)
  - `USER_INPUT`
    - `mode=run`（workflow 流程中）：
      - `- forNodeId: <currentNodeId>`
      - `<raw user text>`
    - `mode=run`（Post-Completion Profile，workflow 已 completed）：
      - `<raw user text>`（不绑定 node；不包含 `forNodeId`）
    - `mode=agent/chat`（非流程对话）：
      - `<raw user text>`（不需要绑定节点）

### Data / Storage
- Reads:
  - `agents.json` (via `RuntimeStore.getAgentDefinition`)
  - Workflow graph + step markdown (via `RuntimeStore.loadWorkflow`)
  - Run state frontmatter (`@state/workflow.md`) (via `RuntimeStore.getWorkflowState`)
- Does not write to disk (writes happen via ToolHost / later stories).

### Errors / Edge Cases
- Missing agent definition for `effectiveAgentId` → block execution with a clear error (do not silently drop persona/tool policy).
- Missing step file for `currentNodeId` → block (align with Story 4.2 load behavior).
- `currentNodeId` invalid per graph → block (align with Story 4.3).
- Variable sanitization: if injected variables contain absolute paths, redact/alias them before including in any message content.
- Workflow already completed (Post-Completion Profile): do not inject active step info (`NODE_BRIEF`/stepFile/step markdown; omit `currentNodeId/forNodeId`). Keep `RUN_DIRECTIVE` with Post-Run Protocol and allow continued conversation. See Story 7-11.
- Provider limitations: if a provider cannot handle multiple `system` messages, merge system layers before sending (LLMAdapter responsibility).

### Test Plan
- Unit tests (`crewagent-runtime/electron/services/promptComposer.test.ts`):
  - Layer ordering (Rules/Policy/Persona/RUN_DIRECTIVE/NODE_BRIEF/USER_INPUT).
  - `systemPrompt` override vs schema-rendered persona.
  - Strict shell headers present: `RUN_DIRECTIVE`, `NODE_BRIEF`, `USER_INPUT`.
  - `NODE_BRIEF.allowedNext` matches graph edges for current node (includes `label/isDefault/conditionText`).
  - No absolute path leakage in `messages[].content` (denylist: `/Users/`, `C:\\`, `runtime-store/`).

## Tasks / Subtasks

- [x] Reuse/extend `RuntimeStore.getAgentDefinition(packageId, agentId)` (validate against `agents.schema.json` v1.1; define structured error shape if needed).
- [x] Extend `PromptComposer` to always emit the step file reference (`@pkg/<node.file>`) + protocol guidance to read it via tools (`fs.read`), per `prompt-composer-examples.md` (avoid inlining large step files).
- [x] Emit strict `RUN_DIRECTIVE` / `NODE_BRIEF` / `USER_INPUT` blocks per protocol spec.
- [x] Add Progressive Disclosure guidance (`fs.search → fs.read(window)`) + tool limits to base rules/policy layer.
- [x] Ensure `RUN_DIRECTIVE.graph` uses the selected workflow’s `graph` path (no hardcoded `@pkg/workflow.graph.json`).
- [x] Define `NODE_BRIEF` minimum fields + formatting (currentNodeId, stepFile, outputsMap, allowedNext with label/default/conditionText).
- [x] Add/extend unit tests for alias-only output, transition listing, and absolute-path leakage scanning.

## Validation Notes

Merged from: `_bmad-output/implementation-artifacts/validation-report-story-4-4.md`

- **Verdict**: Approved for `design-story` (minor recommendations).
- **Design must lock in**:
  - `RUN_DIRECTIVE.graph: @pkg/<graphRelPath>` comes from workflow selection (no hardcoding).
  - `NODE_BRIEF` minimum field set + allowedNext includes `label/default/conditionText`.
  - Tests enforce **no absolute paths** in `messages[].content` (denylist: `/Users/`, `C:\\`, `runtime-store/`).

## References

- Epics: `_bmad-output/epics.md` (Story 4.4)
- Tech Spec: `_bmad-output/implementation-artifacts/tech-spec-4-4-compose-complete-prompt-for-llm.md`
- Architecture: `_bmad-output/architecture/runtime-architecture.md` (PromptComposer section)
- Examples: `_bmad-output/tech-spec/prompt-composer-examples.md`
- Protocol: `_bmad-output/tech-spec/llm-conversation-protocol-openai.md`
- Validation: `_bmad-output/implementation-artifacts/validation-report-story-4-4.md`

## Change Log

- 2026-01-01: Implemented PromptComposer prompt layering + strict shells + state variable sanitization; added unit tests.
- 2026-01-01: Code review fixes: sanitize DataContext injection; keep variables summary as valid JSON when truncated.
- 2026-01-24: Documented Post-Completion Run prompt profile (no active step injection); see Story 7-11.

## Dev Agent Record

### Completion Notes

- Implemented `RUN_DIRECTIVE` with `runType` + workflow graph alias path (no hardcoding).
- Implemented `NODE_BRIEF` with `outputsMap` + `allowedNext` derived from graph edges.
- Added `STATE_SUMMARY` injection with variable sanitization (redacts absolute paths).
- Tests: `npx vitest run`; Build: `npm -C crewagent-runtime run build:ci`.

### File List

- `crewagent-runtime/electron/services/promptComposer.ts`
- `crewagent-runtime/electron/services/promptComposer.test.ts`
