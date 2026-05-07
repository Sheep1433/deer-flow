# SSE 悬空工具调用问题分析

## 问题背景

当用户在 AI Agent 对话过程中刷新页面时，如果 LLM 已经生成了 `tool_calls`（工具调用请求），但尚未收到对应的 `ToolMessage`（工具执行结果），消息历史就会出现"悬空"状态。这种不完整的消息格式会导致后续 LLM 调用出错。

## 各项目对比分析

### 1. DeerFlow（有防护）

**架构**：LangGraph Server（内置连接跟踪）

**SSE 配置**：
```python
stream_resumable: true          # 启用可恢复模式
on_disconnect: "cancel"          # 断开时取消执行
```

**防护机制**：`DanglingToolCallMiddleware`
- 检测 AIMessage 有 `tool_calls` 但缺少对应 ToolMessage 的情况
- 在悬空的 AIMessage 之后注入合成的错误 ToolMessage
- 使用 `wrap_model_call` 而非 `before_model`，确保补丁插入到正确位置

**中间件链顺序**（第4个）：
1. ThreadDataMiddleware
2. UploadsMiddleware
3. SandboxMiddleware
4. **DanglingToolCallMiddleware** ← 悬空工具调用修复
5. LLMErrorHandlingMiddleware
6. GuardrailMiddleware
7. SandboxAuditMiddleware
8. ToolErrorHandlingMiddleware
9. ...

### 2. TongWen（无防护）

**架构**：FastAPI + StreamingResponse

**SSE 配置**：无 streamResumable 配置，纯 Request/Response 模式

**中断处理**：
```python
# qa_service.py
except asyncio.CancelledError:
    task_cancelled = True
    logging.info(f"SSE连接已关闭，任务被取消: session_id={session_id}")
    raise
```

**前端**：`useSSEStream.ts`
- 使用 AbortController 管理请求生命周期
- 刷新页面 → 浏览器终止 fetch → 后端收到 CancelledError → yield abort

**风险**：
- ❌ 无 streamResumable，无法恢复中断
- ❌ 无悬空工具调用检测
- ❌ 如果中断时 LLM 已发 tool_calls，消息历史可能不完整

### 3. Aix-DB（无防护）

**架构**：Sanic + ResponseStream

**SSE 配置**：无 streamResumable，有 Keepalive（25秒）

**中断处理**：
```python
# enhanced_common_agent.py
async def _safe_write(self, response, content: str, message_type: str = "continue") -> bool:
    try:
        await response.write(...)
        return True
    except Exception as e:
        if self._is_connection_error(e):  # 检测连接断开
            logger.info(f"客户端连接已断开: {type(e).__name__}")
            return False
        raise

async def _stream_response(...):
    try:
        ...
    except asyncio.CancelledError:
        await self._safe_write(response, "\n> 这条消息已停止", "info")
        await self._safe_write(response, "", "end")
```

**其他机制**：`ToolCallManager`（循环检测）但无悬空工具调用检测

**风险**：
- ❌ 无 streamResumable，无法恢复中断
- ❌ 无悬空工具调用检测
- ✅ 有连接断开检测（_is_connection_error）

## 核心差异总结

| 特性 | DeerFlow | TongWen | Aix-DB |
|------|----------|---------|--------|
| 连接跟踪 | ✅ LangGraph 内置 | ❌ 无 | ❌ 无 |
| Keepalive | ✅ 依赖 LangGraph | ❌ 无 | ✅ 25秒间隔 |
| streamResumable | ✅ 有 | ❌ 无 | ❌ 无 |
| on_disconnect:cancel | ✅ 有 | ❌ 无 | ❌ 无 |
| 断开检测 | LangGraph 管理 | AbortController | _is_connection_error() |
| **悬空工具调用处理** | ✅ **有** | ❌ 无 | ❌ 无 |
| 循环检测 | LoopDetectionMiddleware | 无 | ToolCallManager |

## 下一步加固建议

### 对于 TongWen

1. **添加悬空工具调用检测中间件**
   ```python
   class DanglingToolCallMiddleware:
       # 参考 DeerFlow 实现
       def wrap_model_call(self, request, handler):
           patched = self._build_patched_messages(request.messages)
           if patched:
               request = request.override(messages=patched)
           return handler(request)
   ```

2. **添加 SSE 可恢复模式支持**
   - 记录当前执行状态（thread_id, checkpoint）
   - 断开时保存中断状态
   - 恢复时从 checkpoint 继续

3. **前端添加 beforeunload 处理**
   ```javascript
   window.addEventListener('beforeunload', () => {
       // 发送停止请求而非静默断开
   });
   ```

### 对于 Aix-DB

1. **添加悬空工具调用检测**
   - 在 `_stream_response` 结束后检查消息历史
   - 如果有悬空的 tool_calls，注入错误 ToolMessage

2. **添加 streamResumable 支持**
   - 使用 Sanic 的 checkpointer 机制
   - 保存执行状态到 InMemorySaver

3. **统一连接断开检测**
   - 现有 `_is_connection_error` 可继续使用
   - 考虑与 LangGraph checkpointer 集成

### 通用建议

1. **消息持久化策略**
   - 中断时不保存部分消息（避免不完整状态）
   - 恢复后重新执行完整流程
   - 或在恢复时先修复悬空工具调用

2. **前端体验优化**
   - 刷新前先调用 stop 接口
   - 提供"恢复对话"选项
   - 中断时显示明确的状态提示

3. **监控与日志**
   - 记录中断发生时的消息状态
   - 统计悬空工具调用发生频率
   - 告警异常中断模式

## 参考实现

DeerFlow 的 `DanglingToolCallMiddleware` 实现位置：
```
backend/packages/harness/deerflow/agents/middlewares/dangling_tool_call_middleware.py
```