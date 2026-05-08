# Task 工具改进

## 概述

改进了 task 工具，消除了浪费的 LLM 轮询。以前，使用后台任务时，LLM 必须反复调用 `task_status` 来轮询完成状态，导致不必要的 API 请求。

## 所做更改

### 1. 移除了 `run_in_background` 参数

`task` 工具的 `run_in_background` 参数已被移除。所有子代理任务默认异步运行，但工具自动处理完成。

**之前：**
```python
# LLM 需要管理轮询
task_id = task(
    subagent_type="bash",
    prompt="Run tests",
    description="Run tests",
    run_in_background=True
)
# 然后 LLM 需要反复轮询：
while True:
    status = task_status(task_id)
    if completed:
        break
```

**之后：**
```python
# 工具阻塞直到完成，后台轮询
result = task(
    subagent_type="bash",
    prompt="Run tests",
    description="Run tests"
)
# 调用返回后立即可用结果
```

### 2. 后台轮询

`task_tool` 现在：
- 异步启动子代理任务
- 在后台轮询完成状态（每 2 秒一次）
- 阻塞工具调用直到完成
- 直接返回最终结果

这意味着：
- ✅ LLM 只进行一次工具调用
- ✅ 无浪费的 LLM 轮询请求
- ✅ 后台处理所有状态检查
- ✅ 超时保护（最多 5 分钟）

### 3. 从 LLM 工具中移除了 `task_status`

`task_status_tool` 不再暴露给 LLM。它保留在代码库中供潜在的内部/调试使用，但 LLM 无法调用它。

### 4. 更新了文档

- 更新了 `prompt.py` 中的 `SUBAGENT_SECTION`，移除所有后台任务和轮询引用
- 简化了使用示例
- 明确说明工具会自动等待完成

## 实现细节

### 轮询逻辑

位于 `packages/harness/deerflow/tools/builtins/task_tool.py`：

```python
# 启动后台执行
task_id = executor.execute_async(prompt)

# 在后台轮询任务完成状态
while True:
    result = get_background_task_result(task_id)

    # 检查任务是否完成或失败
    if result.status == SubagentStatus.COMPLETED:
        return f"[Subagent: {subagent_type}]\n\n{result.result}"
    elif result.status == SubagentStatus.FAILED:
        return f"[Subagent: {subagent_type}] Task failed: {result.error}"

    # 等待下一次轮询
    time.sleep(2)

    # 超时保护（5 分钟）
    if poll_count > 150:
        return "Task timed out after 5 minutes"
```

### 执行超时

除了轮询超时外，子代理执行现在有内置的超时机制：

**配置**（`packages/harness/deerflow/subagents/config.py`）：
```python
@dataclass
class SubagentConfig:
    # ...
    timeout_seconds: int = 300  # 默认 5 分钟
```

**线程池架构**：

为避免嵌套线程池和资源浪费，我们使用两个专用线程池：

1. **调度池**（`_scheduler_pool`）：
   - 最大工作线程：4
   - 用途：编排后台任务执行
   - 运行管理任务生命周期的 `run_task()` 函数

2. **执行池**（`_execution_pool`）：
   - 最大工作线程：8（更大以避免阻塞）
   - 用途：带超时支持的实际子代理执行
   - 运行调用代理的 `execute()` 方法

**工作原理**：
```python
# 在 execute_async() 中：
_scheduler_pool.submit(run_task)  # 提交编排任务

# 在 run_task() 中：
future = _execution_pool.submit(self.execute, task)  # 提交执行
exec_result = future.result(timeout=timeout_seconds)  # 带超时等待
```

**好处**：
- ✅ 清晰的关注点分离（调度 vs 执行）
- ✅ 无嵌套线程池
- ✅ 在正确层面强制执行超时
- ✅ 更好的资源利用率

**两级超时保护**：
1. **执行超时**：子代理执行本身有 5 分钟超时（在 SubagentConfig 中可配置）
2. **轮询超时**：工具轮询有 5 分钟超时（30 次轮询 × 10 秒）

这确保即使子代理执行挂起，系统也不会无限等待。

### 好处

1. **降低 API 成本**：不再有重复的 LLM 轮询请求
2. **更简单的用户体验**：LLM 不需要管理轮询逻辑
3. **更好的可靠性**：后台一致处理所有状态检查
4. **超时保护**：两级超时防止无限等待（执行 + 轮询）

## 测试

验证更改是否正确工作：

1. 启动一个需要几秒钟的子代理任务
2. 验证工具调用阻塞直到完成
3. 验证结果直接返回
4. 验证没有进行 `task_status` 调用

示例测试场景：
```python
# 这应该阻塞约 10 秒然后返回结果
result = task(
    subagent_type="bash",
    prompt="sleep 10 && echo 'Done'",
    description="Test task"
)
# result 应该包含 "Done"
```

## 迁移说明

对于之前使用 `run_in_background=True` 的用户/代码：
- 只需移除该参数
- 移除任何轮询逻辑
- 工具会自动等待完成

无需其他更改——API 向后兼容（除了移除的参数）。
