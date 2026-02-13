# Tech Spec: Epic BAI-1 - User-Level LLM Profile Foundation

**Epic**: `BAI-1`（`_bmad-output/epics-builder-ai.md`）  
**Status**: Done（Aligned to session/tool-loop architecture）  
**Scope**: BAI-1.1 ~ BAI-1.5  
**Related Docs**:
- `_bmad-output/prd-builder-AI.md`
- `_bmad-output/architecture/builder-ai-llm-interaction-architecture.md`
- `_bmad-output/tech-spec/builder-ai-implementation-spec.md`

---

## 1. 目标与边界

### 1.1 目标

完成用户级 LLM 配置基础设施，替代当前全局 `settings.ai_*` 直读模式，满足：

- FR15：按用户读取 provider/baseUrl/model/apiKey/timeout
- FR16：可用性状态明确（可用/禁用/未配置/失败原因）
- FR17：支持连接测试 API，供 Profile 页面使用
- NFR5b：密钥隔离，不跨用户泄露
- NFR12：配置流程可在 1 分钟内完成

### 1.2 非目标（本 Epic 不做）

- 不实现完整 AI Workbench 页面（BAI-4）
- 不实现 changeSet apply（BAI-3）
- 不实现复杂组织策略 UI（仅保留后端 hook）

---

## 2. 实现现状（已完成对齐）

### 2.1 当前实现现状（代码）

- 用户级 profile 数据模型已落地：`app/models/user_llm_profile.py`
- profile API 已落地：`GET/PUT/POST test /users/me/llm-profile*`（`app/routers/llm_profile.py`）
- API key 已使用加密存储与脱敏返回（`app/utils/secret_crypto.py` + service 封装）
- 组织策略 hook 已统一落地：`app/services/org_policy_service.py`
- AI 网关已接入用户级 profile 解析：`resolve_effective_llm_config(...)` 于 session/message 链路生效

### 2.2 本次对齐结果（Architecture Sync）

1. provider 枚举统一为 `disabled|openai-compatible`，不再接受 `mock`
2. 组织策略枚举统一为 `allow-all|disable-all|allow-openai-compatible-only`
3. profile 生效主链路统一到 `/packages/{projectId}/ai/sessions/{sessionId}/messages`
4. legacy `step-draft` 不再作为 BAI-1 的验收路径

---

## 3. 数据库 Migration 设计（BAI-1.1）

## 3.1 新表 `user_llm_profiles`

建议字段：

- `id` INTEGER PK
- `user_id` INTEGER NOT NULL UNIQUE FK -> `users.id` (`ON DELETE CASCADE`)
- `provider` VARCHAR(32) NOT NULL DEFAULT `'disabled'`
- `base_url` VARCHAR(500) NULL
- `model` VARCHAR(200) NULL
- `api_key_encrypted` TEXT NULL
- `timeout_seconds` INTEGER NOT NULL DEFAULT `60`
- `context_window` INTEGER NULL
- `health_status` VARCHAR(32) NOT NULL DEFAULT `'unknown'`
- `last_tested_at` DATETIME(timezone=True) NULL
- `created_at` DATETIME(timezone=True) NOT NULL DEFAULT `CURRENT_TIMESTAMP`
- `updated_at` DATETIME(timezone=True) NOT NULL DEFAULT `CURRENT_TIMESTAMP`

枚举约束：

- `provider in ('disabled', 'openai-compatible')`
- `health_status in ('unknown', 'ok', 'failed')`

索引：

- `ux_user_llm_profiles_user_id`（唯一）
- `ix_user_llm_profiles_provider`

## 3.2 Alembic 实际落地

已落地 revision：

- `6f9f0b1f2a3c_add_user_llm_profiles_table.py`（创建 `user_llm_profiles`）
- `8c4d9e7f1a2b_remove_mock_llm_provider_values.py`（历史 provider 归一化）

## 3.3 模型与配置任务

新增模型文件：

- `crewagent-builder-backend/app/models/user_llm_profile.py`

并在 `app/main.py` 的 model import 区域确保模型被加载（沿用当前 pattern）。

新增配置项（用于密钥加解密与组织策略）：

- `LLM_PROFILE_ENCRYPTION_KEY`（必填于生产）
- `LLM_ORG_POLICY_MODE`（`allow-all|disable-all|allow-openai-compatible-only`）

---

## 4. API Contract（BAI-1.2 / BAI-1.3）

统一返回格式维持现有约定：

- 成功：`{"data": ..., "error": null}`
- 失败：`{"data": null, "error": {"code": "...", "message": "...", "details": {...}}}`

所有接口均需 `Authorization: Bearer <token>`。

## 4.1 GET `/users/me/llm-profile`

### 行为

- 若 profile 不存在：懒创建默认记录 `provider=disabled` 后返回（满足 BAI-1.1 AC）
- 永不返回明文 `apiKey`

### Response

```json
{
  "data": {
    "provider": "openai-compatible",
    "baseUrl": "https://api.example.com",
    "model": "gpt-4o-mini",
    "apiKeyMasked": "sk-***abcd",
    "timeoutSeconds": 60,
    "contextWindow": 100000,
    "healthStatus": "ok",
    "lastTestedAt": "2026-02-08T12:00:00Z",
    "effectiveAvailability": {
      "available": true,
      "code": "OK",
      "message": "可用"
    }
  },
  "error": null
}
```

## 4.2 PUT `/users/me/llm-profile`

### Request

```json
{
  "provider": "openai-compatible",
  "baseUrl": "https://api.example.com/v1",
  "model": "gpt-4o-mini",
  "apiKey": "sk-xxx",
  "timeoutSeconds": 60,
  "contextWindow": 100000
}
```

### 规则

- `provider=disabled`：允许清空 `baseUrl/model/apiKey`
- `provider=openai-compatible`：
  - 必填：`baseUrl`, `model`
  - `apiKey` 可选；若请求缺失该字段则保持旧值
  - `timeoutSeconds > 0`
- 成功后重置 `healthStatus=unknown`（直到下次 test）

### Response（成功）

返回与 GET 同结构。

## 4.3 POST `/users/me/llm-profile/test`

### Request

- 支持两种模式：
1) 不带 body：使用当前已保存 profile 测试  
2) 带 body：使用传入配置做临时测试（不落库）

```json
{
  "provider": "openai-compatible",
  "baseUrl": "https://api.example.com/v1",
  "model": "gpt-4o-mini",
  "apiKey": "sk-xxx",
  "timeoutSeconds": 30
}
```

### Response（成功）

```json
{
  "data": {
    "ok": true,
    "code": "OK",
    "message": "连接成功",
    "hints": []
  },
  "error": null
}
```

### Response（失败）

```json
{
  "data": {
    "ok": false,
    "code": "AI_PROVIDER_ERROR",
    "message": "401 unauthorized",
    "hints": ["检查 API Key", "确认 baseUrl/model 是否匹配"]
  },
  "error": null
}
```

## 4.4 错误码字典（BAI-1 相关）

- `AUTH_MISSING_TOKEN` / `AUTH_INVALID_TOKEN` / `AUTH_TOKEN_EXPIRED`
- `LLM_PROFILE_INVALID_PROVIDER`
- `LLM_PROFILE_MISSING_BASE_URL`
- `LLM_PROFILE_MISSING_MODEL`
- `LLM_PROFILE_INVALID_TIMEOUT`
- `LLM_PROFILE_ENCRYPTION_ERROR`
- `LLM_PROFILE_ORG_POLICY_BLOCKED`
- `AI_PROVIDER_ERROR`
- `AI_PROVIDER_TIMEOUT`
- `AI_PROVIDER_UNREACHABLE`

---

## 5. 实施任务拆分（已落地，供追溯）

## 5.1 Story BAI-1.1（数据模型）

### 任务

1. 新增 SQLAlchemy 模型 `UserLlmProfile`
2. 新增 Alembic migration
3. 新增 service 层 `get_or_create_profile(user_id)`
4. 新增密钥加解密工具（`app/utils/secret_crypto.py`）
5. 在 `.env.example` 补充配置项说明

### 交付文件（建议）

- `app/models/user_llm_profile.py`
- `migrations/versions/<rev>_add_user_llm_profiles.py`
- `app/utils/secret_crypto.py`
- `app/config.py`
- `.env.example`

## 5.2 Story BAI-1.2（GET/PUT API）

### 任务

1. 新增 schema：`app/schemas/llm_profile.py`
2. 新增 service：`app/services/llm_profile_service.py`
3. 新增路由：`app/routers/llm_profile.py`（或扩展 `users.py`）
4. 注册路由到 `app/main.py`
5. 接入组织策略覆盖判定（仅读）

### 交付文件（建议）

- `app/schemas/llm_profile.py`
- `app/services/llm_profile_service.py`
- `app/routers/llm_profile.py`
- `app/main.py`

## 5.3 Story BAI-1.3（连接测试 API）

### 任务

1. 新增 provider 测试器（复用 httpx）
2. 实现 `POST /users/me/llm-profile/test`
3. 测试成功时更新 `health_status/last_tested_at`（仅保存模式）
4. 统一错误映射：401/403/timeout/network -> 结构化 code

### 交付文件（建议）

- `app/services/llm_profile_test_service.py`
- `app/routers/llm_profile.py`

## 5.4 Story BAI-1.4（接入 AI Gateway）

### 任务

1. 新增 `resolve_effective_llm_config(user_id)`（Profile + OrgPolicy + fallback）
2. 修改 `ai_step_service.py`：从“全局 settings”切到“用户配置优先”
3. 无 profile 时回退到全局默认配置（通常为 `disabled`，可被组织策略继续阻断/放行）
4. 在 `/packages/{projectId}/ai/sessions/{sessionId}/messages` 链路透传 `current_user.id` 到 service

### 交付文件（建议）

- `app/services/llm_profile_service.py`
- `app/services/ai_step_service.py`
- `app/routers/packages.py`

## 5.5 Story BAI-1.5（组织策略 Hook）

### 任务

1. 新增 `OrgPolicyResolver`（先用 env 驱动，后续可接 DB）
2. 在 GET Profile 返回 `effectiveAvailability`
3. 在 AI 调用前强制执行 policy（阻止越权 provider）

### 交付文件（建议）

- `app/services/org_policy_service.py`
- `app/services/llm_profile_service.py`
- `app/services/ai_step_service.py`

---

## 6. 验收用例（Acceptance + Test Matrix）

## 6.1 API 验收（黑盒）

### 用例 A1：未登录访问 profile

- Given：无 token
- When：`GET /users/me/llm-profile`
- Then：`401` + `AUTH_MISSING_TOKEN`

### 用例 A2：首次 GET 自动建默认 profile

- Given：用户已登录且无 profile 记录
- When：`GET /users/me/llm-profile`
- Then：`200`，`provider=disabled`，数据库生成一条记录

### 用例 A3：PUT openai-compatible 缺 baseUrl/model

- Given：用户已登录
- When：`PUT /users/me/llm-profile` 且缺必填
- Then：`400` + `LLM_PROFILE_MISSING_*`

### 用例 A4：GET 不返回明文 key

- Given：数据库已有加密 key
- When：`GET /users/me/llm-profile`
- Then：`apiKeyMasked` 存在，响应内无明文 key

### 用例 A5：Test API 返回结构化失败

- Given：错误 API key 或不可达 baseUrl
- When：`POST /users/me/llm-profile/test`
- Then：`200` + `data.ok=false` + `code/hints`

### 用例 A6：用户隔离

- Given：A 用户配置 provider=openai-compatible，B 用户 provider=disabled
- When：分别 GET
- Then：两者配置互不影响

### 用例 A7：组织策略阻断

- Given：`LLM_ORG_POLICY_MODE=disable-all`
- When：PUT/TEST openai-compatible 或触发 AI 调用
- Then：返回 `LLM_PROFILE_ORG_POLICY_BLOCKED`

## 6.2 集成验收（与现有 Step AI 集成）

### 用例 I1：Session messages 使用用户级 profile

- Given：两个用户各自 profile 不同
- When：分别调用 `/packages/{projectId}/ai/sessions/{sessionId}/messages`
- Then：provider 行为按用户生效（响应 provider 字段可验证）

### 用例 I2：profile 为 disabled 时阻止 AI session

- Given：profile.provider=disabled
- When：调用 `/packages/{projectId}/ai/sessions/{sessionId}/messages`
- Then：返回 `AI_DISABLED`（或统一禁用错误码）

---

## 7. 测试文件拆分建议

新增测试文件：

- `tests/test_user_llm_profile_api.py`
  - 覆盖 A1~A6
- `tests/test_user_llm_profile_org_policy.py`
  - 覆盖 A7
- `tests/test_ai_step_profile_resolution.py`
  - 覆盖 I1~I2

建议新增 helper：

- `tests/helpers/profile_factory.py`（可选）

执行命令：

```bash
cd crewagent-builder-backend
pytest tests/test_user_llm_profile_api.py tests/test_user_llm_profile_org_policy.py tests/test_ai_step_profile_resolution.py -q
```

---

## 8. DoD（BAI-1 完成标准）

1. migration 可升级/可回滚
2. GET/PUT/TEST 三接口稳定可用，错误码可预测
3. API key 全链路不明文落库、不明文返回
4. session/messages 已切换到用户级 profile 解析
5. 组织策略 hook 生效，且有测试覆盖
