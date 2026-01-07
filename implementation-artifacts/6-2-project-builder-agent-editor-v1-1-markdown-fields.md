# Story 6.2: ProjectBuilder Agent Editor (v1.1 Spec Fields + Markdown Editor/Preview)

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Creator**,  
I want the Agent edit modal to fully support the v1.1 Agent manifest fields and provide Step-style Markdown editing + preview for rich-text inputs,  
so that `agents.json` stays schema-valid and authoring experience is consistent across Steps and Agents.

## Acceptance Criteria

### AC-1: Agent Editor aligns with v1.1 template/schema

**Given** I am in ProjectBuilder and open **New agent** / **Edit agent**  
**When** the Agent modal is shown  
**Then** it supports editing all fields defined in the v1.1 agent template (including):  
- `id` (read-only, stable `agentId`)  
- `metadata.name/title/icon` (required)  
- `metadata.module/description/sourceId` (optional)  
- `persona.role/identity/communication_style/principles` (required)  
- `critical_actions[]` (optional)  
- `prompts[]` (optional; each item supports `id/content/description`)  
- `menu[]` (optional; each item supports at least `trigger/description/exec`)  
- `tools.fs.enabled/maxReadBytes/maxWriteBytes` (optional; defaults applied when omitted)  
- `tools.mcp.enabled/allowedServers[]` (optional; defaults applied when omitted)

**And** saving produces a v1.1 manifest that passes the existing `agents.json` schema validation/preview/export.

### AC-2: Markdown editor + preview for rich-text fields (same as Step editor)

**Given** I am editing an Agent  
**When** I click any rich-text field preview  
**Then** a Markdown editor modal opens with **Editor (left)** and **Preview (right)** and the same toolbar/UX as the Step editor  
**And** I can close it via **Close button** or **Esc**  
**And** changes are reflected back into the Agent form and persisted after **Save**.

Rich-text fields include at least:
- `metadata.description`
- `persona.identity`
- `persona.communication_style`
- `persona.principles` (see AC-3)
- `critical_actions` items (see AC-3)
- `prompts[].content` and `prompts[].description`
- `menu[].description`
- (if present in schema) `systemPrompt` / `userPromptTemplate`

### AC-3: List fields edited via Markdown list (normalized to `string[]`)

**Given** I edit list-like fields (e.g. `persona.principles`, `critical_actions`, `tools.mcp.allowedServers`)  
**When** I enter content as either newline-separated text or Markdown bullets (`- item`)  
**Then** the saved data is normalized to `string[]` (trimmed, de-duplicated, empty lines removed)  
**And** the form preview renders as a Markdown list.

### AC-4: Prompts & Menu CRUD

**Given** I want to configure `prompts` or `menu`  
**When** I add, edit, reorder, or remove items  
**Then** the list order is preserved on save/reload  
**And** required fields are validated in the UI:
- `prompts[].id` and `prompts[].content` are required
- `menu[].description` is required

### AC-5: Tool policy editing

**Given** I change tool policy options  
**When** I save the Agent  
**Then** the values are persisted into `agent.tools` and included in `agents.json` preview/export  
**And** defaults match the v1.1 template when omitted:
- `tools.fs.enabled: true`
- `tools.mcp.enabled: false`
- `tools.mcp.allowedServers: []`

## Design

### Summary

- Scope: ProjectBuilder Agent modal (currently in `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`).
- Conformance target:
  - Template: `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/agents.json`
  - Schema (Builder copy): `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/agents.schema.json`
- UX target: reuse the Step editor’s Markdown modal behavior (see `crewagent-builder-frontend/src/components/StepEditorPanel.tsx`).

### UX / UI

- Keep `agentId` read-only (stable).
- Organize the Agent modal into collapsible sections (similar to Step editor sections):
  - **Metadata**: name/title/icon/module/sourceId/description
  - **Persona**: role/identity/communication_style/principles
  - **Critical actions** (list)
  - **Prompts** (list of items)
  - **Menu** (list of items)
  - **Tools**: fs/mcp policy
  - **Advanced** (optional): `systemPrompt`, `userPromptTemplate`, etc.
- For every rich-text field, render a preview box (button) that opens the Markdown editor modal:
  - Preview shows a trimmed excerpt (same heuristic as `previewMarkdown(...)` in Step editor).
  - If empty, show placeholder “Click to open Markdown editor…”.
- For list-like fields (string[]):
  - Preview renders as a Markdown bullet list.
  - Editor content is a Markdown list or newline list; on save normalize back to `string[]`.
- Prompts/Menu list UX:
  - Add/remove buttons.
  - Each item shows a compact row + “Edit” buttons for Markdown fields (content/description).
  - Preserve unknown keys in existing `menu[]` items (do not drop fields not editable in the UI).

### Data / Storage

- No backend changes expected (backend schema already supports v1.1 optional fields).
- Frontend should extend Agent modal state to include the additional optional fields and persist them via existing `PUT /packages/{projectId}/agents`.
- Normalization rules:
  - For `string[]` fields: trim + remove blanks + de-dupe.
  - For `menu[]` items: edit known keys (`trigger/description/exec`) but preserve other keys.

### Errors / Edge Cases

- Editing existing agents with optional fields: values must round-trip without being dropped.
- Validation:
  - Required fields remain enforced (existing v1.1 required set).
  - Numeric limits for `maxReadBytes/maxWriteBytes` must be `>= 1` when present.
  - `tools.mcp.allowedServers` must remain a string list.
- If `agentsJson` is currently invalid (schema errors), Agent modal should block save and surface the error (existing behavior).

### Test Plan

- Manual (frontend):
  - Create agent with required fields only → save → schema preview still passes.
  - Edit optional fields (description, critical_actions, prompts, menu, tools) → save → reload → values persist.
  - Open Markdown editor modal for multiple fields → preview updates live and persists after save.
  - Enter list fields as bullets (`- a`) and as plain lines (`a`) → saved as `string[]` and preview renders list.
- Automated (frontend unit):
  - Add tests for list normalization (Markdown bullets/newlines → `string[]`).
  - Add a round-trip test ensuring unknown `menu[]` keys are preserved when editing supported keys.

## Tasks / Subtasks

- [x] 1) Extract/reuse Markdown editor components from Step editor (preview + modal)
- [x] 2) Extend Agent modal form fields to match v1.1 template (metadata/persona/critical_actions/prompts/menu/tools)
- [x] 3) Implement Markdown preview boxes for all rich-text fields
- [x] 4) Implement list normalization helpers (`string` ⇄ `string[]`) for Markdown lists
- [x] 5) Add prompts/menu CRUD UI (add/edit/remove/reorder) with validation + round-trip preservation
- [x] 6) Add unit tests for normalization + round-trip, run `npm run lint` + `npm run build`

## References

- `crewagent-runtime/spec/bmad-package-spec/v1.1/templates/agents.json`
- `crewagent-builder-frontend/src/lib/bmad-spec/v1.1/agents.schema.json`
- Step Markdown editor UX reference: `crewagent-builder-frontend/src/components/StepEditorPanel.tsx`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Frontend:
  - `npm -C crewagent-builder-frontend run test`
  - `npm -C crewagent-builder-frontend run lint`
  - `npm -C crewagent-builder-frontend run build`

### Completion Notes List

- 抽取 Step 风格的 Markdown Editor/Preview（左右栏编辑+预览、工具栏、Esc 关闭），并复用到 Step 与 Agent 编辑体验。
- ProjectBuilder Agent 编辑弹窗对齐 v1.1 `agents.json` schema：`metadata/persona/critical_actions/prompts/menu/tools`，并补充 Advanced 字段（`systemPrompt/userPromptTemplate/discussion/webskip`）。
- rich-text / list 字段统一通过 Markdown 编辑框编辑：列表字段保存时归一化为 `string[]`（去空行、trim、去重），并在表单中以 Markdown list 预览。
- Prompts/Menu 支持增删改与上下移动排序；Menu 未编辑的额外键会原样保留（round-trip）。
- 增加单元测试覆盖 list 归一化与 menu extra keys round-trip。

### File List

- `_bmad-output/implementation-artifacts/6-2-project-builder-agent-editor-v1-1-markdown-fields.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-frontend/src/app/builder/[projectId]/page.tsx`
- `crewagent-builder-frontend/src/components/MarkdownEditorModal.tsx`
- `crewagent-builder-frontend/src/components/MarkdownPreview.tsx`
- `crewagent-builder-frontend/src/components/StepEditorPanel.tsx`
- `crewagent-builder-frontend/src/lib/agent-menu-v11.ts`
- `crewagent-builder-frontend/src/lib/markdown.ts`
- `crewagent-builder-frontend/tests/agent-menu-v11.test.ts`
- `crewagent-builder-frontend/tests/markdown.test.ts`
- `crewagent-builder-frontend/tsconfig.tests.json`
