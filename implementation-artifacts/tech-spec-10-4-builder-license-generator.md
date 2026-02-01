# Tech Spec: Story 10.4 Builder Offline License Generator

## 1. Overview
Provide a Builder tool to generate offline license keys from Machine ID. **Dev builds skip license requirements.**

## 2. License Payload + Signature
- Use same payload schema as Runtime verification (Story 10.3).
- Generate signature with private key stored in Builder backend.

## 3. API

### Backend endpoint
`POST /license/generate`
```
{
  machineId: string
  expiresAt: number | -1
  customerName?: string
  type: 'commercial'
}
```

Response:
```
{ licenseKey: string }
```

## 4. UI
- Simple form with Machine ID, expiry date, customer name.
- Generate button calls backend.
- Copy output button.

## 5. Files & Changes

| File | Change |
|:-----|:-------|
| `crewagent-builder-backend/...` | **NEW**: license generation endpoint + key storage |
| `crewagent-builder-frontend/...` | **NEW/MOD**: License Generator UI |

## 6. Verification
1. Generated key verifies in Runtime.
2. Invalid machineId rejected.
