# CrewAgent Runtime — ExecutionEngine ↔ LLM 对话协议（OpenAI-compatible / ToolCalls）

> 本文用于指导 Runtime 开发：如何用 **OpenAI Chat Completions + ToolCalls** 协议与 LLM 对话，并把 ToolHost（尤其是 `fs.*`）的执行结果回填到下一次请求中，确保模型能“理解已发生的过程”，同时不爆 token。
>
> 参考对照脚本（强烈建议先通读）：
> - `crewagent-runtime/docs/create-story-micro-ideal-trace.md`
> - `_bmad-output/implementation-artifacts/runtime/create-story-micro-ideal-trace.md`

## 1. 设计目标

1) **OpenAI-compatible**：DeepSeek 等兼容 OpenAI API 的 Provider 可直接复用同一套 `messages/tools/tool_calls/tool` 交互模型。  
2) **可恢复**：不依赖“聊天记忆”来恢复；恢复只依赖 `@state/workflow.md`（Document-as-State）+ `@pkg/workflow.graph.json`。  
3) **窄读优先**：大文档默认不整段回传；让 LLM 通过 `fs.search → fs.read(window)` 按需取证。  
4) **强约束 + 可审计**：ToolHost 强制 mounts、只读规则、原子写、frontmatter 校验、跳转合法性校验，并写审计日志（JSONL）。

## 2. 基本约定（与架构一致）

### 2.1 Mounts（LLM 只能看到这些路径）

- `@project/...`：用户 ProjectRoot（读写，默认产物写 `@project/artifacts/...`）
- `@pkg/...`：`.bmad` 解包后的 package cache（只读）
- `@state/...`：run 的私有状态目录（读写但用户不可见），核心文件 `@state/workflow.md`

### 2.2 LLM-as-Engine（核心循环）

与 `create-story-micro` 的对照脚本一致：每轮执行大体遵循：

1. 读取 `@state/workflow.md` frontmatter（状态）
2. 读取 `@pkg/workflow.graph.json`（允许的后继跳转）
3. 读取当前节点 step 文件（`@pkg/steps/...md`）
4. 产出文件到 `@project/...`，并 patch `@state/workflow.md`（推进状态）
5. Runtime 做硬校验（见 6/7）

## 3. OpenAI Chat Completions 消息模型（必须遵守）

### 3.1 我们使用的角色（roles）

- `system`：Runtime 基础规则（mounts、只读、状态更新要求、autopilot 等）
- `user`：启动/继续指令（RUN_DIRECTIVE）、节点导航（NODE_BRIEF）、用户输入（USER_INPUT）
- `assistant`：LLM 输出；可能包含 `tool_calls`（函数调用）
- `tool`：ToolHost 执行后的结果回填（必须关联 `tool_call_id`）

> 说明：本文不依赖 `developer` role（不同 Provider 支持度不一），统一使用 `system`/`user`。

补充（ChatMode / ConversationType=`chat`）：
- 仍使用同样的 `messages/tools/tool_calls/tool` 协议与 ToolHost 回填规则。
- PromptComposer 采用 `mode=chat`：不发送 RUN_DIRECTIVE/NODE_BRIEF；也不注入 agent persona（保持通用聊天），但会发送 ToolPolicy（默认开启所有可用 tools；由 Settings 的 SystemToolPolicy 控制系统级开关/限额，agent.tools 可进一步收紧）。

### 3.2 工具调用与回填的“硬规则”

当 assistant 返回 `tool_calls` 时：

1. Runtime **必须**按 `tool_calls[i].id` 逐个执行工具（可串行；或并行但要保持回填顺序一致）。  
2. Runtime **必须**把每个 tool result 作为一条 `role="tool"` 的 message 追加到对话历史，并填写 `tool_call_id`。  
3. 下一次 `chat.completions.create()` 的 `messages[]` **必须包含**：
   - 上一条 assistant（含 tool_calls）
   - 对应的 tool messages（tool results）

这样模型才能可靠理解“我刚调用的工具返回了什么”。

## 4. 与 LLM 对话的内部数据结构（建议 TypeScript）

> 这些结构分两层：
> - **对外（OpenAI-compatible）**：最终发给 Provider 的 request/response 格式。
> - **对内（ExecutionEngine）**：为了可恢复、可审计、可调试，Runtime 自己记录的结构化事件。

### 4.1 对外：OpenAI-compatible 请求/响应（最小字段）

```ts
export type OpenAIChatMessage =
  | { role: 'system'; content: string }
  | { role: 'user'; content: string }
  | { role: 'assistant'; content: string | null; tool_calls?: OpenAIToolCall[] }
  | { role: 'tool'; tool_call_id: string; content: string };

export interface OpenAIToolCall {
  id: string;
  type: 'function';
  function: { name: string; arguments: string }; // JSON string
}

export interface OpenAIFunctionTool {
  type: 'function';
  function: {
    name: string;
    description?: string;
    parameters: Record<string, unknown>; // JSON Schema
  };
}

export interface ChatCompletionsRequest {
  model: string;
  messages: OpenAIChatMessage[];
  tools?: OpenAIFunctionTool[];
  tool_choice?: 'auto' | 'none' | { type: 'function'; function: { name: string } };
  temperature?: number;
  stream?: boolean;
}
```

**关键点**：
- `tool` message 的 `content` 必须是字符串；建议统一放 **JSON 序列化后的 ToolResult**（见 6）。
- 即使你内部使用对象传递 tool result，对外发送时也要 `JSON.stringify(result)`。

> 注：`create-story-micro` 对照脚本为可读性使用 `templateRef` 标注 system 片段。真实发送给 Provider 时，`messages[]` 必须是“已展开的 content”；`templateRef` 只保留在内部日志即可。

### 4.2 对内：ExecutionEngine 运行记录（对照 trace）

```ts
export type RunPhase = 'Running' | 'WaitingUser' | 'Completed' | 'Failed';

export interface EngineTurn {
  id: string;                        // e.g. C01, C02...（对照 trace）
  phaseBefore: RunPhase;
  request: {
    messages: OpenAIChatMessage[];    // 最终发送给 LLM 的 messages
    tools: OpenAIFunctionTool[];      // 本轮暴露的工具（按 agent policy 过滤）
  };
  response: {
    assistant: OpenAIChatMessage & { role: 'assistant' };
  };
  toolRuns?: Array<{
    toolCallId: string;
    toolName: string;
    args: unknown;
    result: ToolResult;
    durationMs: number;
  }>;
  uiAfter?: { phase: RunPhase; stopReason?: string };
}
```

## 5. “可机器解析”的 user 内容：RUN_DIRECTIVE / NODE_BRIEF / USER_INPUT

`create-story-micro` 的对照脚本使用了 **纯文本壳**（LLM 既能读懂、也便于人类 debug）。建议保持一致：

### 5.1 RUN_DIRECTIVE

```text
RUN_DIRECTIVE
- runType: bmad-micro
- intent: start | continue | resume
- workflow: <workflowId or workflow.md path>
- state: @state/workflow.md
- graph: @pkg/workflow.graph.json
- artifactsRoot: @project/artifacts/
- currentNodeId: <from @state/workflow.md>
- effectiveAgentId: <node.agentId ?? activeAgentId>
- autopilot: true | false
```

### 5.2 NODE_BRIEF

```text
NODE_BRIEF
- currentNodeId: step-01-select-story
- stepFile: @pkg/steps/step-01-select-story.md
- outputsMap:
  - artifacts/create-story/target.md -> @project/artifacts/create-story/target.md
- allowedNext:
  - step-02-discover-inputs (label=next)
```

### 5.3 USER_INPUT

```text
USER_INPUT
- forNodeId: step-01-select-story
<raw user text>
```

- `mode=run`（workflow 流程中）建议包含 `- forNodeId: <currentNodeId>`，用于把用户回答绑定到当前节点。
- `mode=agent/chat`（非流程对话）不需要绑定节点：省略 `forNodeId` 行，直接附上 `<raw user text>` 即可。

## 6. ToolHost `fs.*`：工具返回结构（建议统一 ToolResult）

### 6.1 统一错误结构

```ts
export interface ToolError {
  code:
    | 'ENOENT'
    | 'E_SANDBOX_VIOLATION'
    | 'E_READ_LIMIT'
    | 'E_WRITE_LIMIT'
    | 'E_INVALID_FRONTMATTER'
    | 'E_SCHEMA_VALIDATION'
    | 'E_INVALID_TRANSITION'
    | 'E_PRECONDITION_FAILED'
    | 'E_INTERNAL';
  message: string;
  details?: Record<string, unknown>;
}
```

### 6.2 `fs.read`

```ts
export type FsReadResult =
  | { ok: true; path: string; bytes: number; sha256: string; truncated: false; content: string }
  | { ok: true; path: string; bytes: number; sha256: string; truncated: true; contentPreview: string; hint: string }
  | { ok: false; error: ToolError };
```

### 6.3 `fs.search`

```ts
export type FsSearchResult =
  | {
      ok: true;
      matches: Array<{
        path: string;
        line: number;
        column?: number;
        text: string;
        before?: string[];
        after?: string[];
      }>;
      truncated?: boolean;
      stats?: { filesScanned?: number; matchesFound?: number };
    }
  | { ok: false; error: ToolError };
```

### 6.4 `fs.list`

```ts
export type FsListResult =
  | { ok: true; path: string; entries: string[]; truncated?: false }
  | { ok: true; path: string; entriesPreview: string[]; truncated: true; hint?: string }
  | { ok: false; error: ToolError };
```

### 6.5 `fs.write`

```ts
export type FsWriteResult =
  | { ok: true; path: string; bytesWritten: number; sha256After?: string }
  | { ok: false; error: ToolError };
```

### 6.6 `fs.apply_patch`（建议支持 `updateFrontmatter`）

```ts
export interface UpdateFrontmatterOperation {
  operation: 'updateFrontmatter';
  update: {
    stepsCompleted?: { append: string[] };
    variables?: { set: Record<string, unknown> };
    decisionLog?: { append: Array<Record<string, unknown>> };
    artifacts?: { append: string[] };
    currentNodeId?: { set: string };
    updatedAt?: { set: string };
  };
  ifMatchSha256?: string;
}
```

建议在 Runtime 内部把 tool result 收敛成一个联合类型，便于日志与 UI 消费：

```ts
export type FsApplyPatchResult =
  | { ok: true; path: string; sha256Before?: string; sha256After?: string }
  | { ok: false; error: ToolError };

export type ToolResult = FsReadResult | FsSearchResult | FsListResult | FsWriteResult | FsApplyPatchResult;
```

## 7. “如何把 tool 结果放进下一次 Request”（落地规则）

1) 追加上一轮 assistant（含 `tool_calls`）  
2) 追加每个 `role="tool"` 回填（`tool_call_id` + `JSON.stringify(result)`）  
3) 继续调用 LLM（内部 loop），直到进入 `WaitingUser` 或 `Completed`

## 8. 额外提示（DeepSeek）

DeepSeek 是 OpenAI-compatible，但如果启用某些“思考/推理”模式，Provider 可能要求你在下一次请求中回传上一次 assistant 的推理字段。  
建议在内部 `EngineTurn` 里保留 provider 原始 response 字段，以便在需要时回填。
