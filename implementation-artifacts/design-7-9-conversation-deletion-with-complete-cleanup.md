# Design: Conversation Deletion with Complete Cleanup

**Story:** `7-9-conversation-deletion-with-complete-cleanup.md`  
**设计原则:** 最小改动、原子操作、向后兼容

---

## 设计目标

1. **完整清理**：删除对话时同时清理工具输出文件
2. **原子操作**：先删除文件再更新索引
3. **健壮性**：文件不存在时不报错

---

## 改动概览

| 位置 | 改动 |
|------|------|
| `runtimeStore.ts:deleteConversation` | 添加清理 `tool-results` 目录逻辑 |

---

## 详细实现

### deleteConversation 改动

```diff
// electron/stores/runtimeStore.ts

public deleteConversation(projectRoot: string, conversationId: string): { success: boolean } {
    this.ensureProjectRuntimeDirs(projectRoot)
    const conversationDir = this.getConversationDir(projectRoot, conversationId)
    
+   // 1. 删除工具输出文件
+   const stateRoot = this.getProjectRuntimeRoot(projectRoot)
+   const toolResultsDir = path.join(stateRoot, 'logs', 'tool-results', conversationId)
+   try {
+       if (fs.existsSync(toolResultsDir)) {
+           fs.rmSync(toolResultsDir, { recursive: true, force: true })
+       }
+   } catch (error) {
+       console.error('Failed to remove tool-results directory:', error)
+       // 继续删除，不阻塞
+   }
    
-   try {
+   // 2. 删除 conversation 目录
+   try {
        if (fs.existsSync(conversationDir)) {
            fs.rmSync(conversationDir, { recursive: true, force: true })
        }
    } catch (error) {
        console.error('Failed to remove conversation directory:', error)
        return { success: false }
    }

+   // 3. 更新 index
    const existing = this.loadConversations(projectRoot)
    const next = existing.filter((item) => item.id !== conversationId)
    this.saveConversationsIndex(projectRoot, next)
    return { success: true }
}
```

---

## 删除顺序

```
1. tool-results/<conversationId>/  ← 先删除外部文件
2. conversations/<conversationId>/ ← 再删除对话目录
3. index.json                      ← 最后更新索引
```

**原因**：如果 index 先更新但文件删除失败，会留下孤立文件。反之，文件删除成功但 index 更新失败，下次可以重试。

---

## 测试策略

### Unit Tests

```typescript
// electron/stores/runtimeStore.test.ts

describe('deleteConversation with tool-results cleanup', () => {
    it('should delete tool-results directory when it exists', () => {
        const projectRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'conv-delete-test-'))
        const conversationId = 'conv-1'
        
        // 创建测试文件
        const stateRoot = store.getProjectRuntimeRoot(projectRoot)
        const toolResultsDir = path.join(stateRoot, 'logs', 'tool-results', conversationId)
        fs.mkdirSync(toolResultsDir, { recursive: true })
        fs.writeFileSync(path.join(toolResultsDir, 'test.json'), '{}')
        
        // 创建对话
        store.appendConversationMessage(projectRoot, conversationId, {
            id: 'msg-1',
            role: 'user',
            content: 'test',
            createdAt: new Date().toISOString(),
        })
        
        // 删除
        const result = store.deleteConversation(projectRoot, conversationId)
        
        expect(result.success).toBe(true)
        expect(fs.existsSync(toolResultsDir)).toBe(false)
        
        fs.rmSync(projectRoot, { recursive: true, force: true })
    })
    
    it('should not fail if tool-results directory does not exist', () => {
        const projectRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'conv-delete-test-'))
        const conversationId = 'conv-2'
        
        // 创建对话（无工具输出）
        store.appendConversationMessage(projectRoot, conversationId, {
            id: 'msg-1',
            role: 'user',
            content: 'test',
            createdAt: new Date().toISOString(),
        })
        
        // 删除不应报错
        const result = store.deleteConversation(projectRoot, conversationId)
        
        expect(result.success).toBe(true)
        
        fs.rmSync(projectRoot, { recursive: true, force: true })
    })
})
```

### 运行测试命令

```bash
cd crewagent-runtime
npm test -- electron/stores/runtimeStore.test.ts
```

### Manual Testing
1. 创建对话，触发工具调用
2. 验证 `tool-results/<conversationId>/` 目录存在
3. 删除对话
4. 验证目录已被清理

---

## 依赖

- ✅ **Story 7-2**: Persist Tool-Call Protocol Messages (done)

---

## Related Artifacts
- **Story Document**: [7-9-conversation-deletion-with-complete-cleanup.md](./7-9-conversation-deletion-with-complete-cleanup.md)
- **Validation Report**: [validation-report-story-7-9.md](./validation-report-story-7-9.md)
