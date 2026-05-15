# Generic SDK Adapter Contract

Status: approved-for-story-14.6

## Purpose

This contract defines how an SDK such as SAM, Abaqus, Ansys, or OpenFOAM connects to CrewAgent Runtime without SDK-specific Runtime branches.

Runtime owns SDK knowledge import, retrieval, API planning, and audit. MCP/integration software owns execution safety.

## 1. SDK Wiki Pack Contract

An SDK Wiki Pack must be produced by the SDK Wiki builder-compatible rules:

- required files: `sdk-wiki.json`, `index/manifest.json`, `index/pages.json`, `index/symbols.json`, `index/terms.json`, `index/relations.json`, `wiki/index.md`, `wiki/log.md`;
- required directories: `raw/`, `wiki/`, `wiki/api/`, `wiki/workflows/`, `wiki/concepts/`, `wiki/relations/`, `index/`;
- `reports/` may exist but is not part of publish validation;
- page discovery includes structured `wiki/**/*.md`;
- page discovery excludes any `README.md`, `wiki/index.md`, and `wiki/log.md`;
- `indexHash` uses only `index/pages.json`, `index/symbols.json`, `index/terms.json`, and `index/relations.json`;
- `index/manifest.json`, `index/README.md`, and `reports/` are not included in `indexHash`.

Runtime imports the pack into `RuntimeStore/sdk-wikis/<sdkId>/<version>/` and exposes it through internal `sdk_wiki.*` tools.

## 2. SDK Knowledge Boundary

Runtime main LLM owns SDK reasoning:

- API selection;
- dependency reasoning;
- workflow planning;
- missing-knowledge reporting;
- source-referenced plan generation.

MCP/integration software must not hide another LLM loop for SDK semantic planning. MCP tools may return deterministic model state, tool results, errors, and tracebacks.

## 3. Execution Safety Boundary

MCP/integration software owns execution safety:

- path and workspace scope;
- destructive operation policy;
- long-running solve resource/license policy;
- domain-specific confirmation, if any;
- SDK traceback capture and deterministic result formatting.

CrewAgent Runtime treats an MCP-exposed SDK tool as trusted when existing effective Runtime tool policy allows the underlying tool/server. Runtime does not add SDK risk confirmation or one-shot confirmation tokens.

## 4. Governance Metadata

An adapter may map an underlying MCP/local tool call to:

```ts
type SdkToolRisk = 'read' | 'model_write' | 'file_write' | 'solve' | 'destructive'

interface SdkToolRiskEnvelope {
  sdkId: string
  toolName: string
  risk: SdkToolRisk
  purpose?: string
  targetPath?: string
  targetObject?: string
}
```

Recommended mapping examples:

| SDK Operation | risk | Recommended Metadata |
|:---|:---|:---|
| inspect current model | `read` | `targetObject` |
| create pressure load | `model_write` | `purpose`, `targetObject` |
| export report | `file_write` | `purpose`, `targetPath` |
| submit static solve | `solve` | `purpose`, `targetObject` |
| delete model entity | `destructive` | `purpose`, `targetObject` |

Missing metadata is an observability warning, not a Runtime execution blocker.

## 5. Runtime Audit Events

Runtime writes SDK governance events when metadata is available:

- `sdk_tool.requested`;
- `sdk_tool.metadata_warning`;
- `sdk_tool.executed`;
- `sdk_tool.failed`.

Common fields:

- `toolCallId`;
- `agentId`;
- `sdkId`;
- `sdkToolName`;
- `risk`;
- `purpose`;
- `targetPath`;
- `targetObject`;
- `governanceState`;
- `warnings`;
- `argsFingerprint`;
- `status`;
- `durationMs`;
- `errorCode` and `message` when failed.

## 6. SAM Golden Path Example

For a user intent such as `给当前板架模型施加向下均布压力载荷`:

1. Runtime imports the SAM SDK Wiki Pack.
2. Runtime uses `sdk_wiki.plan_api_usage`.
3. The plan must surface source-referenced API pages such as Pressure, Surface, and StaticStep.
4. The main LLM calls exposed MCP tools for deterministic execution.
5. The adapter maps model edits to `model_write` and solve submission to `solve`.
6. Runtime records governance audit and does not ask for a separate SDK confirmation.

## 7. Non-SAM Reuse Rule

Runtime code must depend only on:

- `sdkId`;
- SDK Wiki Pack structure;
- `sdk_wiki.*` tool contracts;
- `SdkToolRiskEnvelope`;
- existing effective tool policy.

It must not branch on `sam`, SAM package paths, SAM API names, or SAM MCP server internals.
