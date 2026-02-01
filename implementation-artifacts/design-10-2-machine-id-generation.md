# Design: Story 10.2 Machine ID Generation + UI Display

**Story:** `10-2-machine-id-generation.md`

---

## Design Goals
1. Provide stable Machine ID based on hardware fingerprint.
2. Display and copy in activation UI.

---

## UX / UI
- Activation modal shows:
  - Machine ID string (monospace)
  - Copy button (with "Copied" toast)
  - Helper text: "Send this ID to admin to obtain a license key"
- Entry: click the **trial banner / version text under Settings** in the sidebar to open the modal.

---

## Runtime Logic
- Use `node-machine-id` to generate Machine ID.
- Cache the Machine ID to avoid recalculating.

---

## Affected Files
- `crewagent-runtime/electron/services/licenseService.ts` (getMachineId)
- `crewagent-runtime/src/components/activation/ActivationModal.tsx`
- `crewagent-runtime/src/components/layout/AppShell.tsx`
- `crewagent-runtime/src/components/layout/Sidebar.tsx`

---

## Test Checklist
1. Machine ID visible on activation screen.
2. Copy button copies full ID.
3. Error state shown if fingerprint fails.
