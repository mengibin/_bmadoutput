# Tech Spec: Story 10.2 Machine ID Generation + UI Display

## 1. Overview
Provide a Machine ID derived from hardware fingerprint and expose it in the activation UI with copy support. **Dev builds skip license UI.**

## 2. Dev Build Bypass (Required)
- Dev builds **do not enforce** license gating.
- Activation UI **can be shown for testing**; Machine ID retrieval still works in dev.

## 3. Machine ID Service

Create `licenseService` in main process:
- `getMachineId(): Promise<string>`
- Uses `node-machine-id` (or equivalent) to return a stable **hashed** id (`original: false`).
- Cache result in memory for fast access.

Expose via IPC:
- `ipcMain.handle('license:getMachineId', ...)`

## 4. UI

Activation modal shows:
- `Machine ID` (monospace)
- Copy button (toast: "Copied"; fallback to IPC clipboard if browser clipboard unavailable)
- Help text for admin
- Entry: click the **trial banner / version text under Settings** in the sidebar to open the modal.

## 5. Files & Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/licenseService.ts` | **NEW**: `getMachineId()` |
| `crewagent-runtime/electron/main.ts` | **MOD**: IPC handler `license:getMachineId` |
| `crewagent-runtime/src/components/activation/ActivationModal.tsx` | **NEW**: activation modal UI with copy |
| `crewagent-runtime/src/components/layout/AppShell.tsx` | **MOD**: activation modal host + trigger |
| `crewagent-runtime/src/components/layout/Sidebar.tsx` | **MOD**: activation entry under Settings |
| `crewagent-runtime/src/stores/appStore.ts` | **MOD**: store machineId state |

## 6. Verification

1. Dev build -> activation UI hidden.
2. Machine ID shown in activation panel.
3. Copy button works.
4. Failure -> error message with retry.
