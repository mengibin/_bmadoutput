# Design: Context Builder Module (松耦合 + 高内聚)

**Story:** `7-3-create-context-builder-module.md`  
**设计原则:** 松耦合、高内聚、依赖倒置

---

## 设计目标

1. **高内聚**：模块内所有组件服务于单一目标 - 构建 LLM 上下文
2. **松耦合**：不依赖外部实现细节，只依赖抽象接口
3. **依赖倒置**：外部模块适配 context-builder 接口，而非 context-builder 适配外部类型

---

## 依赖关系图

```
                    ┌─────────────────────────────────┐
                    │  shared/conversationTypes.ts     │
                    │  (基础类型: ConversationMessage) │
                    └─────────────────────────────────┘
                                    ▲
                                    │ 依赖
                    ┌─────────────────────────────────┐
                    │  electron/core/context-builder   │
                    │  - 定义自己的接口 (AgentInfo等)  │
                    │  - 不依赖 runtimeStore           │
                    │  - 不依赖 promptComposer        │
                    └─────────────────────────────────┘
                                    ▲
                   ┌────────────────┼────────────────┐
                   │                │                │
┌──────────────────┴─┐  ┌──────────┴─────────┐  ┌───┴──────────────┐
│ ExecutionEngine     │  │ AgentHandler       │  │ ChatHandler      │
│ (适配 RunContext)   │  │ (适配 AgentInfo)   │  │ (简单调用)       │
└────────────────────┘  └────────────────────┘  └──────────────────┘
```

---

## 模块结构

```
electron/core/context-builder/
├── index.ts                      # Public API 导出
├── types.ts                      # 独立接口定义 (不依赖外部类型)
├── buildLlmMessages.ts           # 核心转换逻辑
├── systemPromptComposer.ts       # System Prompt 构建
├── templates/                    # Prompt 模板
│   ├── base-rules-run.md
│   ├── base-rules-chat.md
│   ├── tool-policy.md
│   └── run-directive-protocol.md
└── context-builder.test.ts       # 单元测试
```

---

## 接口设计 (types.ts)

```typescript
/**
 * Context Builder 独立类型定义
 * 设计原则: 不依赖 runtimeStore 或其他外部模块的类型
 */

import type { ConversationMessage, OpenAIChatMessage } from '../../../shared/conversationTypes'

// ============================================================================
// Session Mode
// ============================================================================

export type SessionMode = 'chat' | 'agent' | 'run'

// ============================================================================
// Agent Info (调用方负责从 AgentDefinition 映射)
// ============================================================================

export interface AgentInfo {
    id: string
    name: string
    role: string
    identity?: string
    communicationStyle?: string
    principles?: string[]
    systemPrompt?: string  // 如果有预编译的 system prompt
}

// ============================================================================
// Tool Policy (调用方负责从 AgentToolPolicy 映射)
// ============================================================================

export interface ToolPolicyConfig {
    allowedCategories?: string[]   // e.g., ['fs', 'project', 'ui']
    deniedCategories?: string[]
    allowedTools?: string[]
    deniedTools?: string[]
    customRules?: string[]         // 自定义规则文本
}

// ============================================================================
// Run Context (调用方负责从 ExecutionEngine 状态映射)
// ============================================================================

export interface RunContext {
    packageName: string
    workflowName: string
    currentStep: {
        id: string
        name: string
        instruction: string
    }
    state?: {
        variables?: Record<string, unknown>
        stepsCompleted?: string[]
        artifacts?: string[]
    }
    graph?: {
        outgoingEdges?: Array<{
            label: string
            targetNodeId: string
            isDefault?: boolean
        }>
    }
}

// ============================================================================
// Context Build Options
// ============================================================================

export interface ContextBuildOptions {
    /** 持久化的对话消息 */
    messages: ConversationMessage[]
    
    /** 会话模式 */
    mode: SessionMode
    
    /** Agent 信息 (agent/run 模式) */
    agent?: AgentInfo
    
    /** 工具策略 */
    toolPolicy?: ToolPolicyConfig
    
    /** Run 上下文 (run 模式) */
    runContext?: RunContext
    
    /** 构建设置 */
    settings?: {
        includeSystemPrompt?: boolean  // default: true
        maxMessages?: number           // 未来压缩用
    }
}

// ============================================================================
// Context Build Result
// ============================================================================

export interface ContextBuildResult {
    /** 构建的 OpenAI 消息数组 */
    messages: OpenAIChatMessage[]
    
    /** 构建元数据 */
    metadata: {
        inputCount: number
        outputCount: number
        filteredCount: number
        systemPromptIncluded: boolean
        systemPromptLength: number
    }
}
```

---

## 核心实现

### buildLlmMessages.ts

```typescript
import { toOpenAIMessage, type ConversationMessage } from '../../../shared/conversationTypes'
import { composeSystemPrompt } from './systemPromptComposer'
import type { ContextBuildOptions, ContextBuildResult, OpenAIChatMessage } from './types'

/**
 * 从持久化的对话消息构建 LLM 消息数组
 * 
 * @param options - 构建选项
 * @returns 构建结果，包含消息数组和元数据
 */
export function buildLlmMessagesFromConversation(
    options: ContextBuildOptions
): ContextBuildResult {
    const result: OpenAIChatMessage[] = []
    let filteredCount = 0
    let systemPromptLength = 0

    // 1. 构建 System Prompt (不存入历史)
    const includeSystemPrompt = options.settings?.includeSystemPrompt !== false
    if (includeSystemPrompt) {
        const systemContent = composeSystemPrompt(options)
        result.push({ role: 'system', content: systemContent })
        systemPromptLength = systemContent.length
    }

    // 2. 转换对话消息
    for (const msg of options.messages) {
        // 跳过 system 消息 (我们动态构建自己的)
        if (msg.role === 'system') continue
        
        // 跳过标记为不包含的消息
        if (msg.includeInContext === false) {
            filteredCount++
            continue
        }

        result.push(toOpenAIMessage(msg))
    }

    return {
        messages: result,
        metadata: {
            inputCount: options.messages.length,
            outputCount: result.length,
            filteredCount,
            systemPromptIncluded: includeSystemPrompt,
            systemPromptLength,
        },
    }
}
```

### systemPromptComposer.ts

```typescript
import type { ContextBuildOptions, AgentInfo, ToolPolicyConfig, RunContext, SessionMode } from './types'

// 模板导入
import baseRulesRunTemplate from './templates/base-rules-run.md?raw'
import baseRulesChatTemplate from './templates/base-rules-chat.md?raw'
import toolPolicyTemplate from './templates/tool-policy.md?raw'
import runDirectiveTemplate from './templates/run-directive-protocol.md?raw'

/**
 * 根据选项组合 System Prompt
 */
export function composeSystemPrompt(options: ContextBuildOptions): string {
    const parts: string[] = []

    // 1. 模式标识
    parts.push(`# Mode: ${options.mode.toUpperCase()}`)

    // 2. 基础规则
    parts.push(buildBaseRules(options.mode))

    // 3. 工具策略
    if (options.toolPolicy) {
        const policy = buildToolPolicy(options.toolPolicy)
        if (policy) parts.push(policy)
    }

    // 4. Agent Persona
    if (options.agent) {
        const persona = buildAgentPersona(options.agent)
        if (persona) parts.push(persona)
    }

    // 5. Run 指令 (仅 run 模式)
    if (options.mode === 'run' && options.runContext) {
        parts.push(buildRunDirective(options.runContext))
    }

    return parts.filter(Boolean).join('\n\n---\n\n')
}

// ============================================================================
// 内部构建函数
// ============================================================================

function buildBaseRules(mode: SessionMode): string {
    return mode === 'run' ? baseRulesRunTemplate : baseRulesChatTemplate
}

function buildToolPolicy(policy: ToolPolicyConfig): string {
    const lines: string[] = ['## Tool Policy']
    
    if (policy.allowedCategories?.length) {
        lines.push(`Allowed categories: ${policy.allowedCategories.join(', ')}`)
    }
    if (policy.deniedCategories?.length) {
        lines.push(`Denied categories: ${policy.deniedCategories.join(', ')}`)
    }
    if (policy.allowedTools?.length) {
        lines.push(`Allowed tools: ${policy.allowedTools.join(', ')}`)
    }
    if (policy.deniedTools?.length) {
        lines.push(`Denied tools: ${policy.deniedTools.join(', ')}`)
    }
    if (policy.customRules?.length) {
        lines.push('', '### Custom Rules', ...policy.customRules.map(r => `- ${r}`))
    }
    
    return lines.length > 1 ? lines.join('\n') : ''
}

function buildAgentPersona(agent: AgentInfo): string {
    // 如果有预编译的 systemPrompt，直接使用
    if (agent.systemPrompt?.trim()) {
        return agent.systemPrompt
    }
    
    const lines: string[] = ['## Agent Persona']
    lines.push(`**Name:** ${agent.name}`)
    lines.push(`**Role:** ${agent.role}`)
    
    if (agent.identity) {
        lines.push(`**Identity:** ${agent.identity}`)
    }
    if (agent.communicationStyle) {
        lines.push(`**Communication Style:** ${agent.communicationStyle}`)
    }
    if (agent.principles?.length) {
        lines.push('', '**Principles:**')
        lines.push(...agent.principles.map(p => `- ${p}`))
    }
    
    return lines.join('\n')
}

function buildRunDirective(ctx: RunContext): string {
    const lines: string[] = ['## Run Directive']
    
    lines.push(`**Package:** ${ctx.packageName}`)
    lines.push(`**Workflow:** ${ctx.workflowName}`)
    lines.push(`**Current Step:** ${ctx.currentStep.name} (${ctx.currentStep.id})`)
    lines.push('')
    lines.push('### Step Instruction')
    lines.push(ctx.currentStep.instruction)
    
    if (ctx.state?.stepsCompleted?.length) {
        lines.push('')
        lines.push(`**Completed Steps:** ${ctx.state.stepsCompleted.join(' → ')}`)
    }
    
    if (ctx.graph?.outgoingEdges?.length) {
        lines.push('')
        lines.push('### Available Transitions')
        for (const edge of ctx.graph.outgoingEdges) {
            const defaultTag = edge.isDefault ? ' (default)' : ''
            lines.push(`- **${edge.label}** → ${edge.targetNodeId}${defaultTag}`)
        }
    }
    
    return lines.join('\n')
}
```

---

## 调用方适配示例

### ExecutionEngine 适配

```typescript
// executionEngine.ts
import { buildLlmMessagesFromConversation, type AgentInfo, type RunContext } from '../core/context-builder'

// 从 ExecutionEngine 状态映射到 context-builder 接口
function mapToContextBuilderOptions(session: EngineSession): ContextBuildOptions {
    return {
        messages: session.conversationMessages,
        mode: 'run',
        agent: mapAgentDefinition(session.activeAgent),
        runContext: mapRunContext(session),
        toolPolicy: mapToolPolicy(session.activeAgent.tools),
    }
}

function mapAgentDefinition(def: AgentDefinition): AgentInfo {
    return {
        id: def.id,
        name: def.metadata.name,
        role: def.persona.role,
        identity: def.persona.identity,
        communicationStyle: def.persona.communication_style,
        principles: def.persona.principles,
        systemPrompt: def.systemPrompt,
    }
}

function mapRunContext(session: EngineSession): RunContext {
    return {
        packageName: session.packageName,
        workflowName: session.workflowName,
        currentStep: {
            id: session.currentNodeId,
            name: session.currentNodeBrief.name,
            instruction: session.currentStepContent,
        },
        state: {
            variables: session.workflowState.variables,
            stepsCompleted: session.workflowState.stepsCompleted,
        },
    }
}
```

---

## 迁移计划

| 步骤 | 内容 | 说明 |
|------|------|------|
| 1 | 创建目录结构 | `electron/core/context-builder/` |
| 2 | 实现 types.ts | 独立接口定义 |
| 3 | 复制模板文件 | `templates/*.md` |
| 4 | 实现 systemPromptComposer.ts | 迁移构建逻辑 |
| 5 | 实现 buildLlmMessages.ts | 核心转换函数 |
| 6 | 添加 index.ts | 导出 public API |
| 7 | 添加单元测试 | 验证功能正确性 |
| 8 | TypeScript 检查 | 确保无编译错误 |

---

## 高内聚验证

✅ 模块内所有文件服务于单一目标：构建 LLM 上下文
✅ 类型定义、构建逻辑、模板在同一模块内
✅ 无分散的相关逻辑

## 松耦合验证

✅ 只依赖 `shared/conversationTypes.ts` 的基础类型
✅ 不 import `runtimeStore.ts`
✅ 不 import `promptComposer.ts`
✅ 定义自己的接口，调用方负责适配
