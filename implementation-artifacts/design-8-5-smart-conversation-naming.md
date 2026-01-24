# Design: Smart Conversation Naming

**Story:** `8-5-smart-conversation-naming.md`  
**设计原则:** 异步非阻塞、快速响应、用户可覆盖

---

## 设计目标

1. **智能命名**：使用 LLM 根据首条消息生成有意义的标题
2. **快速响应**：使用轻量模型，异步生成不阻塞 UI
3. **用户可控**：支持手动重命名覆盖

---

## 改动范围

| 文件 | 改动类型 | 说明 |
|------|----------|------|
| `src/shared/conversationTypes.ts` | MODIFY | 扩展 ConversationMetadata |
| `electron/services/titleGenerator.ts` | NEW | 标题生成服务 |
| `electron/main.ts` | MODIFY | 添加标题生成触发逻辑 |
| `src/components/ConversationList.tsx` | MODIFY | 显示智能标题，支持重命名 |

---

## 数据结构

### ConversationMetadata 扩展

```typescript
interface ConversationMetadata {
    id: string
    createdAt: string
    updatedAt: string
    agentId?: string
    workflowId?: string
    runCount?: number
    
    // NEW: Smart naming fields
    generatedTitle?: string   // LLM 生成的标题
    customTitle?: string      // 用户自定义标题
}

// 显示优先级: customTitle > generatedTitle > workflowId > 'New Conversation'
function getDisplayTitle(conv: ConversationMetadata): string {
    return conv.customTitle 
        || conv.generatedTitle 
        || conv.workflowId 
        || 'New Conversation'
}
```

---

## 数据流图

```
用户发送首条消息
        │
        ▼
┌────────────────────────┐
│ LLM 处理，返回响应      │
└────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│ 检查: generatedTitle 为空?      │
└────────────────────────────────┘
        │
       Yes
        │
        ▼
┌────────────────────────────────┐
│ 异步调用 generateTitle()       │
│ (不阻塞主流程)                  │
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│ 更新 ConversationMetadata      │
│ generatedTitle = "..."         │
└────────────────────────────────┘
        │
        ▼
┌────────────────────────────────┐
│ IPC 通知前端更新 UI            │
└────────────────────────────────┘
```

---

## 详细实现

### 1. titleGenerator 服务

```typescript
// electron/services/titleGenerator.ts

import type { LlmAdapter } from './llmAdapter'

const TITLE_PROMPT = `Generate a very short title (max 30 chars) for this conversation based on the user's first message.

User message:
{user_message}

Requirements:
- Maximum 30 characters
- Concise and descriptive
- Capture the main topic/intent
- If the message is in Chinese, output a Chinese title
- If the message is in English, output an English title
- No quotes, punctuation at end, or special characters

Output only the title, nothing else.`

export async function generateConversationTitle(
    llmAdapter: LlmAdapter,
    userMessage: string,
    config: { model?: string; timeout?: number } = {}
): Promise<string | null> {
    try {
        // 截取前 500 字符，避免过长
        const truncatedMessage = userMessage.slice(0, 500)
        const prompt = TITLE_PROMPT.replace('{user_message}', truncatedMessage)
        
        const response = await llmAdapter.call({
            messages: [{ role: 'user', content: prompt }],
            model: config.model || 'gpt-3.5-turbo', // 使用快速模型
            temperature: 0.3,
            max_tokens: 50,
            timeout: config.timeout || 5000, // 5秒超时
        })
        
        if (!response.success || !response.message?.content) {
            return null
        }
        
        // 清理生成的标题
        let title = response.message.content.trim()
        title = title.replace(/^["']|["']$/g, '') // 移除引号
        title = title.replace(/[。.!！?？]$/, '') // 移除末尾标点
        
        // 限制长度
        if (title.length > 30) {
            title = title.slice(0, 27) + '...'
        }
        
        return title || null
    } catch (error) {
        console.warn('[TitleGenerator] Failed to generate title:', error)
        return null
    }
}
```

### 2. 触发逻辑集成

```typescript
// electron/main.ts

// 在 callAgentChat 或消息处理后

async function handleFirstMessageTitleGeneration(
    projectRoot: string,
    conversationId: string,
    userMessage: string
) {
    // 检查是否已有标题
    const conversations = runtimeStore.loadConversations(projectRoot)
    const conv = conversations.find(c => c.id === conversationId)
    
    if (conv?.generatedTitle) {
        return // 已有标题，跳过
    }
    
    // 异步生成标题
    const title = await generateConversationTitle(llmAdapter, userMessage)
    
    if (title) {
        // 更新 conversation metadata
        runtimeStore.updateConversation(projectRoot, conversationId, {
            generatedTitle: title,
        })
        
        // 通知前端更新
        sendIpcEvent(undefined, 'conversation:titleUpdated', {
            conversationId,
            generatedTitle: title,
        })
    }
}
```

### 3. RuntimeStore 更新方法

```typescript
// electron/stores/runtimeStore.ts

class RuntimeStore {
    /**
     * 更新 Conversation 元数据
     */
    public updateConversation(
        projectRoot: string,
        conversationId: string,
        updates: Partial<ConversationMetadata>
    ): { success: boolean } {
        const conversations = this.loadConversations(projectRoot)
        const index = conversations.findIndex(c => c.id === conversationId)
        
        if (index === -1) {
            return { success: false }
        }
        
        conversations[index] = {
            ...conversations[index],
            ...updates,
            updatedAt: new Date().toISOString(),
        }
        
        this.saveConversationsIndex(projectRoot, conversations)
        return { success: true }
    }
}
```

### 4. ConversationList 组件修改

```tsx
// src/components/ConversationList.tsx

import { useState } from 'react'
import { Edit2 } from 'lucide-react'

function ConversationItem({ conversation, onSelect, onRename }) {
    const [isEditing, setIsEditing] = useState(false)
    const [editValue, setEditValue] = useState('')
    
    const displayTitle = conversation.customTitle 
        || conversation.generatedTitle 
        || conversation.workflowId 
        || 'New Conversation'
    
    const handleStartEdit = (e: React.MouseEvent) => {
        e.stopPropagation()
        setEditValue(displayTitle)
        setIsEditing(true)
    }
    
    const handleSave = () => {
        if (editValue.trim() && editValue !== displayTitle) {
            onRename(conversation.id, editValue.trim())
        }
        setIsEditing(false)
    }
    
    return (
        <div 
            className="conversation-item"
            onClick={() => onSelect(conversation)}
            title={displayTitle} // Tooltip 显示完整名称
        >
            {isEditing ? (
                <input
                    value={editValue}
                    onChange={(e) => setEditValue(e.target.value)}
                    onBlur={handleSave}
                    onKeyDown={(e) => e.key === 'Enter' && handleSave()}
                    autoFocus
                    onClick={(e) => e.stopPropagation()}
                />
            ) : (
                <>
                    <span className="conversation-title">
                        {truncate(displayTitle, 28)}
                    </span>
                    <button 
                        className="rename-btn"
                        onClick={handleStartEdit}
                        title="Rename"
                    >
                        <Edit2 size={14} />
                    </button>
                </>
            )}
        </div>
    )
}
```

### 5. IPC 事件监听

```tsx
// src/hooks/useConversationUpdates.ts

useEffect(() => {
    const handler = (event: any, data: { conversationId: string; generatedTitle: string }) => {
        // 更新本地状态
        setConversations(prev => prev.map(c => 
            c.id === data.conversationId 
                ? { ...c, generatedTitle: data.generatedTitle }
                : c
        ))
    }
    
    window.electron.on('conversation:titleUpdated', handler)
    return () => window.electron.off('conversation:titleUpdated', handler)
}, [])
```

---

## 性能考虑

| 考虑点 | 方案 |
|--------|------|
| 延迟 | 使用快速模型 (gpt-3.5-turbo, gemini-flash) |
| 成本 | 每对话仅调用一次 |
| 失败 | 静默失败，保持默认名称 |
| 超时 | 5 秒超时，避免阻塞 |

---

## 测试策略

### 手动测试

1. 开始新对话，发送消息
2. 验证几秒后标题自动更新
3. 重命名对话，验证自定义名称生效
4. 验证 tooltip 显示完整名称

### 自动测试

```typescript
describe('generateConversationTitle', () => {
    it('should generate title from user message', async () => {})
    it('should truncate long titles to 30 chars', async () => {})
    it('should return null on error', async () => {})
    it('should handle Chinese messages', async () => {})
})
```
