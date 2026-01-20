# Validation Report: Story 7-1

**Story**: 7-1 – Extend ConversationMessage Data Model (Protocol-Complete)  
**Validated**: 2026-01-18  
**Status**: ✅ **APPROVED**

---

## 1. Code Review Summary

### 1.1 Current Type Definitions

| 位置 | 类型 | 状态 |
|------|------|------|
| `src/stores/appStore.ts:154-164` | `ConversationMessage` | ✅ 存在 |
| `src/stores/appStore.ts:11` | `ConversationMessageRole` | ✅ 存在 |
| `electron/stores/runtimeStore.ts:606-616` | `ConversationMessage` | ✅ 存在（重复定义）|

**发现**：`ConversationMessage` 在两处重复定义，需要统一到 `shared/` 目录。

### 1.2 shared/ 目录现状

```
shared/
├── agentToolPolicy.ts
└── logTypes.ts
```

✅ 已有 shared 目录和类型共享先例，可新建 `conversationTypes.ts`。

### 1.3 类型引用分析

`ConversationMessage` 被以下文件引用：

| 文件 | 引用方式 |
|------|---------|
| `WorksPage.tsx` | import from appStore |
| `RunWorkspace.tsx` | import from appStore |
| `ChatPanel.tsx` | import from appStore |
| `MessageList.tsx` | import from appStore |
| `MessageItem.tsx` | import from appStore |
| `MessageItem.test.tsx` | import from appStore |

✅ 所有前端组件都从 `appStore` 导入，统一修改导入源即可。

---

## 2. Feasibility Assessment

### 2.1 类型扩展

| 新增字段 | 类型 | 可行性 |
|---------|------|--------|
| `tool_calls` | `OpenAIToolCall[]` | ✅ 可选字段，不影响现有消息 |
| `tool_call_id` | `string` | ✅ 可选字段 |
| `mode` | `'chat' \| 'agent' \| 'run'` | ✅ 可选字段 |
| `runId` | `string` | ✅ 可选字段 |
| `includeInContext` | `boolean` | ✅ 可选字段，默认 true |

### 2.2 向后兼容性

- ✅ 所有新字段都是**可选的**
- ✅ `content` 改为 `string | null` 不影响现有逻辑（TypeScript union 兼容）
- ✅ 旧 `messages.json` 加载时缺少字段会是 `undefined`，符合预期

### 2.3 Role 扩展

```typescript
// Before
type ConversationMessageRole = 'user' | 'assistant' | 'system'

// After
type ConversationMessageRole = 'user' | 'assistant' | 'system' | 'tool'
```

⚠️ **注意点**：UI 渲染逻辑需要处理 `role='tool'` 的情况。检查 `MessageItem.tsx`：

- 当前基于 `role` 判断消息样式（用户消息右侧，assistant 左侧）
- `tool` 消息建议与 `assistant` 同侧显示，或合并到工具调用组中折叠

---

## 3. Risk Assessment

| 风险 | 等级 | 缓解措施 |
|------|------|---------|
| UI 渲染 `role=tool` | Low | 复用 `assistant` 样式，或默认折叠 |
| 类型导入路径变更 | Low | 批量替换 + TypeScript 编译检查 |
| 旧消息兼容 | Low | 所有新字段可选 |

---

## 4. Implementation Checklist

- [ ] 创建 `shared/conversationTypes.ts`
- [ ] 定义 `OpenAIToolCall` 接口
- [ ] 扩展 `ConversationMessageRole` 添加 `'tool'`
- [ ] 扩展 `ConversationMessage` 添加新字段
- [ ] `appStore.ts` 改为从 shared 导入
- [ ] `runtimeStore.ts` 改为从 shared 导入
- [ ] 验证 TypeScript 编译通过
- [ ] 验证现有消息正常加载

---

## 5. Verdict

✅ **Story 7-1 可以开始实现**

- 技术方案可行
- 向后兼容
- 影响范围可控
- `shared/` 目录已有先例

---

## Related Files
- `crewagent-runtime/src/stores/appStore.ts`
- `crewagent-runtime/electron/stores/runtimeStore.ts`
- `crewagent-runtime/shared/logTypes.ts`（参考）
