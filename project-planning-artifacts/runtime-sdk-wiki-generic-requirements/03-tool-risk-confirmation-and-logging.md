# Requirement: Trusted MCP Tool Governance And Logging

Revision: 2026-05-15

## Goal

SDK operations can modify CAE models, write files, or start long-running solves.
Execution safety is owned by the integration MCP software that exposes the tool.
CrewAgent Runtime treats an exposed MCP tool as safe to execute under the
existing effective tool policy, and provides governance metadata plus audit
behavior instead of a user confirmation gate.

This requirement replaces the earlier Runtime-side confirmation-gate model.
Runtime must not require user confirmation for `file_write`, `solve`, or
`destructive` solely because of SDK risk metadata. If a tool must be blocked,
scoped, sandboxed, path-guarded, licensed, or confirmed, that must be enforced by
the MCP/integration layer before the tool is exposed or inside the tool itself.

## Risk Levels

```text
read
model_write
file_write
solve
destructive
```

## Required Behavior

- `read`: execute without Runtime SDK confirmation.
- `model_write`: execute autonomously; Runtime should log purpose when provided
  and warn when missing.
- `file_write`: execute autonomously; Runtime should log target path when
  provided and rely on MCP/path policy for actual path safety.
- `solve`: execute autonomously; Runtime should log risk and duration/status.
- `destructive`: execute autonomously if the MCP exposes the tool and effective
  Runtime tool policy allows it; destructive safety is an MCP responsibility.

Runtime may still hide or disable an entire MCP server/tool through existing
effective tool policy. SDK risk metadata cannot grant a tool that the effective
policy disabled, and it should not block a tool that policy allowed.

## Logging

Each SDK tool call log should include:

- tool name
- purpose
- risk
- governance state or metadata warnings
- execution status
- duration
- error code and traceback summary when failed

## Acceptance Criteria

- SDK `solve` and `destructive` tool calls do not trigger Runtime user
  confirmation solely due to SDK risk metadata.
- Runtime logs requested/executed/failed SDK tool audit events when SDK metadata
  is available.
- Missing or invalid SDK governance metadata is logged as a warning, not used as
  a Runtime execution blocker.
- Traceback from SAM is retained for debugging.
