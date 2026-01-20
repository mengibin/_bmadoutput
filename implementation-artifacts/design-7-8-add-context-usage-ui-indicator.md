# Design: Add Context Usage UI Indicator

**Story:** `7-8-add-context-usage-ui-indicator.md`  
**设计原则:** 复用现有 IPC 模式、渐进式提示、低侵入性

---

## 设计目标

1. **复用 IPC 模式**：使用与 `llm:stream-*` 相同的事件模式
2. **实时更新**：每次 LLM 调用后更新
3. **渐进式提示**：normal → warning → critical → compressing

---

## 改动概览

| 文件 | 改动类型 | 说明 |
|------|---------|------|
| `electron/services/chatToolLoop.ts` | MODIFY | 添加 `onContextUsage` 回调 |
| `electron/main.ts` | MODIFY | 发送 `context:usage` IPC 事件 |
| `electron/preload.ts` | MODIFY | 添加 `onContextUsage` 方法 |
| `src/stores/appStore.ts` | MODIFY | 添加 contextUsage 状态 |
| `src/components/ContextIndicator.tsx` | NEW | 指示器组件 |
| `src/pages/ChatPage/` | MODIFY | 集成组件 |

---

## 详细实现

### 1. 类型定义

```typescript
// shared/types.ts 或 electron/types.ts

export interface ContextUsageEvent {
    usedTokens: number
    totalBudget: number
    usagePercent: number
    remaining: number
    status: 'normal' | 'warning' | 'critical' | 'compressing'
}
```

### 2. chatToolLoop 改动

```diff
// electron/services/chatToolLoop.ts

export async function chatToolLoop(params: {
    // ...existing params...
+   onContextUsage?: (usage: ContextUsageEvent) => void
}): Promise<ChatToolLoopResult> {
    
+   // 计算状态辅助函数
+   function getUsageStatus(percent: number, isCompressing: boolean): string {
+       if (isCompressing) return 'compressing'
+       if (percent > 0.8) return 'critical'
+       if (percent > 0.6) return 'warning'
+       return 'normal'
+   }
    
    for (let turn = 0; turn < maxTurns; turn++) {
        // 压缩检查
        if (params.compressionPipeline) {
            const usage = params.compressionPipeline.getUsage(toCompressionMessages(params.messages))
            
+           // 发送使用情况
+           params.onContextUsage?.({
+               usedTokens: usage.usedTokens,
+               totalBudget: usage.totalBudget,
+               usagePercent: usage.usagePercent,
+               remaining: usage.remaining,
+               status: getUsageStatus(usage.usagePercent, false),
+           })
            
            if (params.compressionPipeline.shouldCompress(...)) {
+               // 发送压缩中状态
+               params.onContextUsage?.({
+                   ...usage,
+                   status: 'compressing',
+               })
                
                // 执行压缩
                const compressed = applyCompression(...)
                
+               // 发送压缩后状态
+               const newUsage = params.compressionPipeline.getUsage(...)
+               params.onContextUsage?.({
+                   ...newUsage,
+                   status: getUsageStatus(newUsage.usagePercent, false),
+               })
            }
        }
        
        // LLM 调用...
    }
    
+   // Loop 结束后发送最终状态
+   if (params.compressionPipeline && params.onContextUsage) {
+       const finalUsage = params.compressionPipeline.getUsage(toCompressionMessages(params.messages))
+       params.onContextUsage({
+           ...finalUsage,
+           status: getUsageStatus(finalUsage.usagePercent, false),
+       })
+   }
    
    return result
}
```

### 3. main.ts 改动

```diff
// electron/main.ts - 在 callAgentChat 中

const loopRes = await chatToolLoop({
    // ...existing params...
+   onContextUsage: (usage) => {
+       if (params.streamTarget) {
+           sendIpcEvent(params.streamTarget, 'context:usage', usage)
+       }
+   },
})
```

### 4. preload.ts 改动

```diff
// electron/preload.ts

contextBridge.exposeInMainWorld('ipcRenderer', {
    // ...existing methods...
    
+   // Story 7-8: Context Usage Indicator
+   onContextUsage: (callback: (payload: ContextUsageEvent) => void) => {
+       const listener = (_: unknown, payload: unknown) => callback(payload as ContextUsageEvent)
+       ipcRenderer.on('context:usage', listener)
+       return () => ipcRenderer.removeListener('context:usage', listener)
+   },
})
```

### 5. appStore.ts 改动

```diff
// src/stores/appStore.ts

interface AppState {
    // ...existing state...
+   contextUsage: {
+       usedTokens: number
+       totalBudget: number
+       usagePercent: number
+       status: 'normal' | 'warning' | 'critical' | 'compressing'
+   } | null
}

export const useAppStore = create<AppState>((set) => ({
    // ...existing state...
+   contextUsage: null,
+   setContextUsage: (usage) => set({ contextUsage: usage }),
}))
```

### 6. ContextIndicator.tsx 组件

```tsx
// src/components/ContextIndicator.tsx

import { useAppStore } from '@/stores/appStore'

const STATUS_COLORS = {
    normal: 'bg-green-500',
    warning: 'bg-yellow-500',
    critical: 'bg-red-500',
    compressing: 'bg-blue-500 animate-pulse',
}

export function ContextIndicator() {
    const contextUsage = useAppStore((state) => state.contextUsage)
    
    if (!contextUsage) return null
    
    const { usagePercent, status } = contextUsage
    const percentage = Math.round(usagePercent * 100)
    
    return (
        <div className="flex items-center gap-2 text-sm text-gray-500">
            <div className="flex gap-0.5">
                {[...Array(5)].map((_, i) => (
                    <div
                        key={i}
                        className={`w-2 h-2 rounded-full ${
                            i < Math.ceil(usagePercent * 5)
                                ? STATUS_COLORS[status]
                                : 'bg-gray-300'
                        }`}
                    />
                ))}
            </div>
            <span>{percentage}%</span>
        </div>
    )
}
```

### 7. ChatPage 集成

```diff
// src/pages/ChatPage/ChatInput.tsx

+ import { ContextIndicator } from '@/components/ContextIndicator'

function ChatInput() {
    return (
        <div className="flex items-center gap-2 p-4">
            <input ... />
            <button>Send</button>
+           <ContextIndicator />
        </div>
    )
}
```

### 8. 事件监听注册

```tsx
// src/App.tsx 或 ChatPage 中

useEffect(() => {
    const unsubscribe = window.ipcRenderer.onContextUsage((payload) => {
        useAppStore.getState().setContextUsage(payload)
    })
    return unsubscribe
}, [])
```

---

## 测试策略

### Unit Tests

```typescript
// electron/services/chatToolLoop.test.ts

describe('chatToolLoop - Context Usage', () => {
    it('should call onContextUsage with usage data', async () => {
        const onContextUsage = vi.fn()
        const mockPipeline = {
            getUsage: () => ({ usedTokens: 5000, totalBudget: 10000, usagePercent: 0.5 }),
            shouldCompress: () => false,
        }
        
        await chatToolLoop({
            ...mockParams,
            compressionPipeline: mockPipeline,
            onContextUsage,
        })
        
        expect(onContextUsage).toHaveBeenCalledWith({
            usedTokens: 5000,
            totalBudget: 10000,
            usagePercent: 0.5,
            remaining: 5000,
            status: 'normal',
        })
    })
    
    it('should send "warning" status when usage > 60%', async () => {
        const onContextUsage = vi.fn()
        const mockPipeline = {
            getUsage: () => ({ usedTokens: 7000, totalBudget: 10000, usagePercent: 0.7 }),
            shouldCompress: () => false,
        }
        
        await chatToolLoop({ ...mockParams, compressionPipeline: mockPipeline, onContextUsage })
        
        expect(onContextUsage).toHaveBeenCalledWith(expect.objectContaining({ status: 'warning' }))
    })
    
    it('should send "critical" status when usage > 80%', async () => {
        // ...
    })
    
    it('should send "compressing" status during compression', async () => {
        // ...
    })
})
```

### 运行测试命令

```bash
cd crewagent-runtime
npm test -- electron/services/chatToolLoop.test.ts
```

### 手动测试

| 步骤 | 操作 | 预期结果 |
|------|------|---------|
| 1 | 打开 Chat 页面 | 指示器初始不显示 |
| 2 | 发送第一条消息 | 指示器出现，显示绿色 |
| 3 | 发送多条长消息 | 指示器逐渐变黄 |
| 4 | 继续直到 >80% | 指示器变红 |
| 5 | 触发压缩 | 指示器显示蓝色动画 |
| 6 | 压缩完成 | 指示器恢复绿色 |

---

## 依赖

- ✅ **Story 7-4**: Context Compression (done)
- ✅ **Story 7-5**: Chat Mode Integration (done)

---

## 实现顺序

1. `chatToolLoop.ts` - 添加 `onContextUsage` 回调
2. `main.ts` - 发送 IPC 事件
3. `preload.ts` - 暴露监听器
4. `appStore.ts` - 添加状态
5. `ContextIndicator.tsx` - 创建组件
6. ChatPage - 集成组件
7. 添加单元测试

---

## Related Artifacts
- Story 7-4: Context Compression (done)
- Story 7-5: Chat Mode Integration (done)
