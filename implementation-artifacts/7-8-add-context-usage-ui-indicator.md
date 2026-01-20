# Story 7-8: Add Context Usage UI Indicator (Optional)

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 4 (optional)  
**Status**: `done`

## Goal
在 UI 中显示上下文剩余量指示器，类似 Claude Code。让用户直观了解当前对话消耗的 token 预算。

## Business Value
- **透明度**：用户知道还能发多少消息
- **预期管理**：提前提示即将压缩
- **用户体验**：类似 Claude Code 的专业体验

## Acceptance Criteria
1. **IPC 事件**：后端发送 `context:usage` 事件
2. **状态显示**：
   - `normal`：< 60%（绿色）
   - `warning`：60-80%（黄色）
   - `compressing`：正在压缩（动画）
   - `critical`：> 80%（红色）
3. **UI 位置**：显示在聊天输入框附近
4. **数据更新**：每次 LLM 调用后更新

## Out of Scope
- 历史压缩详情展示
- 手动触发压缩

## Dependencies
- **Story 7-4**: Context Compression Module (done)
- **Story 7-5**: Chat Mode Integration (done)

---

## Technical Design

### IPC Event

```typescript
// 事件类型
interface ContextUsageEvent {
    usedTokens: number
    totalBudget: number
    usagePercent: number
    status: 'normal' | 'warning' | 'critical' | 'compressing'
}

// 发送事件
mainWindow.webContents.send('context:usage', {
    usedTokens: usage.usedTokens,
    totalBudget: usage.totalBudget,
    usagePercent: usage.usagePercent,
    status: getUsageStatus(usage.usagePercent),
})
```

### 状态计算

```typescript
function getUsageStatus(percent: number, isCompressing: boolean): string {
    if (isCompressing) return 'compressing'
    if (percent > 0.8) return 'critical'
    if (percent > 0.6) return 'warning'
    return 'normal'
}
```

### 触发时机

1. **每次 LLM 调用后**：在 `chatToolLoop` 结束后发送
2. **压缩开始/结束**：发送 `compressing` 状态
3. **新对话开始**：重置为 `normal`

---

## UI Design

### 指示器位置

```
┌─────────────────────────────────────────┐
│                                         │
│       [Chat Messages Area]              │
│                                         │
├─────────────────────────────────────────┤
│  [Input Box]              [●●●○○] 45%   │
│                                         │
│         [Send Button]                   │
└─────────────────────────────────────────┘
```

### 视觉样式

| 状态 | 颜色 | 图标 |
|------|------|------|
| normal | 绿色 | ●●○○○ |
| warning | 黄色 | ●●●○○ |
| critical | 红色 | ●●●●○ |
| compressing | 蓝色动画 | ⟳ |

---

## Files to Modify

### Backend
| File | Changes |
|------|---------|
| `electron/main.ts` | 添加 IPC 事件发送逻辑 |
| `electron/services/chatToolLoop.ts` | 在 loop 结束后发送使用情况 |
| `electron/preload.ts` | 暴露 `context:usage` 监听器 |

### Frontend
| File | Changes |
|------|---------|
| `src/stores/appStore.ts` | 添加 contextUsage 状态 |
| `src/components/ChatInput/ContextIndicator.tsx` | 新建指示器组件 |
| `src/pages/ChatPage/` | 集成指示器组件 |

---

## Testing Strategy

### Unit Tests
```typescript
describe('getUsageStatus', () => {
    it('returns normal for < 60%', () => {
        expect(getUsageStatus(0.5, false)).toBe('normal')
    })
    it('returns warning for 60-80%', () => {
        expect(getUsageStatus(0.7, false)).toBe('warning')
    })
    it('returns critical for > 80%', () => {
        expect(getUsageStatus(0.85, false)).toBe('critical')
    })
    it('returns compressing when isCompressing is true', () => {
        expect(getUsageStatus(0.5, true)).toBe('compressing')
    })
})
```

### Manual Testing
1. 打开 Chat 页面
2. 发送消息，观察指示器变化
3. 发送多轮对话直到触发 warning
4. 继续直到触发压缩，观察动画

---

## Related Artifacts
- **Design Document**: [design-7-8-add-context-usage-ui-indicator.md](./design-7-8-add-context-usage-ui-indicator.md)
- **Validation Report**: [validation-report-story-7-8.md](./validation-report-story-7-8.md)
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-4: Context Compression (done)
- Story 7-5: Chat Mode Integration (done)
