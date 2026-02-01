# Design: Story 10.4 Builder Offline License Generator

**Story:** `10-4-builder-license-generator.md`

---

## Design Goals
1. Provide simple license generator in Builder.
2. Sign payload with private key.

---

## UX / UI
- Form fields:
  - Machine ID
  - Customer name (optional)
  - Expiry (date or "permanent")
- Generate button
- Output field + Copy

---

## Backend Logic
- Build payload JSON
- Sign with private key
- Return license string

---

## Affected Files
- `crewagent-builder-frontend/...`
- `crewagent-builder-backend/...`

---

## Test Checklist
1. Generate key -> non-empty output.
2. Key verifies in Runtime.
