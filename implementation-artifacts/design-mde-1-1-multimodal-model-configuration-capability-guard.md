# Design: Multimodal Model Configuration + Capability Guard

**Story:** `mde-1-1-multimodal-model-configuration-capability-guard.md`  
**设计原则:** 配置解耦、fail-fast、可观测优先

---

## 设计目标

1. **配置解耦**：在现有 `llm` 之外引入独立 `multimodalLlm` 配置，不影响文本链路。
2. **能力前置校验**：多模态提取前先做 capability guard，不支持时统一返回 `LLM_MULTIMODAL_NOT_SUPPORTED`。
3. **可审计**：记录校验结果与关键上下文，不记录密钥。

---

## 范围与非范围

### In Scope (MDE-1.1)
- Runtime Settings 增加 `multimodalLlm` 配置项（`provider/baseUrl/model/apiKey/timeout/contextWindow?`）。
- 主进程引入 `MultimodalCapabilityService`（模型能力判定 + 统一错误）。
- 在提取入口增加 guard 调用契约（当前以服务层 seam 落地，供 MDE-1.2 直接复用）。
- 日志/审计字段补齐：`runId/provider/model/capabilityCheck/result/errorCode`。

### Out of Scope (MDE-1.1)
- `media.extract` first-class tool 完整实现（MDE-1.2）。
- 拖拽批处理与 schema 编排（MDE-1.3）。
- 全局多模态消息协议改造（后续阶段）。

---

## 关键决策

### 决策 1：`RuntimeSettings` 增加 `multimodalLlm`（不覆盖 `llm`）
- 原因：当前 `llm` 已用于 chat/workflow 主链路，直接复用会引入回归风险。
- 结果：提取链路默认读取 `multimodalLlm`；未配置时显式失败，不隐式降级到 `llm`。

### 决策 2：能力校验采用“显式 allowlist + 可选 probe”
- 原因：仅靠 provider 无法判断 model 是否支持视觉输入。
- 结果：
  - MVP 先用本地 allowlist/规则（低成本、可控）。
  - 预留 probe 接口用于后续在线探测（不阻塞本 story）。

### 决策 3：错误码统一落在结构化错误对象
- 统一形状：`{ code, message, details? }`
- 本 story 新增/约定：`LLM_MULTIMODAL_NOT_SUPPORTED`

---

## 数据模型设计

```ts
type LLMProvider = 'openai' | 'openai-compatible' | 'azure' | 'ollama' | 'gemini'

interface LLMConfig {
  provider: LLMProvider
  baseUrl: string
  model: string
  apiKey: string
  timeout: number
  contextWindow?: number
}

interface RuntimeSettings {
  theme: 'system' | 'light' | 'dark'
  llm: LLMConfig
  multimodalLlm?: LLMConfig
  // ... existing fields
}

interface MultimodalCapabilityResult {
  ok: boolean
  provider: LLMProvider
  model: string
  reason?: string
  error?: { code: 'LLM_MULTIMODAL_NOT_SUPPORTED'; message: string; details?: unknown }
}
```

---

## 处理流程

### A. Settings 保存/读取
1. Renderer 提交 `settings:update({ multimodalLlm })`
2. Main `RuntimeStore.updateSettings` 校验并持久化
3. Renderer `initialize/saveSettings` 同步 `multimodalLlm` 到状态

### B. 多模态提取前置校验
1. 提取入口调用 `assertMultimodalReady(context)`
2. `MultimodalCapabilityService` 读取 `settings.multimodalLlm`
3. 若不支持：
   - 返回 `LLM_MULTIMODAL_NOT_SUPPORTED`
   - 阻断下游 provider 请求（fail-fast）
4. 若支持：进入后续提取逻辑（MDE-1.2 实现）

---

## 模块与文件改动

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `crewagent-runtime/electron/stores/runtimeStore.ts` | MODIFY | 扩展 `RuntimeSettings`、默认值与 `updateSettings` 合并逻辑 |
| `crewagent-runtime/src/stores/appStore.ts` | MODIFY | 增加 `multimodalLlmConfig` state + save/load action |
| `crewagent-runtime/src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 新增 Multimodal 配置区（字段、校验、保存） |
| `crewagent-runtime/electron/services/multimodalCapabilityService.ts` | NEW | 能力校验与 `assertMultimodalReady` |
| `crewagent-runtime/electron/services/fileSystemToolHost.ts` | MODIFY | 预留/接入 guard seam（供 MDE-1.2 调用） |
| `crewagent-runtime/electron/stores/runtimeStore.test.ts` | MODIFY | 覆盖配置持久化与非法输入保护 |
| `crewagent-runtime/electron/services/fileSystemToolHost.test.ts` | MODIFY | 覆盖“不支持模型 -> 阻断并返回错误码” |

---

## 兼容性与迁移

1. `settings.json` 无 `multimodalLlm` 时按可选字段处理，不破坏旧版本。
2. `llm` 行为保持原样，不因 `multimodalLlm` 引入默认切换。
3. 日志中对 `apiKey` 强制脱敏（仅记录 provider/model）。

---

## 错误与日志约定

### 结构化错误
```json
{
  "ok": false,
  "error": {
    "code": "LLM_MULTIMODAL_NOT_SUPPORTED",
    "message": "Configured model does not support multimodal input",
    "details": { "provider": "openai-compatible", "model": "deepseek-chat" }
  }
}
```

### 审计字段
- `runId`
- `toolName`（后续可为 `media.extract`）
- `provider`
- `model`
- `capabilityCheck`（`pass`/`fail`）
- `errorCode`（失败时）
- `durationMs`

---

## 测试设计

### Unit
1. `runtimeStore.updateSettings`：
   - 合法 `multimodalLlm` 可持久化
   - 非法字段/类型被拒绝或回退
2. `multimodalCapabilityService`：
   - 支持模型返回 `ok=true`
   - 不支持模型返回 `LLM_MULTIMODAL_NOT_SUPPORTED`

### Integration
1. Settings 保存后重启，`multimodalLlm` 仍可读取。
2. 触发提取入口时，unsupported 模型不发起 provider 调用。

### Regression
1. 原 `llm` 文本对话链路不受影响。
2. 既有 `settings:get/settings:update` IPC 契约兼容。

---

## 交付边界（与后续 Story 对齐）

- MDE-1.1 交付：配置模型 + guard + observability。
- MDE-1.2 在此基础上接入 first-class `media.extract`。
- MDE-1.3 继续接入拖拽批处理与 schema 流程。
