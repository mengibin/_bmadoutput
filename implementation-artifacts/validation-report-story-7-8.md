# Validation Report: Story 7-8

**Story**: 7-8 – Add Context Usage UI Indicator  
**Validated**: 2026-01-19  
**Status**: ✅ **APPROVED**

---

## 1. 现有模式分析

### 1.1 IPC 事件发送模式

```typescript
// main.ts:488-493
const sendIpcEvent = (target, channel, payload) => target.webContents.send(channel, payload)

onStart: (payload) => sendIpcEvent(target, 'llm:stream-start', payload),
onChunk: (payload) => sendIpcEvent(target, 'llm:stream-chunk', payload),
onEnd: (payload) => sendIpcEvent(target, 'llm:stream-end', payload),
onToolStart: (payload) => sendIpcEvent(target, 'llm:stream-tool-start', payload),
onToolEnd: (payload) => sendIpcEvent(target, 'llm:stream-tool-end', payload),
```

**可复用**：直接使用相同模式添加 `context:usage` 事件。

### 1.2 preload.ts 监听模式

```typescript
// preload.ts:84-110
onLlmStreamStart: (listener) => {
    ipcRenderer.on('llm:stream-start', listener)
    return () => ipcRenderer.removeListener('llm:stream-start', listener)
},
```

**可复用**：添加 `onContextUsage` 方法。

### 1.3 现有事件类型

| 事件 | 用途 |
|------|------|
| `llm:stream-*` | LLM 流式响应 |
| `widget:request` | Widget 请求 |
| `mcp:installProgress` | MCP 安装进度 |
| `mcp:statusChange` | MCP 状态变化 |

**结论**：已有成熟的 IPC 事件模式，可直接复用。

---

## 2. 实现可行性

### 2.1 Backend 改动

| 位置 | 改动 | 复杂度 |
|------|------|--------|
| `main.ts` | 添加 `sendIpcEvent(target, 'context:usage', payload)` | Low |
| `chatToolLoop.ts` | 在 loop 结束后调用回调 | Low |
| `preload.ts` | 添加 `onContextUsage` 方法 | Low |

### 2.2 Frontend 改动

| 位置 | 改动 | 复杂度 |
|------|------|--------|
| `appStore.ts` | 添加 `contextUsage` 状态 | Low |
| 新建 `ContextIndicator.tsx` | 指示器组件 | Medium |
| `ChatPage` | 集成组件 | Low |

---

## 3. 依赖检查

- ✅ **Story 7-4**: Context Compression (done) - 提供 `getUsage()` 方法
- ✅ **Story 7-5**: Chat Mode Integration (done) - CompressionPipeline 已集成
- ✅ IPC 基础设施已存在

---

## 4. Verdict

✅ **Story 7-8 可以开始实现**

**实现顺序**：
1. 在 `chatToolLoop` 添加 `onContextUsage` 回调参数
2. 在 `main.ts` 发送 `context:usage` 事件
3. 在 `preload.ts` 添加监听器
4. 在 `appStore.ts` 添加状态
5. 创建 `ContextIndicator.tsx` 组件
6. 集成到 ChatPage

---

## 5. 实现 Checklist

- [ ] 添加 `onContextUsage` 回调到 chatToolLoop
- [ ] 在 main.ts 发送 IPC 事件
- [ ] 更新 preload.ts 暴露监听器
- [ ] 更新 appStore.ts 添加状态
- [ ] 创建 ContextIndicator 组件
- [ ] 集成到 ChatPage
- [ ] 添加单元测试
