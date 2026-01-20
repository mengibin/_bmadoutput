# Validation Report: Story 7-7

**Story**: 7-7 – Integrate Context Builder into Run Mode  
**Validated**: 2026-01-19  
**Status**: ✅ **APPROVED**

---

## 1. 代码分析

### 1.1 session.history 使用点

| Line | Usage | Description |
|------|-------|-------------|
| 100 | 类型定义 | `history: OpenAIChatMessage[]` |
| 303 | 初始化 | `{ history: [], ... }` |
| 528 | 写入 | `session.history.push({ role: 'user', ... })` |
| 618 | 读取 | `[...toOpenAiMessages(baseMessages), ...session.history]` |
| 660 | 写入 | `session.history.push({ role: 'tool', ... })` |
| 686 | 写入 | `session.history.push(assistant)` |
| 801 | 写入 | `session.history.push({ role: 'tool', ... })` |
| 824 | 写入 | `session.history.push(assistantMessage)` |

**总计**: 8 处使用点（1 定义 + 1 初始化 + 5 写入 + 1 读取）

### 1.2 trimHistory 调用点

| Line | Context |
|------|---------|
| 529 | 用户输入后 |
| 669 | 工具结果后 |
| 687 | assistant 消息后 |
| 806 | 工具结果后 |
| 825 | assistant 消息后 |

**总计**: 5 次调用，全部跟在 `session.history.push()` 之后

---

## 2. 改动规模评估

| 改动项 | 复杂度 | 说明 |
|--------|--------|------|
| 删除 `RunSession.history` | Low | 删除类型定义和初始化 |
| 删除 `trimHistory` 方法 | Low | 删除方法和 5 处调用 |
| 读取改为 `loadConversationMessages` | Medium | 修改 L618 |
| 写入改为 `appendConversationMessage` | Medium | 修改 5 处 push |
| 集成 context-builder | Medium | 替换 promptComposer 调用 |
| 集成 CompressionPipeline | Low | 复用 Story 7-5 模式 |

**总体评估**: Medium 复杂度，约 80-120 行代码改动

---

## 3. 关键改动点

### 3.1 RunSession 类型修改

```diff
interface RunSession {
-   history: OpenAIChatMessage[]
+   conversationId: string  // 关联的对话 ID
    lastFailFingerprint: string | null
    consecutiveFailCount: number
    streamSequence: number
}
```

### 3.2 消息读取 (L618)

```diff
- const requestMessages: OpenAIChatMessage[] = [...toOpenAiMessages(baseMessages), ...session.history]
+ const history = this.runtimeStore.loadConversationMessages(params.projectRoot, session.conversationId)
+ const { messages } = buildLlmMessagesFromConversation({
+     messages: history,
+     mode: 'run',
+     runContext: { ... },
+ })
```

### 3.3 消息写入 (5 处)

```diff
- session.history.push({ role: 'user', content: ... })
- this.trimHistory(session.history)
+ this.runtimeStore.appendConversationMessage(params.projectRoot, session.conversationId, {
+     id: randomUUID(),
+     role: 'user',
+     content: ...,
+     mode: 'run',
+     runId: params.runId,
+ })
```

---

## 4. 依赖检查

- ✅ **Story 7-3**: Context Builder (done)
- ✅ **Story 7-4**: Context Compression (done)
- ✅ **Story 7-5**: Chat Mode Integration (done)
- ✅ `loadConversationMessages` 已存在
- ✅ `appendConversationMessage` 已存在

---

## 5. 风险分析

| 风险 | 级别 | 缓解措施 |
|------|------|----------|
| 性能下降（频繁读写 JSON） | Medium | 考虑内存缓存 + 增量写入 |
| 改动范围大 | Medium | 分步重构，逐步测试 |
| Run 恢复逻辑复杂 | Medium | 先确保基本功能，恢复作为增强 |

---

## 6. Verdict

✅ **Story 7-7 可以开始实现**

**建议实现顺序**：
1. 添加 `conversationId` 到 RunSession
2. 将消息写入改为 `appendConversationMessage`
3. 将消息读取改为 `loadConversationMessages` + context-builder
4. 删除 `trimHistory` 方法
5. 集成 `CompressionPipeline`
6. 测试 Run 恢复功能

---

## 7. 实现 Checklist

- [ ] 修改 `RunSession` 类型，添加 `conversationId`
- [ ] 删除 `RunSession.history`
- [ ] 删除 `trimHistory` 方法和 5 处调用
- [ ] 修改 L618 消息读取逻辑
- [ ] 修改 5 处消息写入为 `appendConversationMessage`
- [ ] 集成 `buildLlmMessagesFromConversation({ mode: 'run', ... })`
- [ ] 集成 `CompressionPipeline`
- [ ] 添加单元测试
- [ ] 测试 Run 恢复功能
