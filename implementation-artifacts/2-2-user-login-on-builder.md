# Story 2.2: User Login on Builder

Status: done

<!-- Note: Validation is optional. Run validate-create-story for quality check before design-story/dev-story. -->

## Story

As a **User**,  
I want to login to the Builder platform,  
so that I can access my saved Workflows.

## Acceptance Criteria

1. **Given** I have a registered account  
   **When** I enter my email and password on the login page  
   **Then** I receive a JWT token and am authenticated  
   **And** I can access protected routes (Dashboard, Editor)

## Design

### Summary

- 在 `crewagent-builder-frontend/src/app/login/page.tsx` 补齐“登录”Tab：email/password 表单，调用后端 `POST /auth/login`。
- 后端新增 `POST /auth/login`：按 email 查询用户、校验密码哈希、签发 JWT（沿用 Story 2.1 的 issuer/audience/leeway 配置）。
- 前端保存 `accessToken`（MVP：localStorage）并跳转 `/dashboard`。
- 前端实现最小“受保护路由”：`/dashboard` 与 `/editor` 页面在无 token 时重定向到 `/login`（安全增强在 Story 2.3）。

### UX / UI

- 页面：`/login`
  - Tab：`登录` / `注册`
  - 登录表单字段：
    - Email（必填；基本格式校验）
    - Password（必填；最少 8 位；登录失败不区分具体原因）
  - 交互：
    - 提交中：按钮 disabled + loading 文案
    - 成功：保存 token 后直接跳转 `/dashboard`
    - 失败：顶部显示错误（不暴露内部堆栈；统一“邮箱或密码错误”）
  - 环境未配置（缺失 `NEXT_PUBLIC_API_BASE_URL`）：显示明确提示，并禁用提交

### API / Contracts

- Frontend env：
  - `NEXT_PUBLIC_API_BASE_URL`（例如：`http://localhost:8000`）
- Backend env（示例）：
  - `DATABASE_URL=postgresql+psycopg://crewagent:crewagent@localhost:5432/crewagent`
  - `JWT_SECRET=...`（HS256；非开发环境必须足够强）
  - `JWT_ALGORITHM=HS256`
  - `JWT_EXPIRES_IN_SECONDS=86400`
  - `JWT_ISSUER=crewagent-builder-backend`
  - `JWT_AUDIENCE=crewagent-builder-frontend`
  - `JWT_LEEWAY_IN_SECONDS=0`
- Backend API：
  - `POST /auth/login`
    - Request: `{ "email": string, "password": string }`
    - Success (200): `{ "data": { "accessToken": string, "user": { "id": number, "email": string, "username": string } }, "error": null }`
    - Failure:
      - 400：字段校验失败（email 格式、密码长度）
      - 401：邮箱或密码错误（不暴露具体是 email 不存在还是密码错误）
      - 500：内部错误（不暴露堆栈）

### Data / Storage

- 无新增表；复用 `users` 表（email/password_hash）。
- Token（MVP）：前端 localStorage 保存 `accessToken`；后续请求使用 `Authorization: Bearer <token>`。

### Errors / Edge Cases

- `NEXT_PUBLIC_API_BASE_URL` 缺失：页面提示“环境未配置”，避免空白失败。
- 用户不存在/密码错误：统一提示“邮箱或密码错误”，返回 401。
- 后端不可达 / 网络错误：提示“服务暂不可用，请稍后重试”。

### Test Plan

- 手工冒烟：
  - 成功登录 → 自动跳转 `/dashboard`
  - 错误密码 → 看到明确错误提示
  - 无 token 访问 `/dashboard`/`/editor` → 自动回到 `/login`
- 工程校验：
  - Backend：`pytest` 通过（覆盖成功登录/错误密码/不存在用户/校验错误）
  - Frontend：`npm run lint`、`npm run build` 通过

## Tasks / Subtasks

- [x] 1) Backend：登录 API（AC: 1）
  - [x] 新增 `POST /auth/login`（email/password）
  - [x] 校验密码哈希（bcrypt via passlib）
  - [x] 成功签发 JWT 并返回 `{ data, error }`
  - [x] pytest 覆盖：成功登录 / 错误密码 / 不存在用户 / 校验错误

- [x] 2) Frontend：登录 UI + 调用后端（AC: 1）
  - [x] `/login` 登录 Tab：email/password 表单（required + basic 校验）
  - [x] 调用 `POST ${NEXT_PUBLIC_API_BASE_URL}/auth/login`，成功后保存 token 并跳转 `/dashboard`
  - [x] 提交态与错误态：loading、提交失败提示、字段错误提示

- [x] 3) Frontend：最小“受保护路由”（AC: 1）
  - [x] `/dashboard` 无 token 时重定向 `/login`
  - [x] 新增 `/editor` 占位页，并做同样重定向
  - [x] （可选）提供退出登录按钮：清理 token 并回到 `/login`

- [ ] 4) 本地验证（AC: 1）
  - [x] Backend：`./.venv/bin/ruff check .`、`./.venv/bin/pytest -q`
  - [x] Frontend：`npm run lint`、`npm run build`
  - [x] `npm run dev` 浏览器手工走通登录流程（成功与失败各至少 1 次）

## Dev Notes

### 关键约束 / 决策

- Story 2.2 只做登录与前端最小路由守卫；后端 JWT 校验与 API 保护在 Story 2.3 完成。
- 错误信息统一，避免泄露账号是否存在。

### Project Structure Notes

- Backend：
  - `crewagent-builder-backend/app/routers/auth.py`
  - `crewagent-builder-backend/app/services/auth_service.py`
  - `crewagent-builder-backend/app/schemas/user.py`
  - `crewagent-builder-backend/tests/`
- Frontend：
  - `crewagent-builder-frontend/src/app/login/page.tsx`
  - `crewagent-builder-frontend/src/app/dashboard/page.tsx`
  - `crewagent-builder-frontend/src/app/editor/page.tsx`（新增）
  - `crewagent-builder-frontend/src/lib/use-require-auth.ts`（新增）
  - `crewagent-builder-frontend/src/lib/auth.ts`

### References

- `_bmad-output/epics.md`（Epic 2 / Story 2.2）
- `_bmad-output/implementation-artifacts/2-1-user-registration-on-builder.md`（已有 token 签发与 env 约定）

## Dev Agent Record

### Agent Model Used

GPT-5.2 (Codex CLI)

### Debug Log References

- Backend:
  - `./.venv/bin/ruff check .`
  - `./.venv/bin/pytest -q`
- Frontend:
  - `npm run lint`
  - `npm run build`

### Completion Notes List

- Backend：实现 `POST /auth/login`，使用 email 查询用户并校验密码哈希，成功返回 `accessToken` 与 user。
- Frontend：实现登录表单 + 错误态/提交态；登录成功保存 token 并跳转 `/dashboard`。
- Frontend：为 `/dashboard` 与 `/editor` 增加最小路由守卫（无 token → `/login`），并提供退出登录。
- Frontend：修复 Dashboard/Editor 路由守卫在刷新时的 hydration mismatch（通过 `useSyncExternalStore` 读取 token，并用自定义事件同步同页 token 变更）。
- 本地冒烟：登录成功跳转 `/dashboard`；无 token 访问 `/dashboard`/`/editor` 会重定向回 `/login`。

### File List

- `_bmad-output/implementation-artifacts/2-2-user-login-on-builder.md`
- `_bmad-output/implementation-artifacts/sprint-status.yaml`
- `crewagent-builder-backend/app/routers/auth.py`
- `crewagent-builder-backend/app/services/auth_service.py`
- `crewagent-builder-backend/app/schemas/user.py`
- `crewagent-builder-backend/tests/test_auth.py`
- `crewagent-builder-backend/tests/test_login.py`
- `crewagent-builder-frontend/src/app/login/page.tsx`
- `crewagent-builder-frontend/src/app/dashboard/page.tsx`
- `crewagent-builder-frontend/src/app/editor/page.tsx`
- `crewagent-builder-frontend/src/lib/use-require-auth.ts`
- `crewagent-builder-frontend/src/lib/auth.ts`
