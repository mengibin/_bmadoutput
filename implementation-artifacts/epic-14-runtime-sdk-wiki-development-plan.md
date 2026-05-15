# Epic 14 Development Plan: Runtime SDK Wiki and Trusted SDK Tool Governance

Status: planning-complete

## Purpose

本计划基于对 `crewagent-runtime` 现有代码的阅读，给 Epic 14 制定 BMAD 风格的开发顺序、故事边界、复用点、风险和验证策略。2026-05-15 修订版明确：Runtime 不再承担 SDK execution safety 硬门禁，执行安全由 MCP/集成软件负责；Runtime 负责工具可见性、治理 metadata 和 audit。

## Inputs

- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Epic breakdown: `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- Source requirements: `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/`
- Runtime code read scope:
  - `crewagent-runtime/electron/stores/runtimeStore.ts`
  - `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - `crewagent-runtime/electron/services/toolHost.ts`
  - `crewagent-runtime/electron/services/projectKbService.ts`
  - `crewagent-runtime/electron/services/chatToolLoop.ts`
  - `crewagent-runtime/electron/services/executionEngine.ts`
  - `crewagent-runtime/electron/services/skillRegistryService.ts`
  - `crewagent-runtime/electron/services/skillActivation.ts`
  - `crewagent-runtime/electron/main.ts`
  - `crewagent-runtime/electron/preload.ts`
  - `crewagent-runtime/shared/agentToolPolicy.ts`
  - `crewagent-runtime/shared/conversationTypes.ts`
  - `crewagent-runtime/src/pages/KnowledgePage/KnowledgePage.tsx`
  - `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
  - `crewagent-runtime/src/stores/appStore.ts`

---

## Current Runtime Findings

### 1. RuntimeStore already has the right storage patterns

`RuntimeStore` owns the private `runtime-store` root and already separates packages, project runtime data, global skills, settings, personal KB, and project KB. It also has:

- safe alias resolution via `resolveAbsolutePath`;
- project runtime root derivation by `projectId`;
- atomic write helpers used throughout the store;
- staged import pattern in package import and project KB archive import;
- per-project knowledge ops logs;
- run-state execution logs under `@state/runs/<runId>/...`.

Implication for Epic 14:

- Add SDK Wiki storage as a new RuntimeStore domain rather than putting it under project KB.
- Use the existing `projectKb` style manifest/ops-log pattern.
- Do not expose real filesystem paths to renderer payloads unless explicitly internal.

### 2. ProjectKbService is the closest implementation model

`ProjectKbService` already implements:

- runtime-private source copying;
- archive import/export with `AdmZip`;
- safe archive path validation;
- manifest-based status;
- staged import workspace and rollback;
- chunk index with SQLite bridge;
- retrieval through ToolHost callback.

Implication for Epic 14:

- `SdkWikiService` should follow the same style: service class with RuntimeStore dependency, manifest/registry IO through RuntimeStore, and focused tests.
- SDK Wiki import must be simpler than Project KB import: validate and register external index files; do not extract/chunk/rebuild.

### 3. FileSystemToolHost is the central internal tool surface

`FileSystemToolHost.getVisibleTools()` currently registers:

- `ui.ask_user`
- skill tools
- `fs.*`
- `media.extract`
- `project_kb.retrieve`
- `python.run`
- `terminal.run`
- `node.run`
- `npm.install`
- `shell.exec`
- MCP tools

`executeToolCall()` centralizes:

- JSON argument validation;
- policy checks for MCP, terminal, fs;
- dispatch to internal tools;
- execution audit append;
- skill activation/resource audit.

Implication for Epic 14:

- Add `sdk_wiki.*` to `ToolHost`/`ToolResult` and `FileSystemToolHost`.
- Prefer a callback or injected service approach like `onProjectKnowledgeRetrieve`, so SDK Wiki logic remains in `SdkWikiService`.
- Keep SDK execution governance wrappers at the ToolHost boundary for audit, not as a Runtime execution gate.

### 4. Prompt/system injection already supports dynamic registry blocks

Skill registry integration uses:

- `SkillSourceProvider`
- `SkillRegistryService`
- `formatSkillRegistrySystemMessage`
- `injectExtraSystemMessages`
- chat / agent / run entrypoint plumbing in `main.ts` and `executionEngine.ts`

Implication for Epic 14:

- SDK Wiki registry can reuse the same "compact registry as extra system messages" pattern.
- Do not load full SDK Wiki pages into the initial prompt.
- Run mode needs integration inside `ExecutionEngine` in addition to chat/agent in `main.ts`.

### 5. SDK execution safety belongs to MCP; Runtime keeps observability

`ui.ask_user` supports `confirmation` widgets through `WidgetPayload` and `WIDGET_SUBMIT` user messages, but SDK tool execution should not use a Runtime confirmation-token loop. The integration MCP software owns path safety, destructive safety, solve resource/license safety, and domain-specific confirmations before exposing or executing a tool.

Implication for Epic 14:

- Story 14.5 must remove the Runtime SDK confirmation gate and token contract.
- Runtime should observe SDK governance metadata, append audit events, and continue dispatch when existing effective tool policy allows the underlying MCP/local tool.
- Missing governance metadata should be logged as warning, not treated as an execution blocker.

### 6. Existing tool policy has no knowledge category

`AgentToolPolicy` currently contains `fs`, `mcp`, and `terminal`. `project_kb.retrieve` is currently exposed inside the `fs.enabled` branch.

Implication for Epic 14:

- MVP can expose `sdk_wiki.*` under the same internal knowledge/tool surface branch as `project_kb.retrieve`, while documenting that `sdk_wiki.*` is read-only and does not grant filesystem access.
- A future `knowledge` policy category can split KB/SDK Wiki visibility, but that is not required for Epic 14 MVP.

---

## BMAD Execution Rule for Epic 14

Epic 14 should execute strictly serially:

```text
14.1 -> 14.2 -> 14.3 -> 14.4 -> 14.5 -> 14.6
```

Story 14.1 through Story 14.6 are now `done` after approved BMAD implementation and review. Epic 14 has completed the SDK Wiki import/query/planning/governance/golden-path sequence.

Rationale:

- 14.2 cannot safely expose `sdk_wiki.*` until import/registry semantics are stable.
- 14.3 is required because Runtime users need a visible import/remove surface before SDK Wiki capabilities can be operated safely.
- 14.4 depends on page/symbol/relation result shape from 14.2 and management UX from 14.3.
- 14.5 needs the final tool naming, audit path, and MCP trust-boundary convention to avoid rework.
- 14.6 is an integration validation story and should not begin before 14.1~14.5 are implemented.

---

## Story Plan

### Story 14.1: SDK Wiki Pack Import and Registry

Current status: `done`

Primary goal:

- Establish `RuntimeStore/sdk-wikis/<sdkId>/<version>/`, `registry.json`, validation reports, import logs, and IPC/service API to list installed SDK Wikis.

Expected implementation scope:

- New service: `crewagent-runtime/electron/services/sdkWikiService.ts`
- New tests: `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- RuntimeStore additions for SDK Wiki root, registry path, temp import workspace, ops log.
- Optional main/preload IPC for developer import/list entrypoints.

Key design choices:

- Import from both directory and archive.
- Validate `sdk-wiki.json` and `index/manifest.json`.
- Reject missing `index/manifest.json`.
- Reject unsupported schema/index schema instead of rebuilding.
- Validate page frontmatter.
- Validate hashes if present.
- Commit atomically by staged directory then registry update.

Test focus:

- valid fixture registers `sam`;
- missing `index/manifest.json` fails;
- unsupported schema fails;
- hash mismatch fails;
- invalid page frontmatter fails;
- failed import leaves registry unchanged.

### Story 14.2: SDK Wiki Search, Read, Symbol, and Relation Module

Current status: `done`

Primary goal:

- Add read-only internal `sdk_wiki.list_sdks`, `sdk_wiki.search_pages`, `sdk_wiki.read_page`, `sdk_wiki.resolve_symbol`, and `sdk_wiki.expand_relations`.

Expected implementation scope:

- Extend `ToolResult` union.
- Add `onSdkWikiToolCall` or direct `SdkWikiService` dependency to `FileSystemToolHost`.
- Add visible tools when SDK Wiki registry has ready entries.
- Add compact registry system message in chat/agent/run.

Key risks:

- Index format may evolve. The service should keep index readers behind a small interface.
- Tool result must return bounded snippets to avoid context bloat.

### Story 14.3: SDK Wiki Pack Management UI and Remove

Current status: `done`

Primary goal:

- Add the visible Runtime operation surface for installing, listing, inspecting, and removing SDK Wiki Packs.

Expected implementation scope:

- Settings page `SDK Wiki Packs` subsection.
- Renderer store actions for list/import directory/import archive/remove.
- Main/preload IPC for remove if not already present.
- `SdkWikiService.removeInstalled(sdkId, version)` and RuntimeStore helpers for registry update plus pack directory deletion.
- Import validation report/error display after failed imports.
- Delete confirmation before removing installed packs.
- Refresh list after import/remove.

Key design choices:

- Place the first UI under Settings rather than a new top-level page.
- Keep deletion transactional: registry and installed files must not diverge on failure.
- Do not expose absolute RuntimeStore paths in renderer payloads.
- Treat directory import and archive import as separate buttons so the user intent is clear.

Test focus:

- list existing SDK Wiki Packs in Settings;
- import directory via dialog and refresh list;
- import archive via dialog and refresh list;
- failed import shows validation error/report without changing list;
- remove installed pack with confirmation;
- remove failure leaves registry unchanged;
- no absolute private paths appear in renderer-facing state.

### Story 14.4: SDK API Usage Planning with Source References

Status after this planning pass: `done`

Primary goal:

- Add `sdk_wiki.plan_api_usage` that returns a source-referenced planning scaffold and `missingInformation`.

Expected implementation scope:

- Deterministic retrieval scaffold inside `SdkWikiService`;
- optional main LLM prompt guidance to use the tool before SDK API claims;
- result schema for `taskType`, `requiredApis`, `executionPlan`, `missingInformation`, `confidence`.

Key risks:

- Do not hide LLM calls inside SDK services. If LLM reasoning is needed, it must stay in the main Runtime loop.
- Avoid returning plan steps without source refs.

### Story 14.5: Trusted SDK Tool Governance and Audit

Status after this planning pass: `done`

Primary goal:

- Rework SDK execution tooling so Runtime observes `read/model_write/file_write/solve/destructive` metadata and writes audit logs, while MCP/integration software owns execution safety and Runtime does not ask for user confirmation solely because of SDK risk.

Expected implementation scope:

- New service: `sdkToolRiskPolicyService.ts`;
- risk metadata contract for SDK/MCP tool adapters;
- audit append to existing execution JSONL and UI log;
- no one-shot confirmation token contract;
- no Runtime SDK `ui.ask_user` confirmation path;
- tests proving solve/destructive/file_write metadata does not block autonomous execution and metadata warnings are auditable.

Key codebase constraint:

- Existing ToolHost remains the central execution boundary, so the governance hook should run there after existing effective tool policy checks and before dispatch.

Recommended MVP governance contract:

1. SDK/MCP adapter optionally supplies `SdkToolRiskEnvelope`.
2. Runtime normalizes metadata and builds a stable argument fingerprint.
3. Runtime appends `sdk_tool.requested` or `sdk_tool.metadata_warning`.
4. Runtime dispatches the underlying tool if existing effective tool policy allows it.
5. Runtime appends `sdk_tool.executed` or `sdk_tool.failed`.

### Story 14.6: SAM Golden Path and Generic SDK Adapter Contract

Status after this planning pass: `done`

Primary goal:

- Validate end-to-end behavior with SAM without hardcoding SAM into Runtime.

Expected implementation scope:

- SAM SDK Wiki fixture or test pack;
- import + list + search + read + plan tests;
- governance-audited mock SAM tool calls;
- developer documentation for generic SDK pack/adapter contract.

Key acceptance target:

- Query `施加压力载荷` surfaces Pressure, Surface, and StaticStep related pages with source refs.
- Solve operation executes autonomously when the MCP exposes the tool and effective tool policy allows it.
- Destructive operation safety is validated at the MCP/integration boundary and Runtime audit remains traceable.

---

## File Ownership Plan

### Main process

- `crewagent-runtime/electron/stores/runtimeStore.ts`
  - SDK Wiki root/registry/temp path helpers.
  - Atomic registry write and import log append.
- `crewagent-runtime/electron/services/sdkWikiService.ts` new
  - Import, validation, registry listing, index reader, `sdk_wiki.*` handlers.
- `crewagent-runtime/electron/services/sdkToolRiskPolicyService.ts` new in Story 14.5
  - Risk metadata normalization, metadata warnings, argument fingerprinting, decision logging.
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
  - Tool schema exposure, `executeToolCall` routing, SDK Wiki/Risk callbacks.
- `crewagent-runtime/electron/services/toolHost.ts`
  - Tool result union additions.
- `crewagent-runtime/electron/main.ts`
  - Service construction, IPC handlers, chat/agent registry injection.
- `crewagent-runtime/electron/preload.ts`
  - IPC wrappers if a UI/developer import path is included.
- `crewagent-runtime/electron/services/executionEngine.ts`
  - Run-mode SDK Wiki registry injection and SDK governance audit behavior.

### Renderer

Renderer work starts in 14.3 because SDK Wiki Pack import/remove requires a visible Runtime operation surface.

- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`
- optional `crewagent-runtime/src/pages/SettingsPage/SdkWikiPacksSection.tsx` if the Settings page needs extraction

### Tests

- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/services/executionEngine.test.ts`
- `crewagent-runtime/electron/services/chatToolLoop.test.ts`
- `crewagent-runtime/electron/stores/runtimeStore.test.ts`

---

## Validation Commands

Recommended per story:

```bash
cd crewagent-runtime && npx vitest run electron/services/sdkWikiService.test.ts
cd crewagent-runtime && npx vitest run electron/services/fileSystemToolHost.test.ts -t "sdk_wiki"
cd crewagent-runtime && npx vitest run electron/services/chatToolLoop.test.ts electron/services/executionEngine.test.ts -t "sdk"
cd crewagent-runtime && npx tsc --noEmit
```

If renderer UI is touched:

```bash
cd crewagent-runtime && npx vitest run src/stores/appStore.test.ts src/pages/SettingsPage/SettingsPage.test.tsx
```

---

## Key Risks and Mitigations

| Risk | Impact | Mitigation |
|:---|:---|:---|
| SDK Wiki index schema unclear or evolving | Rework in 14.2/14.4 | 14.1 validates manifest but hides reader behind `SdkWikiIndexReader` contract |
| Full SDK Wiki content bloats context | Poor LLM reliability | Expose compact registry and bounded page/snippet reads only |
| SDK Wiki Pack cannot be managed from UI | Feature unusable without developer IPC calls | 14.3 adds Settings import/list/remove and validation report display |
| Runtime confirmation gate fights Agent autonomy | Tool execution becomes awkward and duplicates MCP safety | 14.5 removes Runtime SDK confirmation and keeps audit-only governance |
| SDK execution tools arrive through MCP with unknown risk | Observability gap | Log missing/invalid metadata as warning; do not block execution solely for metadata |
| SAM-specific assumptions leak into generic Runtime | Future SDK integrations become expensive | Keep `sdkId` generic and put SAM details only in 14.6 fixtures |

---

## Development Readiness

Completed:

- Story 14.1 imports and registers SDK Wiki Packs.
- Story 14.2 exposes SDK Wiki search/read/symbol/relation tools.
- Story 14.3 provides Settings import/list/remove UI.
- Story 14.4 provides source-referenced API usage planning.
- Story 14.5 provides trusted MCP SDK tool governance and audit.
- Story 14.6 validates the SAM golden path and documents the generic SDK adapter contract.
