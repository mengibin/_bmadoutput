# Story BAI-5.2: Audit Event Pipeline

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As an **Operator**,  
I want key AI lifecycle events recorded,  
So that every apply operation is traceable.

## Acceptance Criteria

### AC-1: Persistent audit model
- **Given** AI workbench lifecycle events
- **When** events are emitted
- **Then** records are stored in a dedicated audit table.

### AC-2: Apply traceability linkage
- **Given** successful/failed apply requests
- **When** querying audit events
- **Then** records include `sessionId`, `changeSetId`, `userId`, and changed object summary.

### AC-3: Query API
- **Given** operator requests by project/session/event type
- **When** calling audit endpoint
- **Then** events are returned in reverse chronological order.

## Implementation Notes

- Added DB model + migration:
  - `crewagent-builder-backend/app/models/ai_audit_event.py`
  - `crewagent-builder-backend/migrations/versions/5d6e7f8a9b0c_add_ai_audit_events_table.py`
- Added audit service:
  - `crewagent-builder-backend/app/services/ai_audit_service.py`
- Added audit list endpoint:
  - `GET /packages/{project_id}/ai/audit/events`
- Wired audit emission in:
  - session create
  - message send (sync/stream)
  - stage/validate
  - apply success/failure/blocked
  - cancel
- Apply summary now includes changed object list:
  - `crewagent-builder-backend/app/services/change_set_apply_service.py`

## Tasks / Subtasks

- [x] Task 1: Add audit table and ORM model
- [x] Task 2: Emit lifecycle audit events in package AI router
- [x] Task 3: Add operator query API and schema contract
- [x] Task 4: Ensure apply events include changed object trace

## Validation

- `pytest -q tests/test_ai_observability_rollout.py::test_ai_apply_audit_contains_changed_objects`
