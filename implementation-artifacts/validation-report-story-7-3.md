# Validation Report: Story 7-3

**Story**: 7-3 – Create Context Builder Module  
**Validated**: 2026-01-18  
**Status**: ✅ **APPROVED**

---

## 1. Code Review Summary

### 1.1 现有资产可复用

| 模块 | 位置 | 可复用程度 |
|------|------|-----------|
| `toOpenAIMessage` | `shared/conversationTypes.ts:160` | ✅ 100% |
| `PromptComposer` | `electron/services/promptComposer.ts` | ✅ 高度复用 |
| Base Rules 模板 | `electron/services/prompt-templates/` | ✅ 直接使用 |

### 1.2 PromptComposer 现有方法

```typescript
// promptComposer.ts
class PromptComposer {
  compose(input: ComposeInput): Message[]   // 主入口
  buildBaseRules(mode: SessionMode): string   // Chat/Run 模式规则
  buildToolPolicy(tools: AgentToolPolicy): string
  buildPersona(agent: AgentDefinition): string
  buildRunDirective(input: ComposeInput): string
  buildNodeBrief(...)
  buildStateSummary(...)
}
```

---

## 2. 架构分析

### 2.1 现有调用链

```
ExecutionEngine.run() 
    → PromptComposer.compose() 
    → chatToolLoop(messages=[...systemMessages, ...userMessages])
```

### 2.2 问题

1. **PromptComposer** 已实现大部分 System Prompt 逻辑，但未处理 `ConversationMessage[]` 转换
2. **Context Builder** 需要整合：
   - 从持久化的 `ConversationMessage[]` 转换
   - 调用 PromptComposer 生成 System Prompt
   - 支持 `includeInContext` 过滤

---

## 3. 实现方案

### Option A: 新建 context-builder 模块（推荐）

```
electron/core/context-builder/
├── index.ts              # 导出 buildLlmMessagesFromConversation
├── types.ts              # ContextBuildOptions, ContextBuildResult
├── buildLlmMessages.ts   # 核心转换逻辑
└── context-builder.test.ts
```

**System Prompt 处理**：调用现有 `PromptComposer.compose()` 的各方法

### Option B: 扩展 PromptComposer

直接在 PromptComposer 中添加 `buildFromConversation()` 方法。

**缺点**：PromptComposer 已有 427 行，进一步扩展会过于臃肿

**推荐**：Option A - 新建模块，依赖但不修改 PromptComposer

---

## 4. Feasibility Assessment

| 改造点 | 复杂度 | 可行性 |
|--------|--------|--------|
| 消息转换 | Low | ✅ 复用 `toOpenAIMessage` |
| System Prompt | Low | ✅ 复用 PromptComposer 方法 |
| includeInContext 过滤 | Low | ✅ 简单 filter |
| 单元测试 | Low | ✅ 纯函数易测 |

---

## 5. 依赖检查

- ✅ `toOpenAIMessage` 已实现并测试（Story 7-1）
- ✅ `ConversationMessage` 已扩展完成（Story 7-1）
- ✅ `PromptComposer` 可独立调用
- ✅ 无循环依赖风险

---

## 6. Verdict

✅ **Story 7-3 可以开始实现**

- 大量现有代码可复用
- 架构清晰
- 实现复杂度低

---

## Implementation Notes

1. **不修改** `PromptComposer`，只从 context-builder 调用其方法
2. 使用 `toOpenAIMessage` 进行消息转换
3. System Prompt 不存入 messages.json，运行时动态生成
