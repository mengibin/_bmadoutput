# Story 10.2: Machine ID Generation + UI Display

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. Design complete. -->

## Story

As a **Consumer**,
I want Runtime to display a hardware-bound Machine ID that I can copy,
So that I can send it to an admin to obtain a license key.

## Acceptance Criteria

### 1. Machine ID Generation
- **Given** I open the activation screen
- **When** the screen loads
- **Then** Runtime generates a Machine ID from hardware fingerprint

### 2. UI Display + Copy
- **Given** the Machine ID is available
- **When** it is shown in the UI
- **Then** I can copy it with one click

### 3. Error Handling
- **Given** hardware fingerprinting fails
- **Then** a readable error is shown and a retry option is available

## Design

### UX / UI
- Activation screen shows:
  - Machine ID string
  - Copy button
  - Help text for admin

## Technical Components / Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/licenseService.ts` | **NEW**: `getMachineId()` |
| `crewagent-runtime/src/components/activation/ActivationModal.tsx` | **NEW**: activation modal UI |
| `crewagent-runtime/src/components/layout/Sidebar.tsx` | **MOD**: activation entry under Settings |

## Dependencies
- Story 10.1 (Trial Gate)

## Verification Plan
1. Open activation screen -> Machine ID shown
2. Copy button copies full value
3. Simulate failure -> error message

> 设计文档：`_bmad-output/implementation-artifacts/design-10-2-machine-id-generation.md`  
> 技术规格：`_bmad-output/implementation-artifacts/tech-spec-10-2-machine-id-generation.md`
