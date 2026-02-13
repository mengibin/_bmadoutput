# Story BAI-5.1: Security Redaction and Secret Handling

Status: done

> Epic: `_bmad-output/epics-builder-ai.md`  
> Tech Spec: `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

## Story

As a **Security Engineer**,  
I want sensitive fields redacted in logs and responses,  
So that keys and secrets are never exposed.

## Acceptance Criteria

### AC-1: Structured secret redaction
- **Given** runtime logs and audit payloads
- **When** fields contain api key / token / authorization semantics
- **Then** values are masked as `***REDACTED***`.

### AC-2: Text-pattern redaction
- **Given** free-form text containing bearer tokens or `sk-*` style keys
- **When** text is persisted to log/audit pipelines
- **Then** sensitive substrings are redacted.

### AC-3: Regression tests
- **Given** nested payloads and mixed plain-text content
- **When** running unit tests
- **Then** redaction behavior is deterministic.

## Implementation Notes

- Added shared redaction utility:
  - `crewagent-builder-backend/app/services/redaction_service.py`
- Integrated redaction in:
  - `crewagent-builder-backend/app/services/ai_step_service.py`
  - `crewagent-builder-backend/app/services/tool_loop_service.py`
- Added unit tests:
  - `crewagent-builder-backend/tests/test_redaction_service.py`

## Tasks / Subtasks

- [x] Task 1: Implement key-based + pattern-based secret redaction
- [x] Task 2: Reuse redaction in AI logging paths
- [x] Task 3: Add regression tests for nested payload and plain-text tokens

## Validation

- `pytest -q tests/test_redaction_service.py`
