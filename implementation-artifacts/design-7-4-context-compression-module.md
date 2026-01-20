# Design: Context Compression Module (松耦合 + 高内聚)

**Story:** `7-4-create-context-compression-module.md`  
**设计原则:** 松耦合、高内聚、策略模式

---

## 设计目标

1. **高内聚**：模块内所有组件服务于单一目标 - 上下文压缩
2. **松耦合**：不依赖外部实现细节，自包含 token 估算
3. **可扩展**：策略模式支持未来升级（LLM 摘要、向量检索）

---

## 依赖关系图

```
                    ┌─────────────────────────────────┐
                    │  shared/conversationTypes.ts     │
                    │  (ConversationMessage 类型)      │
                    └─────────────────────────────────┘
                                    ▲
                                    │ 只依赖类型
                    ┌─────────────────────────────────┐
                    │ electron/core/context-compression│
                    │  - 内部实现 token 估算           │
                    │  - 不依赖 chatToolLoop          │
                    │  - 不依赖 context-builder       │
                    └─────────────────────────────────┘
                                    ▲
                   ┌────────────────┼────────────────┐
                   │                │                │
                context-builder  ExecutionEngine   ChatHandler
                    (可选调用)
```

---

## 模块结构

```
electron/core/context-compression/
├── index.ts                           # Public API 导出
├── types.ts                           # 独立类型定义
├── tokenEstimator.ts                  # Token 估算 (内部实现)
├── ContextUsageMonitor.ts             # 使用量监控
├── CompressionPipeline.ts             # 压缩管道
├── strategies/
│   ├── CompressionStrategy.ts         # 策略接口
│   └── KeyMessageExtractionStrategy.ts # v1 策略
└── context-compression.test.ts        # 单元测试
```

---

## 接口设计 (types.ts)

```typescript
/**
 * Context Compression 独立类型定义
 * 设计原则: 不依赖外部模块的实现类型
 */

import type { ConversationMessage } from '../../../shared/conversationTypes'

// ============================================================================
// Context Usage (上下文使用情况)
// ============================================================================

export interface ContextUsage {
    /** 已使用的 token 数 */
    usedTokens: number
    
    /** 总预算 */
    totalBudget: number
    
    /** 使用百分比 (0-1) */
    usagePercent: number
    
    /** 剩余 token 数 */
    remaining: number
}

// ============================================================================
// Compression Config (压缩配置)
// ============================================================================

export interface CompressionConfig {
    /** 触发压缩的阈值 (0-1)，default: 0.8 */
    triggerThreshold: number
    
    /** 压缩后的目标使用率 (0-1)，default: 0.5 */
    targetUsage: number
    
    /** 最少保留的最近消息数，default: 10 */
    minRecentMessages: number
    
    /** 模型的 token 预算，default: 128000 */
    tokenBudget: number
}

// ============================================================================
// Message Priority (消息优先级)
// ============================================================================

export interface ScoredMessage {
    message: ConversationMessage
    index: number
    priority: number
    tokens: number
    isPartOfToolGroup: boolean
    toolGroupId?: string
}

// ============================================================================
// Compression Result (压缩结果)
// ============================================================================

export interface CompressionResult {
    /** 压缩后的消息 */
    messages: ConversationMessage[]
    
    /** 是否执行了压缩 */
    compressed: boolean
    
    /** 压缩元数据 */
    metadata: {
        inputCount: number
        outputCount: number
        droppedCount: number
        inputTokens: number
        outputTokens: number
        strategyUsed: string
        droppedMessageIds: string[]
    }
}
```

---

## 核心实现

### tokenEstimator.ts (内部实现，不依赖外部)

```typescript
/**
 * Token 估算器
 * 
 * 设计决策：在 context-compression 模块内部实现
 * 不依赖 chatToolLoop，保持模块独立性
 */

/**
 * 估算字符串的 token 数
 * 
 * 算法：
 * - ASCII 字符: 4 字符 ≈ 1 token
 * - 非 ASCII 字符（中日韩等）: 1 字符 ≈ 1 token
 */
export function estimateTokens(content: string): number {
    let ascii = 0
    let nonAscii = 0
    
    for (const ch of content) {
        const code = ch.charCodeAt(0)
        if (code <= 0x7f) {
            ascii += 1
        } else {
            nonAscii += 1
        }
    }
    
    return Math.ceil(ascii / 4 + nonAscii)
}

/**
 * 估算消息的 token 数
 */
export function estimateMessageTokens(message: { 
    content?: string | null
    tool_calls?: unknown[]
}): number {
    let tokens = 0
    
    // 消息内容
    if (message.content) {
        tokens += estimateTokens(message.content)
    }
    
    // 工具调用元数据开销
    if (message.tool_calls?.length) {
        tokens += message.tool_calls.length * 50 // 每个工具调用约 50 token 开销
        for (const call of message.tool_calls as { function?: { arguments?: string } }[]) {
            if (call.function?.arguments) {
                tokens += estimateTokens(call.function.arguments)
            }
        }
    }
    
    // 消息元数据开销
    tokens += 10
    
    return tokens
}
```

### ContextUsageMonitor.ts

```typescript
import type { ContextUsage, CompressionConfig } from './types'
import type { ConversationMessage } from '../../../shared/conversationTypes'
import { estimateMessageTokens } from './tokenEstimator'

const DEFAULT_CONFIG: CompressionConfig = {
    triggerThreshold: 0.8,
    targetUsage: 0.5,
    minRecentMessages: 10,
    tokenBudget: 128000,
}

export class ContextUsageMonitor {
    private config: CompressionConfig
    
    constructor(config: Partial<CompressionConfig> = {}) {
        this.config = { ...DEFAULT_CONFIG, ...config }
    }
    
    /**
     * 计算上下文使用情况
     */
    calculateUsage(messages: ConversationMessage[]): ContextUsage {
        let usedTokens = 0
        
        for (const msg of messages) {
            usedTokens += estimateMessageTokens(msg)
        }
        
        const usagePercent = usedTokens / this.config.tokenBudget
        
        return {
            usedTokens,
            totalBudget: this.config.tokenBudget,
            usagePercent,
            remaining: this.config.tokenBudget - usedTokens,
        }
    }
    
    /**
     * 是否需要压缩
     */
    shouldCompress(usage: ContextUsage): boolean {
        return usage.usagePercent > this.config.triggerThreshold
    }
    
    /**
     * 获取压缩目标 token 数
     */
    getTargetTokens(): number {
        return Math.floor(this.config.tokenBudget * this.config.targetUsage)
    }
    
    /**
     * 获取配置
     */
    getConfig(): Readonly<CompressionConfig> {
        return this.config
    }
}
```

### strategies/CompressionStrategy.ts

```typescript
import type { ConversationMessage } from '../../../../shared/conversationTypes'

/**
 * 压缩策略接口
 * 策略模式支持未来扩展
 */
export interface CompressionStrategy {
    /** 策略名称 */
    readonly name: string
    
    /**
     * 执行压缩
     * @param messages - 输入消息
     * @param targetTokens - 目标 token 数
     * @param minRecentMessages - 最少保留的最近消息数
     * @returns 保留的消息
     */
    compress(
        messages: ConversationMessage[],
        targetTokens: number,
        minRecentMessages: number
    ): ConversationMessage[]
}
```

### strategies/KeyMessageExtractionStrategy.ts

```typescript
import type { ConversationMessage } from '../../../../shared/conversationTypes'
import type { CompressionStrategy, ScoredMessage } from '../types'
import { estimateMessageTokens } from '../tokenEstimator'

/**
 * v1 压缩策略：关键消息提取
 * 
 * 优先级规则（从高到低）：
 * 1. 最近 N 条消息 (priority +1000)
 * 2. 所有用户消息 (priority 100)
 * 3. 产出/完成记录 (priority 90)
 * 4. 完整工具调用组 (priority 80)
 * 5. 普通 assistant 消息 (priority 50)
 * 
 * 评分加权（可解释）：
 * - 含文件路径/代码块/数值/关键词（must/should/need/必须/需要）加分
 * - 近期消息时间衰减加分
 * - 冗长或重复的 tool 结果降分
 * 
 * 特殊处理：
 * - 工具调用组必须完整保留或完整丢弃
 * - 历史 system 消息默认不保留（避免重复），仅保留 SUMMARY 类 system 消息（如 `SUMMARY` / `CONVERSATION_SUMMARY`）
 * - 当必保留集合仍超预算时，启用兜底裁剪（见下方说明）
 */
export class KeyMessageExtractionStrategy implements CompressionStrategy {
    readonly name = 'KeyMessageExtraction'
    
    compress(
        messages: ConversationMessage[],
        targetTokens: number,
        minRecentMessages: number
    ): ConversationMessage[] {
        // 1. 标记工具调用组
        const toolGroups = this.identifyToolGroups(messages)
        
        // 2. 评分和标记
        const scored = messages.map((msg, index) => 
            this.scoreMessage(msg, index, messages.length, minRecentMessages, toolGroups)
        )
        
        // 3. 按优先级排序（高优先级在前）
        const sortedByPriority = [...scored].sort((a, b) => b.priority - a.priority)
        
        // 4. 选择直到达到目标 token，保持工具组完整
        const selectedIndexes = new Set<number>()
        let tokenCount = 0
        
        for (const item of sortedByPriority) {
            if (selectedIndexes.has(item.index)) continue
            
            // 如果是工具组的一部分，需要选择整个组
            if (item.toolGroupId) {
                const groupMembers = scored.filter(s => s.toolGroupId === item.toolGroupId)
                const groupTokens = groupMembers.reduce((sum, m) => sum + m.tokens, 0)
                
                if (tokenCount + groupTokens <= targetTokens) {
                    for (const member of groupMembers) {
                        selectedIndexes.add(member.index)
                    }
                    tokenCount += groupTokens
                }
            } else {
                if (tokenCount + item.tokens <= targetTokens) {
                    selectedIndexes.add(item.index)
                    tokenCount += item.tokens
                }
            }
        }
        
        // 5. 按原始顺序返回
        return messages.filter((_, index) => selectedIndexes.has(index))
    }
    
    /**
     * 识别工具调用组
     * assistant(tool_calls) + 对应的 tool(result) 消息
     */
    private identifyToolGroups(messages: ConversationMessage[]): Map<number, string> {
        const toolGroups = new Map<number, string>()
        
        for (let i = 0; i < messages.length; i++) {
            const msg = messages[i]
            
            if (msg.tool_calls?.length) {
                // 这是一个 assistant 消息带工具调用
                const groupId = `group-${i}`
                toolGroups.set(i, groupId)
                
                // 找到所有对应的 tool 结果消息
                const toolCallIds = new Set(msg.tool_calls.map(tc => tc.id))
                
                for (let j = i + 1; j < messages.length; j++) {
                    const nextMsg = messages[j]
                    if (nextMsg.role === 'tool' && nextMsg.tool_call_id && toolCallIds.has(nextMsg.tool_call_id)) {
                        toolGroups.set(j, groupId)
                    }
                    // 遇到下一个 assistant 消息就停止
                    if (nextMsg.role === 'assistant') break
                }
            }
        }
        
        return toolGroups
    }
    
    /**
     * 评分单条消息
     */
    private scoreMessage(
        msg: ConversationMessage,
        index: number,
        total: number,
        minRecentMessages: number,
        toolGroups: Map<number, string>
    ): ScoredMessage {
        // 最近消息加成
        const recencyBonus = index >= total - minRecentMessages ? 1000 : 0
        
        // 基础分数
        let baseScore = 0
        
        if (msg.role === 'user') {
            baseScore = 100 // 用户消息最高优先级
        } else if (msg.content?.includes('ARTIFACT_SAVED')) {
            baseScore = 90
        } else if (msg.content?.includes('NODE_COMPLETE')) {
            baseScore = 85
        } else if (msg.tool_calls?.length) {
            baseScore = 80 // 带工具调用的 assistant
        } else if (msg.role === 'tool') {
            baseScore = 79 // 工具结果
        } else if (msg.role === 'assistant') {
            baseScore = 50 // 普通 assistant
        }
        
        // 位置微调（后面的消息优先）
        const positionBonus = index * 0.01
        
        return {
            message: msg,
            index,
            priority: baseScore + recencyBonus + positionBonus,
            tokens: estimateMessageTokens(msg),
            isPartOfToolGroup: toolGroups.has(index),
            toolGroupId: toolGroups.get(index),
        }
    }
}
```

**超预算兜底（K=4）**

当“必保留集合 + 贪心选取结果”仍超预算时，允许降级：
- 工具结果裁剪为 preview（保留 tool name + status + 前后 N 行/字符）
- 超长 user / assistant 文本裁剪（首段 + 关键 bullet / 首句或“结论/summary”句）
- 若仍超预算：仅保留最近 K 回合（K=4）
  - 回合以 user 消息为界，保留该 user 到下一条 user 之间的 assistant 内容
  - 若保留回合内包含 `tool_calls`，必须保留对应 `tool(result)`，保证协议完整性

### CompressionPipeline.ts

```typescript
import type { ConversationMessage } from '../../../shared/conversationTypes'
import type { CompressionConfig, CompressionResult } from './types'
import type { CompressionStrategy } from './strategies/CompressionStrategy'
import { ContextUsageMonitor } from './ContextUsageMonitor'
import { KeyMessageExtractionStrategy } from './strategies/KeyMessageExtractionStrategy'

export class CompressionPipeline {
    private monitor: ContextUsageMonitor
    private strategy: CompressionStrategy
    
    constructor(config: Partial<CompressionConfig> = {}) {
        this.monitor = new ContextUsageMonitor(config)
        this.strategy = new KeyMessageExtractionStrategy()
    }
    
    /**
     * 如果需要则压缩消息
     */
    compressIfNeeded(messages: ConversationMessage[]): CompressionResult {
        const usage = this.monitor.calculateUsage(messages)
        
        // 不需要压缩
        if (!this.monitor.shouldCompress(usage)) {
            return {
                messages,
                compressed: false,
                metadata: {
                    inputCount: messages.length,
                    outputCount: messages.length,
                    droppedCount: 0,
                    inputTokens: usage.usedTokens,
                    outputTokens: usage.usedTokens,
                    strategyUsed: 'none',
                    droppedMessageIds: [],
                },
            }
        }
        
        // 执行压缩
        const targetTokens = this.monitor.getTargetTokens()
        const minRecentMessages = this.monitor.getConfig().minRecentMessages
        
        const compressed = this.strategy.compress(messages, targetTokens, minRecentMessages)
        const outputUsage = this.monitor.calculateUsage(compressed)
        
        // 计算被丢弃的消息 ID
        const keptIds = new Set(compressed.map(m => m.id))
        const droppedMessageIds = messages
            .filter(m => !keptIds.has(m.id))
            .map(m => m.id)
        
        return {
            messages: compressed,
            compressed: true,
            metadata: {
                inputCount: messages.length,
                outputCount: compressed.length,
                droppedCount: messages.length - compressed.length,
                inputTokens: usage.usedTokens,
                outputTokens: outputUsage.usedTokens,
                strategyUsed: this.strategy.name,
                droppedMessageIds,
            },
        }
    }
    
    /**
     * 获取当前上下文使用情况（供 UI 显示）
     */
    getUsage(messages: ConversationMessage[]): ContextUsage {
        return this.monitor.calculateUsage(messages)
    }
    
    /**
     * 更换压缩策略（未来扩展）
     */
    setStrategy(strategy: CompressionStrategy): void {
        this.strategy = strategy
    }
}
```

### index.ts (Public API)

```typescript
// Types
export type {
    ContextUsage,
    CompressionConfig,
    CompressionResult,
    ScoredMessage,
} from './types'

export type { CompressionStrategy } from './strategies/CompressionStrategy'

// Classes
export { ContextUsageMonitor } from './ContextUsageMonitor'
export { CompressionPipeline } from './CompressionPipeline'
export { KeyMessageExtractionStrategy } from './strategies/KeyMessageExtractionStrategy'

// Utilities
export { estimateTokens, estimateMessageTokens } from './tokenEstimator'
```

---

## 与 context-builder 集成 (可选)

```typescript
// context-builder/buildLlmMessages.ts
import { CompressionPipeline } from '../context-compression'

export function buildLlmMessagesFromConversation(
    options: ContextBuildOptions
): ContextBuildResult {
    let { messages } = options
    
    // 可选：压缩
    if (options.settings?.enableCompression) {
        const pipeline = new CompressionPipeline({
            tokenBudget: options.settings.tokenBudget,
        })
        const result = pipeline.compressIfNeeded(messages)
        messages = result.messages
    }
    
    // 继续构建...
}
```

---

## Loop 内压缩设计 ⚠️ 重要

### 需求背景

在 `chatToolLoop` 内部，每轮 LLM 调用后会累积 `assistant(tool_calls)` 和 `tool(result)` 消息。如果工具输出很大，可能超出 token 限制。

### 设计原则

**只压缩 Loop 开始前的历史消息，不压缩当前 Loop 内产生的消息**

这样保证当前轮次的工具调用协议连贯性（LLM 需要看到完整的 tool_calls + tool results）。

### 实现方案

```typescript
// chatToolLoop.ts 改进

export async function chatToolLoop(params: {
    messages: OpenAIChatMessage[]
    compressionPipeline?: CompressionPipeline  // 新增可选参数
    // ...existing params...
}) {
    // 记录 Loop 开始时的消息边界
    let loopStartIndex = params.messages.length
    
    for (let turn = 0; turn < maxTurns; turn++) {
        // Loop 内压缩检查：只压缩 loopStartIndex 之前的消息
        if (params.compressionPipeline) {
            const usage = params.compressionPipeline.getUsage(params.messages)
            
            if (usage.usagePercent > 0.8) {  // 触发阈值
                // 分离历史消息和当前 Loop 消息
                const historical = params.messages.slice(0, loopStartIndex)
                const currentLoop = params.messages.slice(loopStartIndex)
                
                // 只压缩历史部分
                const compressed = params.compressionPipeline.compressIfNeeded(historical)
                
                // 重组：压缩后的历史 + 完整的当前 Loop
                params.messages.length = 0
                params.messages.push(...compressed.messages, ...currentLoop)
                
                // 更新边界索引
                loopStartIndex = compressed.messages.length
                
                console.log(`[Loop Compression] Compressed ${historical.length} → ${compressed.messages.length} historical messages`)
            }
        }
        
        // LLM 调用...
        const llmRes = await llmAdapter.chatCompletions({
            messages: params.messages,
            // ...
        })
        
        // 累积 assistant 消息
        params.messages.push(assistant)
        
        // 执行工具调用，累积 tool 结果
        for (const call of assistant.tool_calls ?? []) {
            const result = await toolHost.executeToolCall(...)
            params.messages.push({
                role: 'tool',
                tool_call_id: call.id,
                content: JSON.stringify(result),
            })
        }
    }
}
```

### 消息区域划分

```
┌─────────────────────────────────────────────────────────┐
│ messages 数组                                            │
├─────────────────────────────────────────────────────────┤
│ [0] system                                              │
│ [1] user (历史)                                          │
│ [2] assistant (历史)                                     │
│ ...                                                     │
│ [loopStartIndex-1] (历史结束)                            │
├─────────────────────────────────────────────────────────┤
│ [loopStartIndex] assistant(tool_calls) ← 当前 Loop       │
│ [loopStartIndex+1] tool(result)        ← 当前 Loop       │
│ [loopStartIndex+2] assistant(tool_calls) ← 当前 Loop     │
│ ...                                                     │
└─────────────────────────────────────────────────────────┘
         ↑                                ↑
      可压缩区域                        不可压缩区域
```

### 压缩行为

| 消息区域 | 压缩行为 | 原因 |
|---------|---------|------|
| `[0, loopStartIndex)` 历史 | ✅ 可压缩 | 已结束的对话，可安全截断 |
| `[loopStartIndex, end)` 当前 Loop | ❌ 保持完整 | 正在进行的工具调用，LLM 需要完整上下文 |

### CompressionPipeline 扩展

需要在 `CompressionPipeline` 中添加方法支持 Loop 内压缩：

```typescript
// CompressionPipeline.ts 扩展

export class CompressionPipeline {
    // ...existing methods...
    
    /**
     * 压缩历史消息，保留当前 Loop 消息
     * 用于 chatToolLoop 内部调用
     */
    compressHistoricalMessages(
        allMessages: ConversationMessage[],
        loopStartIndex: number
    ): {
        messages: ConversationMessage[]
        newLoopStartIndex: number
    } {
        const historical = allMessages.slice(0, loopStartIndex)
        const currentLoop = allMessages.slice(loopStartIndex)
        
        const compressed = this.compressIfNeeded(historical)
        
        return {
            messages: [...compressed.messages, ...currentLoop],
            newLoopStartIndex: compressed.messages.length,
        }
    }
    
    /**
     * 检查是否需要压缩
     */
    shouldCompress(messages: ConversationMessage[]): boolean {
        const usage = this.monitor.calculateUsage(messages)
        return this.monitor.shouldCompress(usage)
    }
}
```

### 与 Story 7-5 的关系

Story 7-5 (Chat 模式集成) 需要：
1. 在 `callAgentChat` 中创建 `CompressionPipeline` 实例
2. 传递给 `chatToolLoop`
3. Loop 会在每轮自动检查并压缩历史

---

## 测试策略

```typescript
describe('tokenEstimator', () => {
    it('should estimate ASCII text correctly', () => {
        expect(estimateTokens('hello')).toBe(2) // 5/4 = 1.25 → 2
    })
    
    it('should estimate Chinese text correctly', () => {
        expect(estimateTokens('你好')).toBe(2) // 2 non-ASCII
    })
    
    it('should estimate mixed text correctly', () => {
        expect(estimateTokens('hello你好')).toBe(4) // 2 + 2
    })
})

describe('ContextUsageMonitor', () => {
    it('should calculate usage percentage correctly', () => {})
    it('should trigger compression at 80% threshold', () => {})
    it('should not trigger compression below threshold', () => {})
})

describe('KeyMessageExtractionStrategy', () => {
    it('should always keep recent messages', () => {})
    it('should prioritize user messages', () => {})
    it('should keep tool call groups together', () => {})
    it('should keep artifact records', () => {})
    it('should respect target token limit', () => {})
})

describe('CompressionPipeline', () => {
    it('should not compress when under threshold', () => {})
    it('should compress to target 50% when over threshold', () => {})
    it('should return dropped message IDs', () => {})
})
```

---

## 松耦合验证

✅ 只依赖 `shared/conversationTypes.ts` 的类型
✅ 内部实现 token 估算，不依赖 `chatToolLoop`
✅ 不依赖 `context-builder`（可被其调用，但不反向依赖）
✅ 策略模式支持扩展

## 高内聚验证

✅ 所有文件服务于单一目标：上下文压缩
✅ Token 估算、监控、策略、管道在同一模块内
✅ 无分散的相关逻辑
