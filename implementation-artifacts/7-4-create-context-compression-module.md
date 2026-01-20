# Story 7-4: Create Context Compression Module

## Overview
**Epic**: 7 – Unified Conversation Context  
**Priority**: Phase 2 (depends on 7.3)  
**Status**: `done`

## Goal
创建独立的上下文压缩模块 `context-compression/`，采用 Claude Code 风格的阈值触发模式。当上下文使用超过阈值时自动触发压缩，保留关键消息。

## Business Value
- **可扩展**：策略模式支持未来升级（LLM 摘要、向量检索）
- **用户体验**：显示上下文剩余量
- **成本控制**：避免超出 token 限制

## Acceptance Criteria
1. **模块结构**：
   ```
   electron/core/context-compression/
   ├── index.ts
   ├── types.ts
   ├── tokenEstimator.ts
   ├── CompressionPipeline.ts
   ├── ContextUsageMonitor.ts
   ├── strategies/
   │   ├── CompressionStrategy.ts
   │   └── KeyMessageExtractionStrategy.ts
   └── context-compression.test.ts
   ```

2. **阈值触发**：当 `usagePercent > 80%` 时触发压缩

3. **压缩目标**：压缩后目标占用 `50%`

4. **v1 策略 (KeyMessageExtractionStrategy)**：
   - 保留所有用户消息
   - 保留产出/完成记录（`ARTIFACT_SAVED`, `NODE_COMPLETE`）
   - 保留完整工具调用组（`assistant(tool_calls)` + `tool(result)`）
   - 保留最近 N 条消息
   - **历史 system 消息默认不保留**（避免重复）
   - **仅保留 SUMMARY 类 system 消息**（v2 摘要，例如以 `SUMMARY` / `CONVERSATION_SUMMARY` 开头）
   - **打分 + 贪心选取**：对非必保留消息做可解释打分（文件路径/代码块/数值/关键词/近期加分，冗长 tool 结果降分），在预算内贪心选取
   - **超预算兜底（K=4）**：当必保留集合仍超预算时，允许降级
     - 工具结果裁剪为 preview
     - 超长 user/assistant 文本裁剪
     - 仍超预算时仅保留最近 K 回合（user + assistant + 回合内 tool 结果），保证协议完整性

5. **接口定义**：
   ```typescript
   interface ContextUsage {
     usedTokens: number
     totalBudget: number
     usagePercent: number
     remaining: number
   }
   ```

6. **Loop 内压缩支持**：
   - `CompressionPipeline.shouldCompress(messages)`
   - `CompressionPipeline.compressHistoricalMessages(allMessages, loopStartIndex)`

## Tasks / Subtasks

### Review Follow-ups (AI)
- [x] [AI-Review][HIGH] Ensure tool-call groups are always preserved (assistant tool_calls + tool results) rather than only when a member is selected [crewagent-runtime/electron/core/context-compression/strategies/KeyMessageExtractionStrategy.ts:30]
- [x] [AI-Review][MEDIUM] Enforce or explicitly handle the 50% compression target when mandatory messages exceed target (document fallback or adjust strategy) [crewagent-runtime/electron/core/context-compression/strategies/KeyMessageExtractionStrategy.ts:33]
- [x] [AI-Review][MEDIUM] Align CompressionStrategy interface with Story (no extra args) or update the story/spec to match implementation [crewagent-runtime/electron/core/context-compression/strategies/CompressionStrategy.ts:6]
- [x] [AI-Review][MEDIUM] Add Dev Agent Record with File List to document actual files created/modified for Story 7-4 [_bmad-output/implementation-artifacts/7-4-create-context-compression-module.md:1]

## Out of Scope
- LLM 摘要策略（未来 v2）
- 向量检索策略（未来 v3）
- UI 指示器（Story 7.8）

## Dependencies
- **Story 7-3**: Create Context Builder Module

---

## Technical Context

### 设计原则

遵循 Story 7-3 的松耦合 + 高内聚原则：
- 定义自己的接口，不依赖外部类型
- 调用方负责适配

### 模块依赖图

```
                    ┌─────────────────────────────────┐
                    │  electron/core/context-builder   │
                    └─────────────────────────────────┘
                                    ▲
                                    │ 依赖
                    ┌─────────────────────────────────┐
                    │ electron/core/context-compression│
                    └─────────────────────────────────┘
                                    ▲
                   ┌────────────────┼────────────────┐
                   │                │                │
                ExecutionEngine  AgentHandler   ChatHandler
```

---

## 接口设计

### types.ts

```typescript
import type { ConversationMessage, OpenAIChatMessage } from '../../../shared/conversationTypes'

// ============================================================================
// Context Usage
// ============================================================================

export interface ContextUsage {
    usedTokens: number
    totalBudget: number
    usagePercent: number
    remaining: number
}

// ============================================================================
// Compression Config
// ============================================================================

export interface CompressionConfig {
    /** 触发压缩的阈值 (0-1)，default: 0.8 */
    triggerThreshold: number
    
    /** 压缩后的目标使用率 (0-1)，default: 0.5 */
    targetUsage: number
    
    /** 最少保留的最近消息数，default: 10 */
    minRecentMessages: number
    
    /** 模型的 token 预算 */
    tokenBudget: number
}

// ============================================================================
// Compression Strategy
// ============================================================================

export interface CompressionStrategy {
    name: string
    
    /**
     * 选择要保留的消息
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

// ============================================================================
// Compression Result
// ============================================================================

export interface CompressionResult {
    messages: ConversationMessage[]
    metadata: {
        inputCount: number
        outputCount: number
        droppedCount: number
        inputTokens: number
        outputTokens: number
        targetTokens: number
        targetExceeded: boolean
        strategyUsed: string
        droppedMessageIds: string[]
    }
}
```

### ContextUsageMonitor.ts

```typescript
import type { ContextUsage, CompressionConfig } from './types'

export class ContextUsageMonitor {
    private config: CompressionConfig
    
    constructor(config: Partial<CompressionConfig> = {}) {
        this.config = {
            triggerThreshold: config.triggerThreshold ?? 0.8,
            targetUsage: config.targetUsage ?? 0.5,
            minRecentMessages: config.minRecentMessages ?? 10,
            tokenBudget: config.tokenBudget ?? 128000,
        }
    }
    
    /**
     * 估算 token 数量
     * 
     * 设计决策：在 context-compression 模块内部实现，不依赖 chatToolLoop
     * 保持模块独立性和松耦合
     * 
     * 算法：
     * - ASCII 字符: 4 字符 ≈ 1 token
     * - 非 ASCII 字符（中日韩等）: 1 字符 ≈ 1 token
     */
    estimateTokens(content: string): number {
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
     * 计算上下文使用情况
     */
    calculateUsage(messages: { content?: string | null }[]): ContextUsage {
        let usedTokens = 0
        for (const msg of messages) {
            if (msg.content) {
                usedTokens += this.estimateTokens(msg.content)
            }
        }
        
        return {
            usedTokens,
            totalBudget: this.config.tokenBudget,
            usagePercent: usedTokens / this.config.tokenBudget,
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
     * 计算压缩目标 token 数
     */
    getTargetTokens(): number {
        return Math.floor(this.config.tokenBudget * this.config.targetUsage)
    }
}
```

### strategies/KeyMessageExtractionStrategy.ts

```typescript
import type { ConversationMessage } from '../../../../shared/conversationTypes'
import type { CompressionStrategy } from '../types'

/**
 * v1 压缩策略：关键消息提取
 * 
 * 优先级：
 * 1. 最近 N 条消息（必须保留）
 * 2. 所有用户消息
 * 3. 产出/完成记录
 * 4. 完整工具调用组
 */
export class KeyMessageExtractionStrategy implements CompressionStrategy {
    name = 'KeyMessageExtraction'
    
    private minRecentMessages: number
    
    constructor(minRecentMessages = 10) {
        this.minRecentMessages = minRecentMessages
    }
    
    compress(
        messages: ConversationMessage[],
        targetTokens: number
    ): ConversationMessage[] {
        // 1. 标记关键消息
        const scored = messages.map((msg, index) => ({
            msg,
            index,
            priority: this.scorePriority(msg, index, messages.length),
        }))
        
        // 2. 按优先级排序（高优先级在前）
        scored.sort((a, b) => b.priority - a.priority)
        
        // 3. 选择直到达到目标 token
        const selected: typeof scored = []
        let tokenCount = 0
        
        for (const item of scored) {
            const msgTokens = this.estimateTokens(item.msg)
            if (tokenCount + msgTokens <= targetTokens) {
                selected.push(item)
                tokenCount += msgTokens
            }
        }
        
        // 4. 按原始顺序排序
        selected.sort((a, b) => a.index - b.index)
        
        return selected.map(s => s.msg)
    }
    
    private scorePriority(
        msg: ConversationMessage,
        index: number,
        total: number
    ): number {
        const recencyBonus = index >= total - this.minRecentMessages ? 1000 : 0
        
        let baseScore = 0
        
        // 用户消息高优先级
        if (msg.role === 'user') baseScore = 100
        
        // 产出记录
        if (msg.content?.includes('ARTIFACT_SAVED')) baseScore = 90
        if (msg.content?.includes('NODE_COMPLETE')) baseScore = 85
        
        // 工具调用组
        if (msg.tool_calls?.length) baseScore = 80
        if (msg.role === 'tool') baseScore = 79
        
        return baseScore + recencyBonus + index * 0.01
    }
    
    private estimateTokens(msg: ConversationMessage): number {
        const content = msg.content ?? ''
        let tokens = 0
        for (const ch of content) {
            tokens += ch.charCodeAt(0) > 0x7f ? 1 : 0.25
        }
        return Math.ceil(tokens) + 10 // overhead
    }
}
```

### CompressionPipeline.ts

```typescript
import type { ConversationMessage } from '../../../shared/conversationTypes'
import type { CompressionConfig, CompressionResult, CompressionStrategy } from './types'
import { ContextUsageMonitor } from './ContextUsageMonitor'
import { KeyMessageExtractionStrategy } from './strategies/KeyMessageExtractionStrategy'

export class CompressionPipeline {
    private monitor: ContextUsageMonitor
    private strategy: CompressionStrategy
    
    constructor(config: Partial<CompressionConfig> = {}) {
        this.monitor = new ContextUsageMonitor(config)
        this.strategy = new KeyMessageExtractionStrategy(config.minRecentMessages)
    }
    
    /**
     * 如果需要则压缩消息
     */
    compressIfNeeded(messages: ConversationMessage[]): CompressionResult {
        const usage = this.monitor.calculateUsage(messages)
        
        if (!this.monitor.shouldCompress(usage)) {
            return {
                messages,
                metadata: {
                    inputCount: messages.length,
                    outputCount: messages.length,
                    droppedCount: 0,
                    inputTokens: usage.usedTokens,
                    outputTokens: usage.usedTokens,
                    strategyUsed: 'none',
                },
            }
        }
        
        const targetTokens = this.monitor.getTargetTokens()
        const compressed = this.strategy.compress(messages, targetTokens)
        const outputUsage = this.monitor.calculateUsage(compressed)
        
        return {
            messages: compressed,
            metadata: {
                inputCount: messages.length,
                outputCount: compressed.length,
                droppedCount: messages.length - compressed.length,
                inputTokens: usage.usedTokens,
                outputTokens: outputUsage.usedTokens,
                strategyUsed: this.strategy.name,
            },
        }
    }
    
    /**
     * 获取当前上下文使用情况
     */
    getUsage(messages: {content?: string | null}[]): ContextUsage {
        return this.monitor.calculateUsage(messages)
    }
}
```

---

## Files to Create

| File | Description |
|------|-------------|
| `electron/core/context-compression/index.ts` | 导出 public API |
| `electron/core/context-compression/types.ts` | 类型定义 |
| `electron/core/context-compression/ContextUsageMonitor.ts` | 使用量监控 |
| `electron/core/context-compression/CompressionPipeline.ts` | 压缩管道 |
| `electron/core/context-compression/strategies/KeyMessageExtractionStrategy.ts` | v1 策略 |
| `electron/core/context-compression/context-compression.test.ts` | 单元测试 |

---

## Testing Strategy

```typescript
describe('ContextUsageMonitor', () => {
    it('should estimate tokens correctly', () => {})
    it('should calculate usage percentage', () => {})
    it('should trigger compression at 80% threshold', () => {})
})

describe('KeyMessageExtractionStrategy', () => {
    it('should always keep recent messages', () => {})
    it('should prioritize user messages', () => {})
    it('should keep artifact records', () => {})
    it('should keep tool call groups together', () => {})
})

describe('CompressionPipeline', () => {
    it('should not compress when under threshold', () => {})
    it('should compress to target 50% when over threshold', () => {})
})
```

---

## Related Artifacts
- Architecture: `_bmad-output/architecture/unified-conversation-context.md`
- Epic Plan: `_bmad-output/implementation-artifacts/7-unified-conversation-context.md`
- Story 7-3: Context Builder Module

## Dev Agent Record

### Agent Model Used
GPT-5 (Codex CLI)

### Debug Log References
- `crewagent-runtime`: `npm test -- electron/core/context-compression/context-compression.test.ts`

### Completion Notes List
- Ensured tool-call groups are always preserved when compressing.
- Added explicit metadata for target token usage/exceedance.
- Added loop-safe compression helpers (shouldCompress/compressHistoricalMessages).
- Updated story interfaces to match design requirements.
- Ran context-compression unit tests (npm reported user config warnings for `python`/`install`).

### File List
- crewagent-runtime/electron/core/context-compression/index.ts
- crewagent-runtime/electron/core/context-compression/types.ts
- crewagent-runtime/electron/core/context-compression/tokenEstimator.ts
- crewagent-runtime/electron/core/context-compression/ContextUsageMonitor.ts
- crewagent-runtime/electron/core/context-compression/CompressionPipeline.ts
- crewagent-runtime/electron/core/context-compression/strategies/CompressionStrategy.ts
- crewagent-runtime/electron/core/context-compression/strategies/KeyMessageExtractionStrategy.ts
- crewagent-runtime/electron/core/context-compression/context-compression.test.ts

## Change Log
- 2026-01-18: Applied new design requirements, fixed compression edge cases, and updated documentation.
- 2026-01-18: Ran context compression tests and marked story done.
