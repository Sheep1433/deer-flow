# SSE 流式输出实现文档

## 1. 概述

DeerFlow 的 SSE 流式输出基于 **LangGraph SDK** 定义的标准 stream mode 协议。DeerFlow 作为适配层，将 LangGraph 的输出转换为 SSE 格式，通过 FastAPI StreamingResponse 返回给前端。

## 2. 架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│ Backend (Python)                                                     │
│                                                                      │
│  graph.astream(stream_mode=["values", "messages-tuple"])            │
│       ↓                                                              │
│  worker.py: serialize(chunk, mode)                                  │
│       ↓                                                              │
│  bridge.publish(run_id, event_type, data)                           │
│       ↓                                                              │
│  services.py: format_sse(event, data) → "event: messages\ndata:..." │
│       ↓                                                              │
│  FastAPI StreamingResponse                                           │
└─────────────────────────────────────────────────────────────────────┘
                            ↓ SSE
┌─────────────────────────────────────────────────────────────────────┐
│ Frontend (React/TypeScript)                                          │
│                                                                      │
│  useStream() hook from @langchain/langgraph-sdk                      │
│       ↓                                                              │
│  onLangChainEvent({ event: "on_tool_end", name, data })             │
│  onUpdateEvent(state)                                               │
│  onCustomEvent({ type: "task_running", ... })                       │
│       ↓                                                              │
│  MessageList → MessageGroup → ToolCall / Reasoning / ChainOfThought  │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. Stream Mode 定义

LangGraph SDK 定义了以下官方 stream mode：

| stream mode | 含义 | 输出格式 |
|---|---|---|
| `messages` | token 级流式输出 | `(AIMessageChunk, metadata)` 元组列表 |
| `messages-tuple` | 同 `messages`，别名 | 同上 |
| `values` | 完整状态快照 | 全部 channel_values |
| `updates` | 每个 node 执行完的快照 | `{node_name: writes}` |
| `custom` | 自定义数据 | `StreamWriter.write()` 的数据 |

## 4. SSE 事件类型

| SSE Event | 来源 | 内容 |
|---|---|---|
| `metadata` | `worker.py` line 159-166 | `run_id`, `thread_id` |
| `values` | LangGraph `values` 模式 | 完整 state snapshot（含所有 messages） |
| `messages` | LangGraph `messages` 模式 | token 级流式 AI 消息和 ToolMessage |
| `updates` | LangGraph `updates` 模式 | 每个 node 执行完的 `{node: writes}` |
| `custom` | `StreamWriter.write()` | 自定义应用事件 |
| `error` | `worker.py` 异常处理 | 错误信息 |
| `end` | `bridge.publish_end()` | 流终止标记 |

## 5. 核心文件说明

### 5.1 后端核心文件

**`backend/app/gateway/services.py`**
- `format_sse(event, data, event_id)` — 格式化 SSE 帧
  ```
  event: {event}
  data: {json.dumps(data)}
  id: {event_id}
  ```
- `sse_consumer(bridge, record, request, run_mgr)` — 异步生成器，从 StreamBridge 消费事件并格式化为 SSE
- `normalize_stream_modes(raw)` — 规范化 stream_mode 参数，默认 `["values"]`

**`backend/packages/harness/deerflow/runtime/runs/worker.py`**
- `run_agent()` — 在后台 asyncio 中执行 agent，通过 `bridge.publish()` 发布事件
- `_lg_mode_to_sse_event(mode)` — 把 LangGraph stream mode 直接映射为 SSE event 名称
- `serialize(chunk, mode)` — mode 感知的序列化

**`backend/packages/harness/deerflow/runtime/serialization.py`**
- `serialize_messages_tuple(obj)` — 序列化 `(chunk, metadata)` 元组
- `serialize_channel_values(channel_values)` — 序列化 state，剥离 `__pregel_*` 内部键

### 5.2 前端核心文件

**`frontend/src/core/threads/hooks.ts`**
- `useStream()` — `@langchain/langgraph-sdk` 的 hook，订阅 SSE 流
- `onLangChainEvent` — 监听 `on_tool_end` 事件
- `onUpdateEvent` — 处理 state 更新
- `onCustomEvent` — 处理自定义事件

**`frontend/src/core/messages/utils.ts`**
- `splitInlineReasoning(content)` — 从字符串 content 中正则提取 `<think>` 内容
- `extractReasoningContentFromMessage(message)` — 提取 thinking 内容（三优先級）
- `hasReasoning(message)` — 判断消息是否包含 thinking
- `findToolCallResult(toolCallId, messages)` — 通过 tool_call_id 查找工具结果

**`frontend/src/components/ai-elements/reasoning.tsx`**
- `Reasoning` 组件 — 可折叠的 thinking 展示，使用 Radix UI Collapsible

**`frontend/src/components/workspace/messages/message-group.tsx`**
- `ToolCall` 组件 — 根据工具名渲染不同 UI
- `convertToSteps(messages)` — 把 messages 转换为 CoT steps

## 6. Thinking 内容区分（标准 ChatOpenAI）

### 你的配置
```yaml
models:
  - name: minimax-m2.7
    use: langchain_openai:ChatOpenAI  # 标准 ChatOpenAI
    model: MiniMax-M2.7
```

### 区分方式

标准 ChatOpenAI **不处理** `reasoning_details`，MiniMax API 返回的 thinking 通过**两种途径**区分：

#### 途径 1：Inline `<think>` 标签（你的情况）

MiniMax API 返回的 `content` 字段包含 `<think>` 和 `</think>` 标签：

```
AIMessageChunk.content = "<think>用户想查看当前目录有什么文件...\n</think>\n\n"
AIMessageChunk.content = "<think>工具调用结束\n</think>\n\n"
AIMessageChunk.content = "好的，我来帮你列出目录内容...\n"
```

前端正则解析：
```typescript
const THINK_TAG_RE = /<think>\s*([\s\S]*?)\s*<\/think>/g;

function splitInlineReasoning(content: string) {
  const reasoningParts: string[] = [];
  const cleaned = content.replace(THINK_TAG_RE, (_, reasoning) => {
    reasoningParts.push(reasoning.trim());
    return "";
  }).trim();
  return {
    content: cleaned,
    reasoning: reasoningParts.join("\n\n") || null,
  };
}
```

#### 途径 2：`additional_kwargs.reasoning_content`（Patched Provider）

如果使用 `PatchedChatMiniMax` 或 `VllmChatModel`，thinking 存入 `additional_kwargs.reasoning_content`：

```typescript
export function extractReasoningContentFromMessage(message: Message) {
  // 优先从 additional_kwargs 取
  if (message.additional_kwargs?.reasoning_content) {
    return message.additional_kwargs.reasoning_content;
  }
  // 其次从 content 数组的 thinking 字段取
  if (Array.isArray(message.content)) {
    const part = message.content[0];
    if (part?.thinking) return part.thinking;
  }
  // 最后正则匹配
  return splitInlineReasoning(message.content).reasoning;
}
```

## 7. 工具调用与结果输出

### 7.1 传输方式

工具调用和结果**不是一次性输出**，而是分开传输：

```
messages 事件（AI 消息流）：
  id 14: AIMessageChunk { tool_calls: [{name: "ls", id: "call_xxx"}] }
  id 15: AIMessageChunk { tool_calls: [{name: "", args: {...}}] }  ← args 在下一个 chunk
  id 16: AIMessageChunk { finish_reason: "tool_calls" }
  id 17: AIMessageChunk { usage_metadata }
  ...
  
messages 事件（工具结果）：
  id 23: ToolMessage { type: "tool", tool_call_id: "call_xxx", content: "(empty)" }
```

### 7.2 关联方式

前端通过 `tool_call.id` 和 `tool_call_id` 匹配：

```typescript
function convertToSteps(messages: Message[]): CoTStep[] {
  for (const message of messages) {
    if (message.type === "ai") {
      for (const tool_call of message.tool_calls ?? []) {
        const step: CoTToolCallStep = {
          id: tool_call.id,
          name: tool_call.name,
          args: tool_call.args,
        };
        // 通过 tool_call.id 查找 ToolMessage 结果
        const toolCallResult = findToolCallResult(tool_call.id, messages);
        step.result = toolCallResult;
      }
    }
  }
}
```

### 7.3 事件对照表

从实际 SSE 日志：

| SSE id | event | 说明 |
|---|---|---|
| 14 | messages | AI chunk 含 `tool_calls: [{name: "ls", ...}]` |
| 15 | messages | AI chunk args 补充 |
| 16 | messages | `finish_reason: "tool_calls"` |
| 17 | messages | `usage_metadata` |
| 19 | updates | `model` node 完成快照 |
| 23 | messages | `type: "tool"` → 工具结果 |
| 24 | updates | `tools` node 完成快照 |
| 25 | values | 全量 state（含 tool result） |

## 8. 前端 Collapsible 实现

### 8.1 Reasoning 组件

```tsx
<Collapsible open={isOpen}>
  <CollapsibleTrigger>思考过程</CollapsibleTrigger>
  <CollapsibleContent>
    <Streamdown>{children}</Streamdown>
  </CollapsibleContent>
</Collapsible>
```

### 8.2 自动折叠逻辑

```tsx
useEffect(() => {
  if (defaultOpen && !isStreaming && isOpen && !hasAutoClosed) {
    const timer = setTimeout(() => {
      setIsOpen(false);
      setHasAutoClosed(true);
    }, AUTO_CLOSE_DELAY); // 1000ms
    return () => clearTimeout(timer);
  }
}, [isStreaming, isOpen, defaultOpen]);
```

- 流式开始时自动展开
- 流式结束 1 秒后自动折叠

## 9. SSE 格式化函数

```python
# services.py line 43-56
def format_sse(event: str, data: Any, *, event_id: str | None = None) -> str:
    payload = json.dumps(data, default=str, ensure_ascii=False)
    parts = [f"event: {event}", f"data: {payload}"]
    if event_id:
        parts.append(f"id: {event_id}")
    parts.append("")
    parts.append("")
    return "\n".join(parts)
```

输出格式：
```
event: messages
data: [{"content": "...", "type": "AIMessageChunk", ...}, {...}]
id: 1778216464274-8


```

## 10. 总结

| 组件 | 职责 | 关键函数 |
|---|---|---|
| `worker.py` | LangGraph 执行 + 事件发布 | `run_agent()`, `_lg_mode_to_sse_event()` |
| `serialization.py` | LangChain 对象序列化 | `serialize()`, `serialize_messages_tuple()` |
| `services.py` | SSE 格式化 + 消费 | `format_sse()`, `sse_consumer()` |
| `hooks.ts` | 前端 SSE 订阅 | `useStream()` |
| `utils.ts` | Thinking/Tool 提取 | `extractReasoningContentFromMessage()` |
| `message-group.tsx` | 消息渲染 | `ToolCall()`, `convertToSteps()` |
| `reasoning.tsx` | Thinking 折叠 | `Reasoning` 组件 |
