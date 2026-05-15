# Tech-Spec: Story 14.6 SAM Golden Path and Generic SDK Adapter Contract

**Created:** 2026-05-15
**Status:** Implemented; Approved
**Source Story:** `_bmad-output/implementation-artifacts/14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`

## Overview

### Problem Statement

Stories 14.1~14.5 implemented SDK Wiki import/management/query/planning and trusted MCP SDK governance audit. The final Epic 14 slice must prove the pieces work together for a realistic SAM pressure-load path while keeping Runtime generic.

The goal is not to add SAM-specific production code. The goal is to validate:

- SAM SDK Wiki Pack import/list/search/read/plan behavior;
- source-referenced Pressure、Surface、StaticStep API planning;
- missing-knowledge behavior;
- SDK governance audit for model_write and solve operations;
- reusable adapter contract for future SDKs.

### Solution

Add focused golden-path coverage and a contract artifact:

- extend `SdkWikiService` tests with an explicit Story 14.6 SAM golden path;
- extend `FileSystemToolHost` tests with a mock SAM adapter resolver for `model_write` and `solve` governance metadata;
- add `_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`.

## Scope

In scope:

- SAM fixture-based SDK Wiki import/list/search/read/plan tests;
- mock SAM governance-audit ToolHost tests;
- adapter contract documentation;
- BMAD story, tech spec, code review, and sprint status.

Out of scope:

- real SAM MCP server;
- real CAE model mutation or solve;
- SAM-specific production branching;
- Runtime SDK confirmation gate.

## Golden Path

1. Build a SAM SDK Wiki fixture with:
   - `sdk-wiki.json`;
   - required `index/*.json`;
   - `wiki/api/Pressure.md`;
   - `wiki/api/Surface.md`;
   - `wiki/api/StaticStep.md`;
   - `wiki/workflows/apply-pressure-load.md`;
   - `source_refs` pointing to `raw/manual.md`.
2. Import through `SdkWikiService.importFromPath`.
3. Confirm `sdk_wiki.list_sdks` includes `sam@0.1.0`.
4. Confirm search/read can resolve SAM pages without leaking RuntimeStore absolute paths.
5. Call `sdk_wiki.plan_api_usage` with `给当前板架模型施加向下均布压力载荷`.
6. Confirm returned plan:
   - `taskType=apply_pressure_load`;
   - includes Pressure、Surface、StaticStep;
   - has source refs for every API and plan step;
   - has no `missingInformation`.
7. Execute mock underlying tools with governance metadata:
   - `sam.model.apply_pressure_load` risk `model_write`;
   - `sam.solve.submit_static` risk `solve`.
8. Confirm ToolHost dispatches without SDK confirmation and writes SDK audit events.

## Generic Adapter Contract

The adapter contract is documented separately in:

`_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`

Required rules:

- SDK Wiki Pack follows the builder-compatible package/page/hash rules already enforced by Runtime.
- SDK semantic planning stays in Runtime main LLM and `sdk_wiki.*`; MCP tools do not call LLMs for SDK understanding.
- MCP/integration software owns execution safety.
- Runtime may observe SDK execution metadata via `SdkToolRiskEnvelope`.
- Runtime audit must remain generic and `sdkId`-based, not SAM-specific.

## Test Plan

`sdkWikiService.test.ts`:

- SAM golden path imports, lists, searches, reads, and plans pressure-load usage;
- unsupported intent returns `missingInformation` without invented API pages;
- compact registry system message does not preload page content.

`fileSystemToolHost.test.ts`:

- mock SAM model_write governance metadata writes `sdk_tool.requested` and `sdk_tool.executed`;
- mock SAM solve metadata dispatches without `SDK_TOOL_CONFIRMATION_REQUIRED`;
- audit events include SDK metadata and no confirmation artifacts.

Verification:

- `npm test -- electron/services/sdkWikiService.test.ts -t "Story 14.6"`
- `npm test -- electron/services/fileSystemToolHost.test.ts -t "Story 14.6"`
- targeted ESLint for touched files;
- TypeScript no-emit.

## Files to Modify

- `_bmad-output/implementation-artifacts/14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/sdk-adapter-contract-14-6.md`
- `_bmad-output/implementation-artifacts/code-review-14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/electron/services/sdkWikiService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`

## Traceability

- Story: `_bmad-output/implementation-artifacts/14-6-sam-golden-path-and-generic-sdk-adapter-contract.md`
- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Development plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
