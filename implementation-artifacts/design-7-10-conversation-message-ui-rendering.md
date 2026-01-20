# Design: Conversation Message UI Rendering

**Story:** `7-10-conversation-message-ui-rendering.md`  
**设计原则:** 纯前端改动、向后兼容、最小侵入

---

## 设计目标

1. **正确显示**：重新加载对话后，tool 消息显示为指示器而非聊天气泡
2. **一致性**：与实时对话显示效果一致
3. **向后兼容**：旧消息（无新字段）正常处理

---

## 核心方案

在渲染层推断 `partType`，而非修改存储层。

---

## 详细实现

### 1. 消息类型推断函数

```typescript
// src/utils/messageRenderer.ts

import type { ConversationMessage } from '../../shared/conversationTypes'

/**
 * 根据消息结构推断 partType（用于重新加载的消息）
 */
export function inferPartType(msg: ConversationMessage): string {
    // 已有 partType 直接返回
    if (msg.partType) return msg.partType
    
    // role='tool' → tool_result
    if (msg.role === 'tool') {
        return 'tool_result'
    }
    
    // role='assistant' + tool_calls → tool_start
    if (msg.role === 'assistant' && msg.tool_calls?.length) {
        return 'tool_start'
    }
    
    // 默认
    return 'content'
}

/**
 * 判断消息是否应该显示为聊天气泡
 */
export function shouldRenderAsChatBubble(msg: ConversationMessage): boolean {
    const partType = inferPartType(msg)
    
    // 工具相关消息不显示为聊天气泡
    if (partType === 'tool_start' || partType === 'tool_result') {
        return false
    }
    
    // thinking 消息已有特殊处理
    if (partType === 'thinking') {
        return false
    }
    
    return true
}

/**
 * 为重新加载的消息补充 partType
 */
export function enrichMessage(msg: ConversationMessage): ConversationMessage {
    if (msg.partType) return msg
    
    return {
        ...msg,
        partType: inferPartType(msg),
        // 补充 toolName（从 tool_calls 或 tool_call_id 推断）
        toolName: msg.toolName ?? msg.tool_calls?.[0]?.function?.name,
    }
}
```

### 2. MessageList 改动

```diff
// src/pages/RunsPage/components/MessageList.tsx

import { useEffect, useRef, useState } from 'react'
import type { ConversationMessage } from '../../../stores/appStore'
import { MessageItem } from './MessageItem'
+ import { enrichMessage } from '../../../utils/messageRenderer'

export function MessageList({
    messages,
    ...
}: MessageListProps) {
    // ...
    
+   // 补充 partType 信息
+   const enrichedMessages = messages.map(enrichMessage)
    
    return (
        <div className="chat-messages" ref={scrollRef} onScroll={handleScroll}>
            <div className="chat-messages-inner">
-               {messages.map((message) => (
+               {enrichedMessages.map((message) => (
                    <MessageItem
                        key={message.id}
                        message={message}
                        ...
                    />
                ))}
                <div ref={endRef} />
            </div>
        </div>
    )
}
```

### 3. MessageItem 改动（可选优化）

```diff
// 添加对 role='tool' 的显式处理（如果 partType 推断不够）

+ if (message.role === 'tool') {
+     return (
+         <div className="message assistant assistant-with-avatar indicator-message">
+             {assistantAvatar}
+             <div className="assistant-indicator-body">
+                 <ToolStatusIndicator
+                     toolName={message.toolName ?? 'tool'}
+                     status="completed"
+                     durationMs={message.duration}
+                     onClick={handleIndicatorClick}
+                 />
+             </div>
+         </div>
+     )
+ }
```

---

## 文件改动

| File | Type | Changes |
|------|------|---------|
| `src/utils/messageRenderer.ts` | **NEW** | 消息类型推断函数 |
| `src/pages/RunsPage/components/MessageList.tsx` | MODIFY | 调用 enrichMessage |
| `src/pages/RunsPage/components/MessageItem.tsx` | MODIFY (optional) | 处理 role='tool' |

---

## 测试策略

### Unit Tests

```typescript
// src/utils/messageRenderer.test.ts

describe('inferPartType', () => {
    it('returns existing partType if present', () => {
        expect(inferPartType({ partType: 'thinking', ... })).toBe('thinking')
    })
    
    it('returns tool_result for role=tool', () => {
        expect(inferPartType({ role: 'tool', ... })).toBe('tool_result')
    })
    
    it('returns tool_start for assistant with tool_calls', () => {
        expect(inferPartType({ 
            role: 'assistant', 
            tool_calls: [{ id: '1', ... }] 
        })).toBe('tool_start')
    })
    
    it('returns content for regular assistant message', () => {
        expect(inferPartType({ role: 'assistant', content: 'hello' })).toBe('content')
    })
})

describe('enrichMessage', () => {
    it('does not modify message with existing partType', () => {
        const msg = { partType: 'content', ... }
        expect(enrichMessage(msg)).toBe(msg)
    })
    
    it('adds partType and toolName for tool message', () => {
        const msg = { role: 'tool', tool_call_id: '1', content: 'result' }
        expect(enrichMessage(msg)).toMatchObject({
            partType: 'tool_result',
        })
    })
})
```

### 运行测试

```bash
cd crewagent-runtime
npm test -- src/utils/messageRenderer.test.ts
```

### Manual Testing

1. 打开 Chat 模式，发送消息触发工具调用
2. 刷新页面 / 重新打开对话
3. 验证：
   - 工具调用显示为指示器（非聊天气泡）
   - 工具结果显示为指示器（非聊天气泡）
   - 普通消息正常显示
4. 在 Agent/Run 模式重复测试

---

## 依赖

- ✅ **Story 7-2**: Persist Tool-Call Protocol Messages (done)

---

## Related Artifacts
- **Story Document**: [7-10-conversation-message-ui-rendering.md](./7-10-conversation-message-ui-rendering.md)
- **Validation Report**: [validation-report-story-7-10.md](./validation-report-story-7-10.md)
