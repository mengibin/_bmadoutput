# Story 7-9: Conversation Deletion with Complete Cleanup

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 4  
**Status**: `done`

## Goal
扩展 Conversation 删除功能，确保删除时清理所有相关数据（扩展后的消息、工具输出文件等）。

## Business Value
- **数据完整性**：不留下孤立的工具输出文件
- **磁盘空间**：及时清理不再需要的数据
- **用户信任**：用户知道删除是彻底的

## Acceptance Criteria
1. **消息清理**：删除 `<conversationId>/messages.json`（含新增的 `role=tool` 消息）
2. **工具输出清理**：删除 `@state/logs/tool-results/<conversationId>/` 下的所有文件
3. **元数据清理**：从 `index.json` 中移除 conversation
4. **原子操作**：先删除文件再更新 index，确保不会留下孤立引用
5. **UI 确认**：删除前提示将清理所有聊天记录和工具输出

## Out of Scope
- 批量删除多个 Conversation
- 回收站功能（软删除）

## Dependencies
- **Story 7-2**: Persist Tool-Call Protocol Messages (done) - 引入了工具输出文件

---

## Technical Context

### 当前删除行为

```typescript
// runtimeStore.ts
async deleteConversation(projectRoot: string, conversationId: string) {
    // 仅删除 conversation 目录
    // 未清理 tool-results 目录
}
```

### 目标删除行为

```typescript
async function deleteConversation(projectRoot: string, conversationId: string) {
    // 1. 删除 messages.json
    await fs.rm(getMessagesPath(projectRoot, conversationId), { force: true })
    
    // 2. 删除工具输出文件
    await fs.rm(getToolResultsPath(projectRoot, conversationId), { 
        recursive: true, 
        force: true 
    })
    
    // 3. 删除 conversation 目录
    await fs.rm(getConversationDir(projectRoot, conversationId), { 
        recursive: true,
        force: true
    })
    
    // 4. 更新 index.json
    await updateIndex(projectRoot, index => 
        index.filter(c => c.id !== conversationId)
    )
}
```

### 路径说明

| 路径 | 说明 |
|------|------|
| `<projectRoot>/RuntimeStore/state/conversations/<conversationId>/` | Conversation 目录 |
| `<projectRoot>/RuntimeStore/state/conversations/<conversationId>/messages.json` | 消息文件 |
| `<projectRoot>/RuntimeStore/state/logs/tool-results/<conversationId>/` | 工具输出目录 |
| `<projectRoot>/RuntimeStore/state/conversations/index.json` | 索引文件 |

---

## Files to Modify

| File | Changes |
|------|---------|
| `electron/stores/runtimeStore.ts` | 扩展 `deleteConversation` 方法 |

---

## UI 确认对话框

```typescript
// 删除前确认
const confirmed = await dialog.confirm({
    title: '删除对话',
    message: `确定要删除对话 "${conversationTitle}" 吗？`,
    detail: '此操作将删除所有聊天记录和相关的工具输出文件，无法恢复。',
    okLabel: '删除',
    cancelLabel: '取消',
})
```

---

## Testing Strategy

### Unit Tests
```typescript
describe('deleteConversation', () => {
    it('should delete messages.json', async () => {
        // Create conversation with messages
        // Delete conversation
        // Verify messages.json is gone
    })
    
    it('should delete tool-results directory if exists', async () => {
        // Create conversation with tool outputs
        // Delete conversation
        // Verify tool-results/<conversationId>/ is gone
    })
    
    it('should remove from index.json', async () => {
        // Create conversation
        // Delete conversation
        // Verify not in index
    })
    
    it('should not throw if tool-results directory does not exist', async () => {
        // Create conversation without tool outputs
        // Delete conversation should not throw
    })
})
```

### Manual Testing
1. 创建对话，发送消息
2. 触发工具调用（如 fs.read）
3. 验证 tool-results 目录存在
4. 删除对话
5. 验证 conversation 目录和 tool-results 目录都已删除

---

## Related Artifacts
- **Design Document**: [design-7-9-conversation-deletion-with-complete-cleanup.md](./design-7-9-conversation-deletion-with-complete-cleanup.md)
- **Validation Report**: [validation-report-story-7-9.md](./validation-report-story-7-9.md)
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-2: Persist Tool-Call Protocol Messages (done)
