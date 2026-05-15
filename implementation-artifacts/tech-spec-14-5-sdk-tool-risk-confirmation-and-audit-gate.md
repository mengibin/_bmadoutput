# Tech-Spec: Story 14.5 Trusted SDK Tool Governance and Audit

**Created:** 2026-05-14
**Revised:** 2026-05-15
**Status:** Implemented; Approved
**Source Story:** `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`

## Overview

### Problem Statement

Story 14.1~14.4 gives Runtime a source-grounded SDK Wiki import/search/read/plan path. The execution boundary was initially designed as a Runtime risk confirmation gate, but the corrected architecture is different: MCP/integration software owns execution safety. If an integration exposes a tool, Runtime should let the Agent execute it under existing effective tool policy and should not add a second SDK confirmation layer.

The remaining Runtime responsibility is governance and audit:

- observe SDK risk metadata when an adapter can provide it;
- log missing/invalid metadata as a warning;
- keep requested/executed/failed SDK audit records;
- never use SDK risk metadata to enable a disabled underlying tool.

### Solution

Add a generic SDK governance layer:

- shared risk metadata contract for SDK execution adapters;
- audit-only governance evaluator for metadata normalization, warnings, and argument fingerprints;
- optional ToolHost boundary hook so SDK/MCP adapters can provide metadata before dispatch;
- audit records in existing `execution.jsonl` for requested/metadata-warning, executed, and failed states;
- removal of Runtime SDK confirmation token and `WIDGET_SUBMIT` interception.

## Scope

In scope:

- `shared/sdkToolRiskTypes.ts`;
- `electron/services/sdkToolRiskPolicyService.ts`;
- focused governance evaluator tests;
- optional SDK risk envelope resolver in `FileSystemToolHost`;
- ToolHost audit events for SDK governance observations;
- removal of chat/run SDK confirmation approval plumbing;
- focused ToolHost test proving metadata observation does not block execution.

Out of scope:

- actual SAM SDK execution adapter;
- external MCP server changes;
- SDK-specific command semantics;
- new UI components;
- SDK Wiki query tool changes;
- Runtime-side SDK execution safety confirmation.

## Contracts

### Risk Types

```ts
export type SdkToolRisk = 'read' | 'model_write' | 'file_write' | 'solve' | 'destructive'

export interface SdkToolRiskEnvelope {
  sdkId: string
  toolName: string
  risk: SdkToolRisk
  purpose?: string
  targetPath?: string
  targetObject?: string
}
```

### Governance Decision

```ts
export type SdkToolGovernanceState = 'observed' | 'metadata_warning'

export interface SdkToolGovernanceDecision {
  action: 'allow'
  sdkId: string
  toolName: string
  risk?: SdkToolRisk
  purpose?: string
  targetPath?: string
  targetObject?: string
  governanceState: SdkToolGovernanceState
  warnings: string[]
  argsFingerprint: string
}
```

There is intentionally no `confirmationRequest`, `confirmationToken`, `confirmationState`, `reject`, or `confirmation_required` decision in the revised contract.

## Governance Algorithm

1. Build stable argument fingerprint from original tool args:
   - recursively sort object keys;
   - preserve all user/tool arguments because Runtime no longer injects confirmation tokens;
   - SHA-256 hash the canonical representation.
2. Normalize `SdkToolRiskEnvelope`:
   - trim `sdkId`, `toolName`, `purpose`, `targetPath`, and `targetObject`;
   - accept only known risk values;
   - replace missing `sdkId` / `toolName` with `unknown` for audit.
3. Create warnings without blocking:
   - missing `sdkId`;
   - missing `toolName`;
   - invalid risk;
   - `model_write` missing `purpose`;
   - `file_write` missing `targetPath`;
   - resolver failure.
4. Return `action: "allow"` for all governance decisions.
5. ToolHost dispatches the underlying tool only if existing effective tool policy already allows that tool.

## ToolHost Integration

`FileSystemToolHostOptions` keeps an optional resolver:

```ts
resolveSdkToolRiskEnvelope?: (payload: {
  toolName: string
  args: Record<string, unknown>
  context: ExecuteContext
}) => SdkToolRiskEnvelope | null | undefined
```

If the resolver returns no envelope, existing tool behavior is unchanged.

If it returns an envelope:

1. ToolHost evaluates SDK governance metadata before dispatch.
2. Governance evaluation never returns a failed tool result.
3. ToolHost appends `sdk_tool.requested` or `sdk_tool.metadata_warning`.
4. ToolHost dispatches the underlying tool with the original args.
5. ToolHost appends `sdk_tool.executed` or `sdk_tool.failed`.

If the resolver throws:

1. ToolHost appends `sdk_tool.metadata_warning`.
2. ToolHost still dispatches the underlying tool if existing effective tool policy allows it.

## Audit Events

ToolHost writes structured SDK governance records:

- `sdk_tool.requested`
- `sdk_tool.metadata_warning`
- `sdk_tool.executed`
- `sdk_tool.failed`

Common fields:

- `toolCallId`
- `agentId`
- `sdkId`
- `sdkToolName`
- `risk`
- `purpose`
- `targetPath`
- `targetObject`
- `governanceState`
- `warnings`
- `argsFingerprint`
- `durationMs` when available
- `errorCode` and `message` when failed

Existing `tool.exec` audit remains unchanged and continues to record the final tool result.

## Files to Modify

- `_bmad-output/project-planning-artifacts/runtime-sdk-wiki-generic-requirements/03-tool-risk-confirmation-and-logging.md`
- `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/epics-runtime-sdk-wiki-and-tool-risk.md`
- `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
- `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- `_bmad-output/implementation-artifacts/tech-spec-14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-runtime/shared/sdkToolRiskTypes.ts`
- `crewagent-runtime/electron/services/sdkToolRiskPolicyService.ts`
- `crewagent-runtime/electron/services/sdkToolRiskPolicyService.test.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.ts`
- `crewagent-runtime/electron/services/fileSystemToolHost.test.ts`
- `crewagent-runtime/electron/main.ts`

## Test Plan

Governance evaluator tests:

- `read` metadata returns allow/observed;
- `model_write` without purpose returns allow/metadata_warning;
- `file_write`, `solve`, and `destructive` metadata do not create confirmation or reject decisions;
- invalid metadata returns allow/metadata_warning;
- fingerprints are stable for object key ordering.

ToolHost tests:

- optional risk resolver observes SDK metadata and the underlying tool executes immediately;
- audit log includes `sdk_tool.requested` or `sdk_tool.metadata_warning`;
- audit log includes `sdk_tool.executed` for success and `sdk_tool.failed` for tool failure;
- no `SDK_TOOL_CONFIRMATION_REQUIRED`, `confirmationToken`, or SDK-risk `WIDGET_SUBMIT` approval path remains in runtime code.

Verification:

- focused governance evaluator tests;
- focused ToolHost SDK governance tests;
- targeted ESLint;
- TypeScript no-emit;
- build verification when runtime checks remain green.

## Traceability

- Story: `_bmad-output/implementation-artifacts/14-5-sdk-tool-risk-confirmation-and-audit-gate.md`
- PRD: `_bmad-output/prd-runtime-sdk-wiki-and-tool-risk.md`
- Architecture: `_bmad-output/architecture/runtime-sdk-wiki-and-tool-risk-architecture.md`
- Development plan: `_bmad-output/implementation-artifacts/epic-14-runtime-sdk-wiki-development-plan.md`
