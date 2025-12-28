# Story 2.3: Implement JWT Auth in Backend

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **Developer**,  
I want to implement JWT authentication in the FastAPI backend,  
so that API endpoints are protected.

## Acceptance Criteria

1. **Given** a request comes with a JWT token in the Authorization header  
   **When** the backend validates the token signature and claims  
   **Then** authenticated requests proceed to the handler  
   **And** unauthenticated requests receive 401 Unauthorized

2. **Given** a request has a missing/expired/invalid JWT token  
   **When** the backend validates the token signature and claims  
   **Then** the request is rejected with 401 Unauthorized  
   **And** the response contains a structured JSON error body (code + message)  
   **And** the protected handler is not executed

## Design

### Summary

- 新增 `get_current_user` 依赖：从 `Authorization: Bearer <token>` 解析 token，校验签名与 claims（iss/aud/exp/iat/sub + leeway），并根据 `sub` 加载用户。
- 将 `GET /users/me` 改为受保护路由：成功返回当前用户信息；失败返回统一 `{ data: null, error: { code, message } }`。

### API / Contracts

- Header：
  - `Authorization: Bearer <accessToken>`
- Backend env：
  - `JWT_SECRET` / `JWT_ALGORITHM` / `JWT_EXPIRES_IN_SECONDS`
  - `JWT_ISSUER` / `JWT_AUDIENCE` / `JWT_LEEWAY_IN_SECONDS`
- Endpoints：
  - `GET /users/me`
    - Success (200): `{ "data": { "id": number, "email": string, "username": string }, "error": null }`
    - Failure (401): `{ "data": null, "error": { "code": string, "message": string } }`
      - `AUTH_MISSING_TOKEN`：缺少 token
      - `AUTH_TOKEN_EXPIRED`：token 过期
      - `AUTH_INVALID_TOKEN`：token 无效

### Errors / Edge Cases

- 缺少 `Authorization` / 非 Bearer scheme：401（`AUTH_MISSING_TOKEN`）
- token 过期：401（`AUTH_TOKEN_EXPIRED`）
- token 无效（签名/issuer/audience/格式/claims 不完整）：401（`AUTH_INVALID_TOKEN`）
- `sub` 对应用户不存在：401（`AUTH_INVALID_TOKEN`）

### Test Plan

- Backend：
  - `GET /users/me` 无 token → 401 + 结构化错误
  - token 过期 → 401 + 结构化错误
  - token 无效 → 401 + 结构化错误
  - 合法 token → 200 + 返回用户
  - `./.venv/bin/ruff check .`、`./.venv/bin/pytest -q`

## Tasks / Subtasks

- [x] 1) 实现 JWT 校验依赖（AC: 1,2）
  - [x] `get_current_user`：解析 Authorization header、校验 token、加载用户
  - [x] 校验 issuer/audience/leeway，缺失/过期/无效 token 返回 401（结构化错误）

- [x] 2) 受保护路由落地（AC: 1,2）
  - [x] `GET /users/me` 使用 `Depends(get_current_user)`，成功返回当前用户

- [x] 3) 测试覆盖（AC: 1,2）
  - [x] 缺失 token / 无效 token / 过期 token / 成功请求

## Dev Notes

- 依赖实现应复用 `app/utils/security.py:decode_jwt()`（PyJWT），并使用 `Settings` 中的 `jwt_*` 配置。
- 错误返回必须走全局异常处理器格式（`{ data, error }`），不要暴露内部堆栈。

### Project Structure Notes

- `crewagent-builder-backend/app/dependencies/auth.py`（新增）
- `crewagent-builder-backend/app/routers/users.py`
- `crewagent-builder-backend/app/utils/security.py`
- `crewagent-builder-backend/tests/`

### References

- `_bmad-output/implementation-artifacts/2-1-user-registration-on-builder.md`
- `_bmad-output/implementation-artifacts/2-2-user-login-on-builder.md`

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- `./.venv/bin/ruff check .`
- `./.venv/bin/pytest -q`

### Completion Notes List

- 新增 JWT 认证依赖：从 `Authorization: Bearer <token>` 校验签名与 claims（iss/aud/exp/iat/sub + leeway）。
- `GET /users/me` 改为受保护路由：认证通过返回当前用户；否则 401 + 结构化错误体 `{ data: null, error: { code, message } }`。
- 增加测试覆盖：缺失/无效/过期 token 以及合法 token 的行为。

### File List

- `_bmad-output/implementation-artifacts/2-3-implement-jwt-auth-in-backend.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/dependencies/auth.py`
- `crewagent-builder-backend/app/routers/users.py`
- `crewagent-builder-backend/app/utils/security.py`
- `crewagent-builder-backend/tests/test_users_me.py`

## Senior Developer Review (AI)

### Review Outcome

- Date: 2025-12-23
- Outcome: Approved
- Issues: 0
- Acceptance Criteria: PASS

### Notes (Non-blocking)

- 建议后续（可选）：对 401 响应补齐 `WWW-Authenticate: Bearer` header（如果后续接入第三方客户端/SDK 会更标准）。
