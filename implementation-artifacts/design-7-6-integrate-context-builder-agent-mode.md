# Design: Integrate Context Builder into Agent Mode

**Story:** `7-6-integrate-context-builder-agent-mode.md`  
**设计原则:** 验证优先、共用代码、最小改动

---

## 设计目标

1. **验证 Agent 模式**：确认 Story 7-5 的改动对 Agent 模式也生效
2. **Persona 注入**：验证 System Prompt 包含 Agent Persona
3. **模式标识**：验证消息带 `mode='agent'` 标记

---

## 架构分析

### 共用代码路径

```
┌─────────────────────────────────────────────────────────────────┐
│ chat:send IPC                        agent:dispatch IPC         │
│     ↓                                      ↓                    │
│     └──────────────┬───────────────────────┘                    │
│                    ↓                                            │
│              callAgentChat                                      │
│                    │                                            │
│     ┌──────────────┼──────────────┐                             │
│     ↓              ↓              ↓                             │
│  mode='chat'    mode='agent'   mode='agent'                     │
│  (chat:send)  (agent:dispatch) (ExecScript/RunAction)           │
│                    │                                            │
│                    ↓                                            │
│  ┌─────────────────────────────────────────┐                    │
│  │ buildLlmMessagesFromConversation        │                    │
│  │   - mode='agent' → 包含 Persona         │                    │
│  │   - mode='chat'  → 不包含 Persona       │                    │
│  └─────────────────────────────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

### Persona 处理逻辑

```typescript
// systemPromptComposer.ts:16-18
const agent = options.agentDefinition ?? options.agent
const persona = agent ? buildAgentPersona(agent) : ''
if (persona && options.mode !== 'chat') parts.push(persona)
```

**关键**：`mode !== 'chat'` 意味着 `mode='agent'` 时自动包含 Persona

---

## 实现方案

### 无需代码改动

Story 7-5 完成后，Agent 模式已自动获得：
- ✅ 对话历史加载
- ✅ Context Builder 上下文构建
- ✅ Persona 注入（`mode='agent'`时）
- ✅ 消息持久化（带 `mode='agent'` 标记）
- ✅ 压缩支持

### 验证工作

本 Story 的主要工作是**验证**，通过单元测试和手动测试确认功能正常。

---

## 测试策略

### Unit Tests

#### 1. systemPromptComposer Agent Persona 测试

```typescript
// File: electron/core/context-builder/context-builder.test.ts

describe('composeSystemPrompt - Agent Mode', () => {
    it('should include agent persona when mode is agent', () => {
        const result = composeSystemPrompt({
            messages: [],
            mode: 'agent',
            agent: {
                id: 'test-agent',
                name: 'Test Agent',
                role: 'Assistant',
                identity: 'A helpful AI assistant',
                principles: ['Be helpful', 'Be accurate'],
            },
        })
        
        expect(result).toContain('## Agent Persona')
        expect(result).toContain('**Name:** Test Agent')
        expect(result).toContain('**Role:** Assistant')
        expect(result).toContain('**Identity:** A helpful AI assistant')
        expect(result).toContain('- Be helpful')
        expect(result).toContain('- Be accurate')
    })
    
    it('should NOT include agent persona when mode is chat', () => {
        const result = composeSystemPrompt({
            messages: [],
            mode: 'chat',
            agent: {
                id: 'test-agent',
                name: 'Test Agent',
                role: 'Assistant',
            },
        })
        
        expect(result).not.toContain('## Agent Persona')
    })
    
    it('should use systemPrompt override when provided', () => {
        const customPrompt = 'Custom system prompt for this agent'
        const result = composeSystemPrompt({
            messages: [],
            mode: 'agent',
            agent: {
                id: 'test-agent',
                name: 'Test Agent',
                role: 'Assistant',
                systemPrompt: customPrompt,
            },
        })
        
        expect(result).toContain(customPrompt)
        expect(result).not.toContain('## Agent Persona')
    })
    
    it('should include mode banner for agent mode', () => {
        const result = composeSystemPrompt({
            messages: [],
            mode: 'agent',
            agent: {
                id: 'test-agent',
                name: 'Test Agent',
                role: 'Assistant',
            },
        })
        
        expect(result).toContain('# Mode: AGENT')
    })
})
```

#### 2. buildLlmMessagesFromConversation Agent Mode 测试

```typescript
describe('buildLlmMessagesFromConversation - Agent Mode', () => {
    it('should build messages with agent persona in system prompt', () => {
        const history: ConversationMessage[] = [
            { id: '1', role: 'user', content: 'Hello', createdAt: '2024-01-01T00:00:00Z', mode: 'agent' },
            { id: '2', role: 'assistant', content: 'Hi there!', createdAt: '2024-01-01T00:00:01Z', mode: 'agent' },
        ]
        
        const result = buildLlmMessagesFromConversation({
            messages: history,
            mode: 'agent',
            agent: {
                id: 'test-agent',
                name: 'Test Agent',
                role: 'Friendly Assistant',
            },
        })
        
        // First message should be system prompt with persona
        expect(result.messages[0].role).toBe('system')
        expect(result.messages[0].content).toContain('## Agent Persona')
        expect(result.messages[0].content).toContain('**Role:** Friendly Assistant')
        
        // History should follow
        expect(result.messages[1]).toEqual({ role: 'user', content: 'Hello' })
        expect(result.messages[2]).toEqual({ role: 'assistant', content: 'Hi there!' })
    })
    
    it('should preserve mode=agent in metadata', () => {
        const result = buildLlmMessagesFromConversation({
            messages: [],
            mode: 'agent',
            agent: { id: 'a', name: 'A', role: 'R' },
        })
        
        expect(result.metadata.systemPromptIncluded).toBe(true)
    })
})
```

#### 3. 消息持久化 mode 字段测试

```typescript
// File: electron/stores/runtimeStore.test.ts

describe('appendConversationMessage - Agent Mode', () => {
    it('should persist message with mode=agent', () => {
        const projectRoot = fs.mkdtempSync(path.join(os.tmpdir(), 'agent-mode-test-'))
        const conversationId = 'conv-agent-1'
        
        const message: ConversationMessage = {
            id: 'msg-1',
            role: 'user',
            content: 'Hello from agent mode',
            createdAt: new Date().toISOString(),
            mode: 'agent',
            agentId: 'test-agent',
        }
        
        store.appendConversationMessage(projectRoot, conversationId, message)
        const loaded = store.loadConversationMessages(projectRoot, conversationId)
        
        expect(loaded).toHaveLength(1)
        expect(loaded[0].mode).toBe('agent')
        expect(loaded[0].agentId).toBe('test-agent')
        
        fs.rmSync(projectRoot, { recursive: true, force: true })
    })
})
```

### Integration Tests (可选)

```typescript
describe('Agent Mode Integration', () => {
    it('should maintain conversation history across multiple turns', async () => {
        // Setup mock LLM adapter
        // Call callAgentChat twice with same conversationId
        // Verify second call includes history from first call
    })
    
    it('should apply compression when context exceeds threshold', async () => {
        // Create large conversation history
        // Verify compression is applied
        // Verify critical messages are preserved
    })
})
```

### Manual Testing Checklist

| 测试项 | 验证方法 | 预期结果 |
|--------|---------|---------|
| Agent Persona 注入 | 查看 LLM 日志中的 system prompt | 包含 `## Agent Persona` |
| 多轮对话 | 发送 2+ 条消息，验证 LLM 引用历史 | LLM 能引用之前的内容 |
| mode 标记 | 检查 messages.json | `mode: 'agent'` |
| 历史加载 | 重启应用后发消息 | LLM 能看到之前的对话 |

---

## 验证命令

```bash
# 运行 Agent 模式相关测试
cd crewagent-runtime && npm test -- electron/core/context-builder/context-builder.test.ts

# 查看消息日志中的 mode 字段
cat <projectRoot>/RuntimeStore/state/conversations/<conversationId>/messages.json | jq '.[] | {id, role, mode, agentId}'
```

---

## Files to Verify

| File | 验证点 |
|------|--------|
| `electron/core/context-builder/systemPromptComposer.ts` | Persona 构建逻辑 |
| `electron/core/context-builder/context-builder.test.ts` | 添加 Agent 模式测试 |
| `electron/main.ts` | callAgentChat 正确传递 mode 和 agent |

---

## 依赖

- ✅ **Story 7-5**: Integrate Context Builder into Chat Mode (done)

---

## Related Artifacts
- Story 7-5: Chat Mode Integration (done)
- Story 7-7: Run Mode Integration (next)
