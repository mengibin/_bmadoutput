# Tech Spec: Story 5-13 – Real-Time LLM Streaming Backend (SSE + IPC)

## 摘要
为 runtime 增加 **真实流式输出**：通过 SSE 解析 token 增量，转发为 Electron IPC 事件，让 UI 实时渲染，同时保持工具调用、持久化与错误回退兼容。

---

## 1. 范围与非目标

### 1.1 范围
- 支持 OpenAI / OpenAI-compatible 的 streaming（`stream: true`）。
- 在 **Run** 和 **Chat** 两条路径都可流式输出。
- 兼容工具调用（tool start/end 指示器事件）。
- 失败/不支持时自动回退到非流式。

### 1.2 非目标
- 不改变 UI 组件或消息持久化 schema（前端已能监听）。
- 不扩展 widgets（5.12 覆盖）。
- 暂不保证 Azure / Ollama streaming（可后续扩展）。

---

## 2. Provider 支持矩阵

| Provider | Streaming | 说明 |
|---|---|---|
| `openai` | ✅ | 直接支持 SSE |
| `openai-compatible` | ✅ | 兼容 SSE（需容错） |
| `azure` | ⛔（本期默认关闭） | 端点/参数不同，先回退非流式 |
| `ollama` | ⛔（本期默认关闭） | 流式协议不同，先回退非流式 |

> **回退策略**：非支持或解析异常 → 自动使用 `stream: false` 重试一次。

---

## 3. Streaming API 设计

### 3.1 LLM Adapter 接口
在 `llmAdapter.chatCompletions` 增加可选回调：

```ts
type LlmStreamEvent =
  | { type: 'start'; messageId: string }
  | { type: 'delta'; messageId: string; delta: string; partType?: 'content' | 'thinking' }
  | { type: 'end'; messageId: string }

type LlmStreamToolEvent =
  | { type: 'tool-start'; messageId: string; toolName: string }
  | { type: 'tool-end'; messageId: string; toolName: string; result?: unknown; duration?: number }

interface LlmAdapter {
  chatCompletions(params: {
    messages: OpenAIChatMessage[]
    tools?: OpenAIFunctionTool[]
    config: LlmAdapterConfig
    signal?: AbortSignal
    onStream?: (event: LlmStreamEvent) => void
  }): Promise<LlmAdapterResult>
}
```

> **注意**：tool-start/tool-end 事件由 `ExecutionEngine`/`chatToolLoop` 发出，不直接由 SSE 解析器触发。

---

## 4. SSE 解析与消息聚合

### 4.1 OpenAI SSE 数据格式
每行以 `data:` 开头，事件以空行分隔，终止标记为 `data: [DONE]`。

```text
data: {"choices":[{"delta":{"content":"hel"},"index":0,"finish_reason":null}]}
data: {"choices":[{"delta":{"content":"lo"},"index":0,"finish_reason":null}]}
data: {"choices":[{"delta":{},"finish_reason":"stop"}]}
data: [DONE]
```

### 4.2 解析策略
- 使用 `ReadableStream` reader 逐块读取。
- 维护 `buffer`，按 `\n\n` 分割事件。
- 对每个 `data:` JSON 解析：
  - `delta.content` → 触发 `onStream({type:'delta', delta})`
  - `delta.tool_calls` → **聚合** tool_calls（见 4.3）
  - `finish_reason` 记录终止条件
- 接收到 `[DONE]` → `onStream({type:'end'})`，并返回最终聚合消息。

### 4.3 tool_calls 聚合规则
OpenAI stream 可能分多段输出 `tool_calls`：
- 以 `tool_calls[index]` 合并
- `function.arguments` 逐段拼接字符串

伪代码：
```ts
for each delta.tool_calls[i]:
  const entry = toolCalls[i] ?? { id, type, function: { name: '', arguments: '' } }
  if (delta.function?.name) entry.function.name = delta.function.name
  if (delta.function?.arguments) entry.function.arguments += delta.function.arguments
  toolCalls[i] = entry
```

最终生成的 assistant message：
```ts
{
  role: 'assistant',
  content: aggregatedContent ?? null,
  tool_calls: aggregatedToolCalls?.length ? aggregatedToolCalls : undefined
}
```

---

## 5. IPC 事件协议

在 `electron/main.ts` 增加发送器（可按调用方窗口 sender 发送，必要时回退 broadcast）：

```ts
function sendIpcEvent(target: WebContents | undefined, channel: string, payload: unknown)
function createLlmStreamBroadcaster(target?: WebContents): {
  onStart(payload: { messageId: string }): void
  onChunk(payload: { messageId: string; delta: string; partType?: MessagePartType }): void
  onEnd(payload: { messageId: string }): void
  onToolStart(payload: { messageId: string; toolName: string }): void
  onToolEnd(payload: { messageId: string; toolName: string; result?: unknown; duration?: number }): void
}
```

并向对应窗口发送：
```
llm:stream-start
llm:stream-chunk
llm:stream-end
llm:stream-tool-start
llm:stream-tool-end
```

Renderer 已在 `preload.ts` 暴露监听 API，无需新增 UI 改动。

---

## 6. messageId 策略

**原则**：同一次 assistant 响应必须稳定一致，避免 UI 拼接错误。

建议格式：
- Run 模式：`run:{runId}:{turn}`
- Chat 模式：`chat:{conversationId}:{turn}`

由 `ExecutionEngine` / `chatToolLoop` 在发起 LLM 请求前生成并传入 `onStream`。

---

## 7. ExecutionEngine / chatToolLoop 集成

### 7.1 ExecutionEngine
1. 生成 `messageId`
2. 调用 `llmAdapter.chatCompletions({ onStream })`
3. `onStream` 触发 IPC broadcast
4. tool calls 执行前后广播 tool-start/tool-end
5. **Widget 兼容（Story 5.12）**：当 tool call 为 `ui.ask_user` 时：
   - 不执行常规 toolHost
   - 直接发 `widget:request`
   - 进入 `waiting-user`
5. 返回最终聚合 `assistant` message（用于持久化）

### 7.2 chatToolLoop
同上，确保 Chat 路径也能流式输出：
- 使用同样 `messageId` 策略（chat + turn）
- 工具调用也发 tool-start/tool-end
 - `ui.ask_user` 同样触发 `widget:request` 并暂停等待

---

## 8. 失败与回退语义

### 8.1 触发条件
- provider 不支持 streaming
- SSE 解析异常
- 超时或网络错误

### 8.2 行为
1. 终止当前流式请求（如有）
2. 重试一次非流式 `stream: false`
3. 返回完整 assistant message（不再发 delta）

### 8.3 IPC 语义
- 如果流式开始但失败，允许发 `llm:stream-end` 以清理 UI 状态。
- 如果直接回退到非流式，则不发 start/chunk，仅走最终消息更新（UI 兼容）。

---

## 9. 测试策略

### 9.1 单测（llmAdapter）
- 解析 SSE `delta.content`
- 解析 `delta.tool_calls` 拼接 arguments
- `[DONE]` 终止与最终消息聚合
- 流式失败回退到非流式

### 9.2 集成（executionEngine / chatToolLoop）
- 确认 `onStream` 顺序：start → chunk* → end
- tool-start/tool-end 在 tool 执行前后触发
- abort 后不再继续发送 chunk

---

## 10. 需要修改的文件

- `crewagent-runtime/electron/services/llmAdapter.ts`
- `crewagent-runtime/electron/services/openaiProtocol.ts`
- `crewagent-runtime/electron/services/executionEngine.ts`
- `crewagent-runtime/electron/services/chatToolLoop.ts`
- `crewagent-runtime/electron/main.ts`
- `crewagent-runtime/electron/preload.ts`（仅需确认暴露 API）
- `crewagent-runtime/electron/electron-env.d.ts`

---

## 11. 备注
- UI 已实现 `llm:stream-*` 监听（5-11），本故事重点在后端补齐真实流式能力。
- 设计应保持 **“失败可用”** 原则：流式失败不影响最终回复。
