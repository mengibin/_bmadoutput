# Design: Gemini LLM Provider Support

**Story:** `8-6-gemini-llm-provider.md`  
**设计原则:** OpenAI 兼容优先、最小改动

---

## 设计目标

1. **添加 Gemini 选项**：在 LLM 设置中支持 Google Gemini
2. **API 兼容**：优先使用 Gemini 的 OpenAI 兼容接口
3. **Function Calling**：确保工具调用正常工作

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `src/types/settings.ts` | MODIFY | 扩展 LLMProvider 类型 |
| `electron/services/llmAdapter.ts` | MODIFY | 支持 Gemini API |
| `src/pages/SettingsPage/SettingsPage.tsx` | MODIFY | 添加 Gemini 配置 UI |

---

## 技术方案

### Gemini API 兼容性

Gemini 提供 OpenAI 兼容接口：

```
POST https://generativelanguage.googleapis.com/v1beta/openai/chat/completions
```

**优势**：可复用现有 OpenAI 适配器逻辑

**差异**：
- API Key 通过 Header `x-goog-api-key` 传递（而非 `Authorization: Bearer`）
- 默认无流式响应需显式开启

---

## 详细实现

### 1. LLMProvider 类型扩展

```typescript
// src/types/settings.ts

type LLMProvider = 'openai' | 'openai-compatible' | 'azure' | 'ollama' | 'gemini'

interface LLMConfig {
    provider: LLMProvider
    baseUrl: string
    model: string
    apiKey: string
    timeout: number
    contextWindow?: number
}

// Gemini 默认配置
const GEMINI_DEFAULTS = {
    baseUrl: 'https://generativelanguage.googleapis.com/v1beta/openai',
    model: 'gemini-1.5-pro',
}

// 可选模型列表
const GEMINI_MODELS = [
    { id: 'gemini-1.5-pro', name: 'Gemini 1.5 Pro' },
    { id: 'gemini-1.5-flash', name: 'Gemini 1.5 Flash' },
    { id: 'gemini-pro', name: 'Gemini Pro' },
]
```

### 2. LlmAdapter 修改

```typescript
// electron/services/llmAdapter.ts

export class LlmAdapter {
    private getHeaders(config: LLMConfig): Record<string, string> {
        const headers: Record<string, string> = {
            'Content-Type': 'application/json',
        }
        
        switch (config.provider) {
            case 'gemini':
                // Gemini 使用不同的认证头
                headers['x-goog-api-key'] = config.apiKey
                break
            case 'azure':
                headers['api-key'] = config.apiKey
                break
            default:
                headers['Authorization'] = `Bearer ${config.apiKey}`
        }
        
        return headers
    }
    
    private getEndpoint(config: LLMConfig): string {
        const baseUrl = config.baseUrl.replace(/\/$/, '')
        
        switch (config.provider) {
            case 'gemini':
                // Gemini OpenAI 兼容 endpoint
                return `${baseUrl}/chat/completions`
            case 'azure':
                return `${baseUrl}/openai/deployments/${config.model}/chat/completions?api-version=2024-02-15-preview`
            default:
                return `${baseUrl}/chat/completions`
        }
    }
    
    async call(params: LlmCallParams): Promise<LlmCallResult> {
        const config = params.config || this.defaultConfig
        const endpoint = this.getEndpoint(config)
        const headers = this.getHeaders(config)
        
        const body = {
            model: config.provider === 'gemini' 
                ? config.model  // Gemini 使用模型 ID
                : config.model,
            messages: params.messages,
            tools: params.tools,
            stream: params.stream ?? false,
            temperature: params.temperature ?? 0.7,
            max_tokens: params.max_tokens,
        }
        
        try {
            const response = await fetch(endpoint, {
                method: 'POST',
                headers,
                body: JSON.stringify(body),
                signal: AbortSignal.timeout(config.timeout * 1000),
            })
            
            if (!response.ok) {
                const error = await response.text()
                return { 
                    success: false, 
                    error: { 
                        code: 'LLM_API_ERROR', 
                        message: `Gemini API error: ${response.status} - ${error}` 
                    } 
                }
            }
            
            const data = await response.json()
            return this.parseResponse(data)
        } catch (error) {
            return { 
                success: false, 
                error: { 
                    code: 'LLM_REQUEST_FAILED', 
                    message: String(error) 
                } 
            }
        }
    }
}
```

### 3. Settings UI 修改

```tsx
// src/pages/SettingsPage/SettingsPage.tsx

const GEMINI_MODELS = [
    { id: 'gemini-1.5-pro', name: 'Gemini 1.5 Pro' },
    { id: 'gemini-1.5-flash', name: 'Gemini 1.5 Flash' },
    { id: 'gemini-pro', name: 'Gemini Pro' },
]

function LLMSettings() {
    const [config, setConfig] = useState<LLMConfig>(defaultConfig)
    
    const handleProviderChange = (provider: LLMProvider) => {
        let updates: Partial<LLMConfig> = { provider }
        
        // 设置提供商默认值
        switch (provider) {
            case 'gemini':
                updates = {
                    ...updates,
                    baseUrl: 'https://generativelanguage.googleapis.com/v1beta/openai',
                    model: 'gemini-1.5-pro',
                }
                break
            case 'openai':
                updates = {
                    ...updates,
                    baseUrl: 'https://api.openai.com/v1',
                    model: 'gpt-4',
                }
                break
            // ... other providers
        }
        
        setConfig(prev => ({ ...prev, ...updates }))
    }
    
    return (
        <div className="settings-section">
            <FormField label="Provider">
                <Select 
                    value={config.provider} 
                    onChange={(e) => handleProviderChange(e.target.value as LLMProvider)}
                >
                    <option value="openai">OpenAI</option>
                    <option value="openai-compatible">OpenAI Compatible</option>
                    <option value="azure">Azure OpenAI</option>
                    <option value="ollama">Ollama</option>
                    <option value="gemini">Google Gemini</option>
                </Select>
            </FormField>
            
            <FormField label="API Key" required>
                <Input
                    type="password"
                    value={config.apiKey}
                    onChange={(e) => setConfig(prev => ({ ...prev, apiKey: e.target.value }))}
                    placeholder={config.provider === 'gemini' 
                        ? 'Enter your Gemini API Key' 
                        : 'Enter your API Key'}
                />
            </FormField>
            
            {config.provider === 'gemini' && (
                <>
                    <FormField label="Model">
                        <Select 
                            value={config.model}
                            onChange={(e) => setConfig(prev => ({ ...prev, model: e.target.value }))}
                        >
                            {GEMINI_MODELS.map(m => (
                                <option key={m.id} value={m.id}>{m.name}</option>
                            ))}
                        </Select>
                    </FormField>
                    
                    <FormField label="Base URL">
                        <Input
                            value={config.baseUrl}
                            onChange={(e) => setConfig(prev => ({ ...prev, baseUrl: e.target.value }))}
                            placeholder="https://generativelanguage.googleapis.com/v1beta/openai"
                        />
                        <small>Default: Gemini OpenAI-compatible endpoint</small>
                    </FormField>
                </>
            )}
            
            {/* ... other fields ... */}
        </div>
    )
}
```

---

## Function Calling 兼容性

Gemini 的 OpenAI 兼容接口支持标准 function calling 格式：

```json
{
    "model": "gemini-1.5-pro",
    "messages": [...],
    "tools": [
        {
            "type": "function",
            "function": {
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
        }
    ]
}
```

**预期响应格式**：与 OpenAI 一致，包含 `tool_calls`

---

## 错误处理

```typescript
// Gemini 特有错误处理
function parseGeminiError(response: Response, body: string): string {
    try {
        const data = JSON.parse(body)
        if (data.error?.message) {
            return data.error.message
        }
    } catch {}
    
    // 常见错误代码
    switch (response.status) {
        case 400: return 'Invalid request. Check your model or parameters.'
        case 401: return 'Invalid API key. Check your Gemini API key.'
        case 403: return 'API key does not have access to this resource.'
        case 429: return 'Rate limit exceeded. Please wait and try again.'
        case 500: return 'Gemini server error. Please try again later.'
        default: return `API error: ${response.status}`
    }
}
```

---

## 测试策略

### 手动测试

1. 在 Settings 中选择 Gemini provider
2. 输入 Gemini API Key
3. 选择模型（gemini-1.5-pro）
4. 执行工作流或对话
5. 验证 Function Calling 正常

### 自动测试

```typescript
describe('Gemini LLM Adapter', () => {
    it('should use correct auth header', () => {
        const headers = adapter.getHeaders(geminiConfig)
        expect(headers['x-goog-api-key']).toBe('test-key')
        expect(headers['Authorization']).toBeUndefined()
    })
    
    it('should construct correct endpoint', () => {
        const endpoint = adapter.getEndpoint(geminiConfig)
        expect(endpoint).toContain('/chat/completions')
    })
    
    it('should handle tool calls correctly', async () => {})
})
```

---

## 向后兼容

- 不影响现有 OpenAI/Azure/Ollama 配置
- 新增 `'gemini'` 选项，现有用户无感知
