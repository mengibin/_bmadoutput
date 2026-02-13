# Story BAI-5.3: Metrics and Alerting Baseline

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As an **Operator**,  
I want metrics for latency/error/conflict/apply conversion,  
So that rollout health is visible.

## Acceptance Criteria

### AC-1: Core metric dimensions
- **Given** workbench traffic
- **When** requests complete
- **Then** metrics include message/apply totals, success/failure, and error-code counts.

### AC-2: Latency + mismatch + conversion
- **Given** message/apply pipelines
- **When** metric snapshots are computed
- **Then** `p95 latency`, `revision mismatch warnings`, and `apply conversion` are present.

### AC-3: Baseline alert signals
- **Given** configurable thresholds
- **When** metric snapshot exceeds/breaches threshold
- **Then** structured alerts are emitted in response payload.

## Implementation Notes

- Added in-memory metrics aggregator:
  - `crewagent-builder-backend/app/services/ai_metrics_service.py`
- Added metrics endpoint:
  - `GET /packages/{project_id}/ai/metrics`
- Added rollout policy snapshot to metrics response:
  - `summarize_rollout_settings(...)`
- Configurable thresholds in settings:
  - `AI_METRICS_ALERT_P95_MS`
  - `AI_METRICS_ALERT_FAILURE_RATE`
  - `AI_METRICS_ALERT_APPLY_CONVERSION_RATE`

## Tasks / Subtasks

- [x] Task 1: Implement metrics registry and p95 calculator
- [x] Task 2: Emit metrics on message/apply success/failure/blocked paths
- [x] Task 3: Add project-scoped metrics API
- [x] Task 4: Add alert threshold evaluation in snapshot output

## Validation

- `pytest -q tests/test_ai_observability_rollout.py::test_ai_metrics_endpoint_exposes_apply_and_alert_baseline`
