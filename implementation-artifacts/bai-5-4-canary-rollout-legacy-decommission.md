# Story BAI-5.4: Canary Rollout + Legacy Decommission Plan

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Release Manager**,  
I want staged rollout with safe feature-flag rollback (without reviving legacy optimizer path),  
So that migration risk is controlled.

## Acceptance Criteria

### AC-1: Canary admission gate
- **Given** canary mode is enabled
- **When** non-canary users hit AI workbench endpoints
- **Then** requests are blocked with deterministic error code and audit trail.

### AC-2: Guarded downgrade modes
- **Given** rollout risk or incident
- **When** `disable apply` or `read-only` flags are enabled
- **Then** apply/mutation requests are safely blocked without touching business objects.

### AC-3: Legacy path remains decommissioned
- **Given** guarded mode
- **When** AI capability is downgraded
- **Then** fallback is policy gating only; legacy `step-draft` path is not revived.

## Implementation Notes

- Added rollout policy service:
  - `crewagent-builder-backend/app/services/ai_rollout_service.py`
- Added settings flags:
  - `AI_WORKBENCH_CANARY_ENABLED`
  - `AI_WORKBENCH_CANARY_PERCENT`
  - `AI_WORKBENCH_CANARY_USER_IDS`
  - `AI_WORKBENCH_READ_ONLY`
  - `AI_WORKBENCH_DISABLE_APPLY`
- Added endpoint-level guard hooks in:
  - session create/message
  - stage/validate
  - apply endpoints (session + change apply)
- Guard blocks are audited and metered for rollout visibility.

## Tasks / Subtasks

- [x] Task 1: Add rollout flag schema and validation
- [x] Task 2: Implement canary user admission logic
- [x] Task 3: Add apply/read-only guard rails with structured errors
- [x] Task 4: Add tests for canary/disable-apply gate behavior

## Validation

- `pytest -q tests/test_ai_observability_rollout.py::test_ai_workbench_canary_gate_blocks_non_canary_user`
- `pytest -q tests/test_ai_observability_rollout.py::test_ai_apply_disable_flag_blocks_manual_apply`
