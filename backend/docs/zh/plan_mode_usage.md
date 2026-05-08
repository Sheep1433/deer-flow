# TodoList 中间件的计划模式

本文档描述如何在 DeerFlow 2.0 中启用和使用带 TodoList 中间件的计划模式功能。

## 概述

计划模式为代理添加了 TodoList 中间件，提供 `write_todos` 工具，帮助代理：
- 将复杂任务分解为更小、可管理的步骤
- 随着工作进展跟踪进度
- 为用户提供正在做什么的可见性

TodoList 中间件建立在 LangChain 的 `TodoListMiddleware` 之上。

## 配置

### 启用计划模式

计划模式通过 `RunnableConfig` 的 `configurable` 部分中的 `is_plan_mode` 参数进行**运行时配置**。这允许您基于每个请求动态启用或禁用计划模式。

```python
from langchain_core.runnables import RunnableConfig
from deerflow.agents.lead_agent.agent import make_lead_agent

# 通过运行时配置启用计划模式
config = RunnableConfig(
    configurable={
        "thread_id": "example-thread",
        "thinking_enabled": True,
        "is_plan_mode": True,  # 启用计划模式
    }
)

# 创建启用计划模式的代理
agent = make_lead_agent(config)
```

### 配置选项

- **is_plan_mode**（bool）：是否启用带 TodoList 中间件的计划模式。默认：`False`
  - 通过 `config.get("configurable", {}).get("is_plan_mode", False)` 传递
  - 可以为每个代理调用动态设置
  - 无需全局配置

## 默认行为

使用默认设置启用计划模式时，代理将可以访问具有以下行为的 `write_todos` 工具：

### 何时使用 TodoList

代理将使用待办列表用于：
1. 复杂的多步骤任务（3 个或更多独立步骤）
2. 需要仔细计划的非平凡任务
3. 用户明确请求待办列表时
4. 用户提供多个任务时

### 何时不使用 TodoList

代理将跳过使用待办列表用于：
1. 简单、直接的任务
2. 平凡任务（< 3 步）
3. 纯对话或信息请求

### 任务状态

- **pending**：任务尚未开始
- **in_progress**：正在进行（可以有多个并行任务）
- **completed**：任务成功完成

## 使用示例

### 基本用法

```python
from langchain_core.runnables import RunnableConfig
from deerflow.agents.lead_agent.agent import make_lead_agent

# 创建启用计划模式的代理
config_with_plan_mode = RunnableConfig(
    configurable={
        "thread_id": "example-thread",
        "thinking_enabled": True,
        "is_plan_mode": True,  # TodoList 中间件将被添加
    }
)
agent_with_todos = make_lead_agent(config_with_plan_mode)

# 创建禁用计划模式的代理（默认）
config_without_plan_mode = RunnableConfig(
    configurable={
        "thread_id": "another-thread",
        "thinking_enabled": True,
        "is_plan_mode": False,  # 无 TodoList 中间件
    }
)
agent_without_todos = make_lead_agent(config_without_plan_mode)
```

### 每个请求动态计划模式

您可以动态地为不同对话或任务启用/禁用计划模式：

```python
from langchain_core.runnables import RunnableConfig
from deerflow.agents.lead_agent.agent import make_lead_agent

def create_agent_for_task(task_complexity: str):
    """根据任务复杂度创建具有计划模式的代理。"""
    is_complex = task_complexity in ["high", "very_high"]

    config = RunnableConfig(
        configurable={
            "thread_id": f"task-{task_complexity}",
            "thinking_enabled": True,
            "is_plan_mode": is_complex,  # 仅对复杂任务启用
        }
    )

    return make_lead_agent(config)

# 简单任务 - 不需要 TodoList
simple_agent = create_agent_for_task("low")

# 复杂任务 - 启用 TodoList 以便更好地跟踪
complex_agent = create_agent_for_task("high")
```

## 工作原理

1. 调用 `make_lead_agent(config)` 时，它从 `config.configurable` 中提取 `is_plan_mode`
2. 配置被传递给 `_build_middlewares(config)`
3. `_build_middlewares()` 读取 `is_plan_mode` 并调用 `_create_todo_list_middleware(is_plan_mode)`
4. 如果 `is_plan_mode=True`，则创建 `TodoListMiddleware` 实例并添加到中间件链中
5. 中间件自动将 `write_todos` 工具添加到代理的工具集中
6. 代理可以在执行过程中使用此工具管理任务
7. 中间件处理待办列表状态并将其提供给代理

## 架构

```
make_lead_agent(config)
  │
  ├─> 提取：is_plan_mode = config.configurable.get("is_plan_mode", False)
  │
  └─> _build_middlewares(config)
        │
        ├─> ThreadDataMiddleware
        ├─> SandboxMiddleware
        ├─> SummarizationMiddleware（如果通过全局配置启用）
        ├─> TodoListMiddleware（如果 is_plan_mode=True）← 新增
        ├─> TitleMiddleware
        └─> ClarificationMiddleware
```

## 实现细节

### 代理模块
- **位置**：`packages/harness/deerflow/agents/lead_agent/agent.py`
- **函数**：`_create_todo_list_middleware(is_plan_mode: bool)` - 如果计划模式启用则创建 TodoListMiddleware
- **函数**：`_build_middlewares(config: RunnableConfig)` - 基于运行时配置构建中间件链
- **函数**：`make_lead_agent(config: RunnableConfig)` - 创建具有适当中间件的代理

### 运行时配置
计划模式通过 `RunnableConfig.configurable` 中的 `is_plan_mode` 参数控制：
```python
config = RunnableConfig(
    configurable={
        "is_plan_mode": True,  # 启用计划模式
        # ... 其他可配置选项
    }
)
```

## 主要好处

1. **动态控制**：无需全局状态即可基于每个请求启用/禁用计划模式
2. **灵活性**：不同对话可以有不同的计划模式设置
3. **简单性**：无需全局配置管理
4. **上下文感知**：计划模式决策可以基于任务复杂度、用户偏好等

## 自定义提示

DeerFlow 使用自定义的 `system_prompt` 和 `tool_description` 用于 TodoListMiddleware，与 DeerFlow 整体提示风格一致：

### 系统提示特点
- 使用 XML 标签（`<todo_list_system>`）与 DeerFlow 主提示的结构一致性
- 强调关键规则和最佳实践
- 清晰的"何时使用"与"何时不使用"指南
- 专注于实时更新和立即完成任务

### 工具描述特点
- 详细的使用场景和示例
- 强烈强调不用于简单任务
- 清晰的任务状态定义（pending、in_progress、completed）
- 全面的最佳实践部分
- 任务完成要求以防止过早标记

自定义提示在 `/Users/hetao/workspace/deer-flow/backend/packages/harness/deerflow/agents/lead_agent/agent.py:57` 的 `_create_todo_list_middleware()` 中定义。

## 注意事项

- TodoList 中间件使用 LangChain 内置的 `TodoListMiddleware`，但使用**自定义 DeerFlow 风格提示**
- 计划模式**默认禁用**（`is_plan_mode=False`）以保持向后兼容性
- 中间件位于 `ClarificationMiddleware` 之前，以允许在澄清流程中进行任务管理
- 自定义提示强调与 DeerFlow 主系统提示相同的原则（清晰、面向行动、关键规则）
