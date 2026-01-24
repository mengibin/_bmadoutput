# Story 8.6: Gemini LLM Provider Support

## 概述

添加 Google Gemini 作为 LLM Provider 选项，让用户可以使用 Gemini 模型执行工作流和对话。

---

## 用户故事

As a **Consumer**,
I want to use Google Gemini as my LLM provider,
So that I can leverage Gemini's capabilities for workflow execution.

---

## 验收标准

### AC-1: Provider 选项可见

**Given** I am on the Settings → LLM page
**When** I view the provider dropdown
**Then** I see "Gemini" as one of the options

### AC-2: Gemini 配置字段

**Given** I select "Gemini" from the provider dropdown
**When** the form updates
**Then** I see Gemini-specific configuration fields:
  - API Key
  - Model (manual input, e.g. gemini-1.5-pro / gemini-1.5-flash / gemini-pro)
  - Base URL (optional, for custom endpoints)

### AC-3: 工作流执行

**Given** I have configured Gemini as my provider
**When** I execute a workflow or chat with an agent
**Then** the request is sent to the Gemini API
**And** tool calls/function calling work correctly

### AC-4: 错误处理

**Given** Gemini API returns an error (quota exceeded, invalid key, etc.)
**When** the error occurs
**Then** a clear, user-friendly error message is shown

---

## 技术设计

### 1. LLMProvider 类型扩展

```typescript
type LLMProvider = 'openai' | 'openai-compatible' | 'azure' | 'ollama' | 'gemini'
```

### 2. Gemini 配置

```typescript
interface LLMConfig {
    provider: LLMProvider
    baseUrl: string  // For Gemini: 'https://generativelanguage.googleapis.com/v1beta'
    model: string    // e.g., 'gemini-1.5-pro', 'gemini-1.5-flash'
    apiKey: string
    timeout: number
    contextWindow?: number
}

// Default Gemini config
const geminiDefaultConfig = {
    baseUrl: 'https://generativelanguage.googleapis.com/v1beta',
    model: 'gemini-1.5-pro',
}
```

### 3. Gemini API 适配

Gemini 支持 OpenAI 兼容格式，可以通过修改 endpoint 来复用现有适配器：

```typescript
// In LlmAdapter
function getApiEndpoint(config: LLMConfig): string {
    switch (config.provider) {
        case 'gemini':
            // Use OpenAI-compatible endpoint
            return `${config.baseUrl}/openai/chat/completions`
        case 'openai':
            return `${config.baseUrl}/chat/completions`
        // ... other providers
    }
}
```

Gemini OpenAI-compatible endpoint 通常要求使用 `Authorization: Bearer <API_KEY>`；部分场景可兼容 `?key=<API_KEY>` 形式（建议仅作为 fallback）。

或者使用 Gemini 原生 API：

```typescript
// Gemini native API endpoint
const endpoint = `${baseUrl}/models/${model}:generateContent?key=${apiKey}`

// Request format
const request = {
    contents: [
        { role: 'user', parts: [{ text: userMessage }] }
    ],
    tools: [{ functionDeclarations: [...] }],
    generationConfig: {
        temperature: 0.7,
        maxOutputTokens: 8192,
    }
}
```

### 4. Settings UI 修改

```tsx
// In SettingsPage.tsx - LLM section
{config.provider === 'gemini' && (
    <>
        <FormField label="API Key" required>
            <Input 
                type="password"
                value={config.apiKey}
                onChange={(e) => handleChange('apiKey', e.target.value)}
                placeholder="Enter your Gemini API Key"
            />
        </FormField>
        <FormField label="Model">
            <Input
                value={config.model}
                onChange={(e) => handleChange('model', e.target.value)}
                placeholder="gemini-1.5-pro"
            />
        </FormField>
        <FormField label="Base URL (Optional)">
            <Input 
                value={config.baseUrl}
                onChange={(e) => handleChange('baseUrl', e.target.value)}
                placeholder="https://generativelanguage.googleapis.com/v1beta"
            />
        </FormField>
    </>
)}
```

---

## Gemini API 说明

### 认证

```bash
# 推荐：OpenAI-compatible 方式用 Authorization header
curl -X POST "https://generativelanguage.googleapis.com/v1beta/openai/chat/completions" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-1.5-pro","messages":[{"role":"user","content":"hi"}]}'

# 兼容方式（可选）：API Key 通过 URL 参数传递（仅建议作为 fallback）
curl -X POST "https://generativelanguage.googleapis.com/v1beta/openai/chat/completions?key=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gemini-1.5-pro","messages":[{"role":"user","content":"hi"}]}'
```

### Function Calling 格式

```json
{
    "contents": [...],
    "tools": [{
        "functionDeclarations": [
            {
                "name": "fs_read",
                "description": "Read file content",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "path": { "type": "string" }
                    },
                    "required": ["path"]
                }
            }
        ]
    }]
}
```

---

## 实现步骤

1. 扩展 `LLMProvider` 类型，添加 `'gemini'`
2. 在 `LlmAdapter` 中添加 Gemini API 支持
3. 处理 Gemini 特有的请求/响应格式
4. 在 Settings UI 添加 Gemini provider 选项和配置字段
5. 提供 Gemini 模型示例（手动输入）
6. 测试 Function Calling 兼容性

---

## 影响分析

| 组件 | 变更 | 风险 |
|:-----|:-----|:-----|
| `LLMProvider` type | 添加 `'gemini'` | 低 |
| `LlmAdapter` | 添加 Gemini API 支持 | 中 |
| Settings UI | 添加 Gemini 配置 | 低 |
| 现有流程 | 不影响其他 Provider | 无 |
