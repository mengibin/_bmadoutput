# Validation Report: Story 4.8 - Create Run Folder (RuntimeStore) & Initialize State

**Date**: 2026-01-03  
**Story**: 4.8 - Create Run Folder (RuntimeStore) & Initialize State  
**Status**: NEEDS REVISION (update before design-story)

---

## Summary

Story 4.8 captures the core run folder creation and state initialization flow, but it misses several requirements already defined in the Epic and architecture. The biggest gaps are the missing `@state/logs/execution.jsonl` creation, unclear run-scoped `@state` access requirements, and an API mismatch (`projectId` vs `projectRoot`) relative to existing RuntimeStore usage. These should be clarified before moving to `design-story 4.8`.

References:
- `_bmad-output/epics.md` (Story 4.8 acceptance criteria)
- `_bmad-output/architecture/runtime-architecture.md`
- `crewagent-runtime/docs/runtime-spec.md`
- `crewagent-runtime/docs/agent-menu-command-contract.md`

---

## Validation Checklist

### Story Structure
- [x] Title and Status fields present
- [x] User story is clear (role, action, benefit)
- [x] Acceptance criteria cover run folder creation and state init
- [x] Tasks and verification steps are listed

### Technical Completeness
- [ ] Run-scoped log directory and `@state/logs/execution.jsonl` creation required
- [ ] Explicit requirement that `@state` file ops require runId context and reject missing context
- [ ] `createRun` inputs aligned with runtime APIs (`projectRoot` vs `projectId`)
- [ ] `runsIndex` path/schema and fields are explicitly defined
- [ ] Frontmatter init preserves required fields and schema compatibility

### Alignment with Project Docs
- [ ] Epic Story 4.8 criteria fully reflected (logs, artifacts dir, runId context)
- [x] Storage layout matches RuntimeStore runs/<runId>/state structure
- [x] RunManager role aligns with runtime spec (create run + init state)

---

## Findings

### Critical Gaps (Must Fix)
1) **Missing log file creation**: The Epic requires `@state/logs/execution.jsonl` to exist per run, but Story 4.8 does not mention creating `logs/` or the JSONL file. This is required for Story 4.6 auditing and Story 5.2 UI log persistence.
2) **Run-scoped @state requirement missing**: The Epic requires `@state` operations to be bound to a runId; the story only lists mounts but does not require rejecting calls without run context.
3) **API mismatch for project identity**: The story uses `projectId`, but existing flow and RuntimeStore APIs operate on `projectRoot` and derive `projectId`. The story must define how projectId is obtained or switch the API to `projectRoot`.

### Significant Gaps (Should Fix)
1) **`runsIndex` schema is vague**: The story says "runsIndex.json or similar" and omits `packageId`. The schema should be explicit and align with existing `ConversationRunMetadata` fields (`phase` vs `status`).
2) **Frontmatter initialization is under-specified**: It should explicitly preserve required fields (`schemaVersion`, `workflowType`, `variables`, `decisionLog`, `artifacts`) and set `updatedAt` when writing.
3) **Validation coverage incomplete**: The story validates `projectId/packageId/workflowRef`, but does not require validating `activeAgentId` existence or that `workflow.md` exists in the package.

---

## Recommendations

1) Add acceptance criteria for `@state/logs/execution.jsonl` creation and `@state` runId enforcement (reject missing run context).
2) Clarify `RunManager.createRun` inputs: prefer `projectRoot` and derive `projectId` (consistent with RuntimeStore), or explicitly define a projectId->root lookup.
3) Define a concrete `runsIndex` schema and path, including `packageId` and a single source of truth for run phase/status.
4) Require frontmatter validation against `workflow-frontmatter.schema.json` after updates and preserve existing fields from package `workflow.md`.

---

## Verdict

NEEDS REVISION: Update Story 4.8 to include the missing run log, run-scoped `@state` rules, and clarify the API/data model before proceeding to `design-story 4.8`.

