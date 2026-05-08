# 对话摘要

DeerFlow 包含自动对话摘要功能，用于处理接近模型 token 限制的长对话。启用后，系统会自动压缩旧消息，同时保留最近的上下文。

## 概述

摘要功能使用 LangChain 的 `SummarizationMiddleware` 来监控对话历史并基于可配置的阈值触发摘要。当激活时，它会：

1. 实时监控消息 token 计数
2. 当达到阈值时触发摘要
3. 保留最近消息的同时压缩旧对话
4. 保持 AI/Tool 消息对的完整性以确保上下文连续性
5. 将摘要注入对话中

## 配置

摘要功能在 `config.yaml` 的 `summarization` 键下配置：

```yaml
summarization:
  enabled: true
  model_name: null  # 使用默认模型或指定轻量级模型

  # 触发条件（OR 逻辑 - 任何条件触发摘要）
  trigger:
    - type: tokens
      value: 4000
    # 额外的触发器（可选）
    # - type: messages
    #   value: 50
    # - type: fraction
    #   value: 0.8  # 模型最大输入 token 的 80%

  # 上下文保留策略
  keep:
    type: messages
    value: 20

  # 摘要调用的 token 修剪
  trim_tokens_to_summarize: 4000

  # 自定义摘要提示（可选）
  summary_prompt: null

  # 视为技能文件读取的工具名称，用于技能救援
  skill_file_read_tool_names:
    - read_file
    - read
    - view
    - cat
```

### 配置选项

#### `enabled`
- **类型**：布尔值
- **默认值**：`false`
- **描述**：启用或禁用自动摘要

#### `model_name`
- **类型**：字符串或 null
- **默认值**：`null`（使用默认模型）
- **描述**：用于生成摘要的模型。建议使用轻量级、成本效益高的模型，如 `gpt-4o-mini` 或同类模型。

#### `trigger`
- **类型**：单个 `ContextSize` 或 `ContextSize` 对象列表
- **必需**：启用时必须至少指定一个触发器
- **描述**：触发摘要的阈值。使用 OR 逻辑 - 当**任何**阈值满足时运行摘要。

**ContextSize 类型：**

1. **基于 token 的触发器**：当 token 计数达到指定值时激活
   ```yaml
   trigger:
     type: tokens
     value: 4000
   ```

2. **基于消息的触发器**：当消息计数达到指定值时激活
   ```yaml
   trigger:
     type: messages
     value: 50
   ```

3. **基于比例的触发器**：当 token 使用达到模型最大输入 token 的百分比时激活
   ```yaml
   trigger:
     type: fraction
     value: 0.8  # 最大输入 token 的 80%
   ```

**多个触发器：**
```yaml
trigger:
  - type: tokens
    value: 4000
  - type: messages
    value: 50
```

#### `keep`
- **类型**：`ContextSize` 对象
- **默认值**：`{type: messages, value: 20}`
- **描述**：指定摘要后保留多少最近的对话历史。

**示例：**
```yaml
# 保留最近的 20 条消息
keep:
  type: messages
  value: 20

# 保留最近的 3000 个 token
keep:
  type: tokens
  value: 3000

# 保留模型最大输入 token 的最近 30%
keep:
  type: fraction
  value: 0.3
```

#### `trim_tokens_to_summarize`
- **类型**：整数或 null
- **默认值**：`4000`
- **描述**：准备摘要调用的消息时包含的最大 token 数。设置为 `null` 以跳过修剪（不推荐用于非常长的对话）。

#### `summary_prompt`
- **类型**：字符串或 null
- **默认值**：`null`（使用 LangChain 的默认提示）
- **描述**：用于生成摘要的自定义提示模板。提示应指导模型提取最重要的上下文。

#### `preserve_recent_skill_count`
- **类型**：整数（≥ 0）
- **默认值**：`5`
- **描述**：从摘要中救援的最进 recently-loaded 技能文件（工具结果，其工具名称在 `skill_file_read_tool_names` 中且目标路径在 `skills.container_path` 下，例如 `/mnt/skills/...`）的数量。防止 agent 在压缩后丢失技能说明。将设置为 `0` 可完全禁用技能救援。

#### `preserve_recent_skill_tokens`
- **类型**：整数（≥ 0）
- **默认值**：`25000`
- **描述**：为技能读取保留的 token 总预算。一旦此预算耗尽，较旧的技能捆绑包将被允许进入摘要。

#### `preserve_recent_skill_tokens_per_skill`
- **类型**：整数（≥ 0）
- **默认值**：`5000`
- **描述**：每个技能的 token 上限。任何工具结果超过此大小的单个技能读取都不会被救援（它会像普通内容一样进入摘要器）。

#### `skill_file_read_tool_names`
- **类型**：字符串列表
- **默认值**：`["read_file", "read", "view", "cat"]`
- **描述**：在摘要救援期间视为技能文件读取的工具名称。只有当工具名称在此列表中且其目标路径在 `skills.container_path` 下时，工具调用才有资格进行技能救援。

**默认提示行为：**
默认的 LangChain 提示指示模型：
- 提取最高质量/最相关的上下文
- 专注于对整体目标至关重要的信息
- 避免重复已完成的操作
- 仅返回提取的上下文

## 工作原理

### 摘要流程

1. **监控**：在每次模型调用之前，中间件计算消息历史中的 token
2. **触发检查**：如果满足任何配置的阈值，则触发摘要
3. **消息分区**：消息被拆分：
   - 要摘要的消息（超过 `keep` 阈值的旧消息）
   - 要保留的消息（在 `keep` 阈值内的最近消息）
4. **摘要生成**：模型生成旧消息的简洁摘要
5. **上下文替换**：消息历史更新：
   - 删除所有旧消息
   - 添加单个摘要消息
   - 保留最近消息
6. **AI/Tool 对保护**：系统确保 AI 消息及其对应的工具消息保持在一起
7. **技能救援**：在生成摘要之前，最近加载的技能文件（工具结果，其工具名称在 `skill_file_read_tool_names` 中且目标路径在 `skills.container_path` 下）被从摘要集中提升出来，并预先添加到保留的末尾。选择在三个预算下 newest-first 遍历：`preserve_recent_skill_count`、`preserve_recent_skill_tokens` 和 `preserve_recent_skill_tokens_per_skill`。触发的 AIMessage 及其配对的 ToolMessages 一起移动，以保持 tool_call ↔ tool_result 配对完整。

### Token 计数

- 使用基于字符数的近似 token 计数
- 对于 Anthropic 模型：每个 token 约 3.3 个字符
- 对于其他模型：使用 LangChain 的默认估计
- 可以使用自定义 `token_counter` 函数进行自定义

### 消息保留

中间件智能地保留消息上下文：

- **最近消息**：始终根据 `keep` 配置保持完整
- **AI/Tool 对**：永不拆分 - 如果截止点落在工具消息中，系统会调整以保持整个 AI + Tool 消息序列在一起
- **摘要格式**：摘要作为 HumanMessage 注入，格式为：
  ```
  Here is a summary of the conversation to date:

  [Generated summary text]
  ```

## 最佳实践

### 选择触发阈值

1. **基于 token 的触发器**：推荐用于大多数用例
   - 设置为模型上下文窗口的 60-80%
   - 示例：对于 8K 上下文，使用 4000-6000 个 token

2. **基于消息的触发器**：用于控制对话长度
   - 适用于有许多短消息的应用程序
   - 示例：根据平均消息长度使用 50-100 条消息

3. **基于比例的触发器**：理想情况是使用多个模型
   - 自动适应每个模型的容量
   - 示例：0.8（模型最大输入 token 的 80%）

### 选择保留策略（`keep`）

1. **基于消息的保留**：最适合大多数场景
   - 保留自然对话流程
   - 推荐：15-25 条消息

2. **基于 token 的保留**：需要精确控制时使用
   - 适用于管理精确的 token 预算
   - 推荐：2000-4000 个 token

3. **基于比例的保留**：用于多模型设置
   - 自动随模型容量缩放
   - 推荐：0.2-0.4（最大输入的 20-40%）

### 模型选择

- **推荐**：使用轻量级、成本效益高的模型进行摘要
  - 示例：`gpt-4o-mini`、`claude-haiku` 或同类模型
  - 摘要不需要最强大的模型
  - 在高流量应用程序上可显著节省成本

- **默认**：如果 `model_name` 为 `null`，则使用默认模型
  - 可能更昂贵但确保一致性
  - 适用于简单设置

### 优化技巧

1. **平衡触发器**：结合 token 和消息触发器以获得稳健的处理
   ```yaml
   trigger:
     - type: tokens
       value: 4000
     - type: messages
       value: 50
   ```

2. **保守保留**：最初保留更多消息，根据性能进行调整
   ```yaml
   keep:
     type: messages
     value: 25  # 开始更高，需要时减少
   ```

3. **策略性修剪**：限制发送到摘要模型的 token
   ```yaml
   trim_tokens_to_summarize: 4000  # 防止昂贵的摘要调用
   ```

4. **监控和迭代**：跟踪摘要质量并调整配置

## 故障排查

### 摘要质量问题

**问题**：摘要丢失重要上下文

**解决方案**：
1. 增加 `keep` 值以保留更多消息
2. 减少触发阈值以更早进行摘要
3. 自定义 `summary_prompt` 以强调关键信息
4. 使用更有能力的模型进行摘要

### 性能问题

**问题**：摘要调用花费时间过长

**解决方案**：
1. 使用更快的模型进行摘要（例如 `gpt-4o-mini`）
2. 减少 `trim_tokens_to_summarize` 以发送更少的上下文
3. 增加触发阈值以减少摘要频率

### Token 限制错误

**问题**：尽管有摘要仍然达到 token 限制

**解决方案**：
1. 降低触发阈值以更早进行摘要
2. 减少 `keep` 值以保留更少的消息
3. 检查个别消息是否非常大
4. 考虑使用基于比例的触发器

## 实现细节

### 代码结构

- **配置**：`packages/harness/deerflow/config/summarization_config.py`
- **集成**：`packages/harness/deerflow/agents/lead_agent/agent.py`
- **中间件**：使用 `langchain.agents.middleware.SummarizationMiddleware`

### 中间件顺序

摘要在线程数据和沙箱初始化之后但在标题和澄清之前运行：

1. ThreadDataMiddleware
2. SandboxMiddleware
3. **SummarizationMiddleware** ← 在此处运行
4. TitleMiddleware
5. ClarificationMiddleware

### 状态管理

- 摘要是无状态的 - 配置在启动时加载一次
- 摘要作为对话历史中的常规消息添加
- 检查点自动持久化摘要的历史

## 示例配置

### 最小配置
```yaml
summarization:
  enabled: true
  trigger:
    type: tokens
    value: 4000
  keep:
    type: messages
    value: 20
```

### 生产配置
```yaml
summarization:
  enabled: true
  model_name: gpt-4o-mini  # 使用轻量级模型以节省成本
  trigger:
    - type: tokens
      value: 6000
    - type: messages
      value: 75
  keep:
    type: messages
    value: 25
  trim_tokens_to_summarize: 5000
```

### 多模型配置
```yaml
summarization:
  enabled: true
  model_name: gpt-4o-mini
  trigger:
    type: fraction
    value: 0.7  # 模型最大输入的 70%
  keep:
    type: fraction
    value: 0.3  # 保留最大输入的 30%
  trim_tokens_to_summarize: 4000
```

### 保守配置（高质量）
```yaml
summarization:
  enabled: true
  model_name: gpt-4  # 使用完整模型以获得高质量摘要
  trigger:
    type: tokens
    value: 8000
  keep:
    type: messages
    value: 40  # 保留更多上下文
  trim_tokens_to_summarize: null  # 不修剪
```

## 参考资料

- [LangChain 摘要中间件文档](https://docs.langchain.com/oss/python/langchain/middleware/built-in#summarization)
- [LangChain 源代码](https://github.com/langchain-ai/langchain)
