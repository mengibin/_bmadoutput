# Tech Spec: Story 10.3 Offline License Verification (Public Key)

## 1. Overview
Implement offline license verification in Runtime using embedded public key. **Dev builds bypass license checks.**

## 2. Dev Build Bypass (Required)
- If `isDevBuild === true`, `verifyLicense()` always returns success and UI activation panel is hidden.

## 3. License Format

Canonical license string:
```
LICENSE = base64(payload) + "." + base64(signature)
```

Payload JSON schema:
```
LicensePayload {
  machineId: string
  issuedAt: number
  expiresAt: number   // -1 means never
  type: 'trial' | 'commercial'
  customerName?: string
}
```

## 4. Verification Flow
1. Split license into payload/signature.
2. Verify signature using embedded public key (RSA/Ed25519).
3. Decode payload JSON.
4. Validate `machineId` matches local.
5. Validate `expiresAt` (unless -1).
6. Check time rollback: `lastActiveAt <= now`.
7. Persist license state on success.

## 5. Storage
- Store verified license in Runtime settings or dedicated license store.

## 6. IPC
- `license:verify` -> `{ ok: boolean; error?: string }`

## 7. Files & Changes

| File | Change |
|:-----|:-------|
| `crewagent-runtime/electron/services/licenseService.ts` | **NEW**: verify + key handling |
| `crewagent-runtime/electron/stores/runtimeStore.ts` | **MOD**: persist license state |
| `crewagent-runtime/electron/main.ts` | **MOD**: IPC handler |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | **MOD**: activation input + feedback |

## 8. Verification
1. Valid key -> unlock.
2. Wrong machineId -> fail.
3. Expired -> fail.
4. Rollback -> fail.
