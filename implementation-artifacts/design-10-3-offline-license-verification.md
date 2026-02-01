# Design: Story 10.3 Offline License Verification (Public Key)

**Story:** `10-3-offline-license-verification.md`

---

## Design Goals
1. Verify license completely offline.
2. Enforce Machine ID + expiry.
3. Persist activation status.

---

## License Format
- Payload JSON -> base64
- Signature (RSA/Ed25519) -> base64
- License string: `payloadBase64.signatureBase64`

---

## Verification Flow
1. Parse license string.
2. Verify signature with embedded public key.
3. Decode payload; check `machineId` and `expiresAt`.
4. Persist activation status on success.

---

## Affected Files
- `crewagent-runtime/electron/services/licenseService.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx`

---

## Test Checklist
1. Valid license -> unlock.
2. Wrong machineId -> fail.
3. Expired -> fail.
4. Time rollback -> fail.
