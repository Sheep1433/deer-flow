# Guardrails：工具调用前授权

> **背景**：[Issue #1213](https://github.com/bytedance/deer-flow/issues/1213) — DeerFlow 有 Docker 沙箱和通过 `ask_clarification` 的人工审批，但没有针对工具调用的确定性、策略驱动的授权层。运行自主多步任务的 agent 可以使用任何参数执行任何已加载的工具。Guardrails 添加了一个中间件，在执行**之前**根据策略评估每个工具调用。

## 为什么需要 Guardrails

```
没有 guardrails：                      有 guardrails：

  Agent                                    Agent
    │                                        │
    ▼                                        ▼
  ┌──────────┐                             ┌──────────┐
  │ bash     │──▶ 立即执行                    │ bash     │──▶ GuardrailMiddleware
  │ rm -rf / │                             │ rm -rf / │        │
  └──────────┘                             └──────────┘        ▼
                                                         ┌──────────────┐
                                                         │  Provider    │
                                                         │  根据策略    │
                                                         │  评估        │
                                                         └──────┬───────┘
                                                                │
                                                          ┌─────┴─────┐
                                                          │           │
                                                        ALLOW       DENY
                                                          │           │
                                                          ▼           ▼
                                                      工具正常     Agent 看到：
                                                      运行        "Guardrail denied:
                                                                   rm -rf blocked"
```

- **沙箱**提供进程隔离但不提供语义授权。沙箱化的 `bash` 仍然可以 `curl` 数据出去。
- **人工审批**（`ask_clarification`）需要有人在循环中处理每个操作。不适合自主工作流。
- **Guardrails** 提供确定性、策略驱动的授权，无需人工干预即可工作。

## 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Middleware Chain                               │
│                                                                      │
│  1. ThreadDataMiddleware     ─── per-thread dirs                     │
│  2. UploadsMiddleware        ─── 文件上传跟踪                          │
│  3. SandboxMiddleware       ─── 沙箱获取                             │
│  4. DanglingToolCallMiddleware ── 修复不完整的工具调用                │
│  5. GuardrailMiddleware ◄──── 评估每个工具调用                      │
│  6. ToolErrorHandlingMiddleware ── 将异常转换为消息                   │
│  7-12. (Summarization, Title, Memory, Vision, Subagent, Clarify)    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
           ┌──────────────────────────┐
           │    GuardrailProvider     │  ◄── 可插拔：任何具有
           │    （在 YAML 中配置）     │      evaluate/aevaluate
           └────────────┬─────────────┘
                        │
              ┌─────────┼──────────────┐
              │         │              │
              ▼         ▼              ▼
         内置        OAP Passport    自定义
         Allowlist   Provider        Provider
         （零依赖）   （开放标准）    （您的代码）
                        │
                  任何实现
                  （例如 APort，或
                   您自己的评估器）
```

`GuardrailMiddleware` 实现 `wrap_tool_call` / `awrap_tool_call`（与 `ToolErrorHandlingMiddleware` 使用的相同 `AgentMiddleware` 模式）。它：

1. 使用工具名称、参数和 passport 引用构建 `GuardrailRequest`
2. 调用所配置 provider 上的 `provider.evaluate(request)`
3. 如果**拒绝**：返回带原因的 `ToolMessage(status="error")` — agent 看到拒绝并调整
4. 如果**允许**：传递给实际工具处理器
5. 如果**provider 错误**且 `fail_closed=true`（默认）：阻止调用
6. `GraphBubbleUp` 异常（LangGraph 控制信号）始终传播，从不捕获

## 三种 Provider 选项

### 选项 1：内置 AllowlistProvider（零依赖）

最简单的选项。随 DeerFlow 一起提供。按名称阻止或允许工具。无需外部包、passport 或网络。

**config.yaml：**
```yaml
guardrails:
  enabled: true
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      denied_tools: ["bash", "write_file"]
```

这会阻止所有请求的 `bash` 和 `write_file`。所有其他工具通过。

您也可以使用 allowlist（只允许这些工具）：
```yaml
guardrails:
  enabled: true
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:
      allowed_tools: ["web_search", "read_file", "ls"]
```

**尝试它：**
1. 将上述配置添加到您的 `config.yaml`
2. 启动 DeerFlow：`make dev`
3. 问 agent："Use bash to run echo hello"
4. agent 看到：`Guardrail denied: tool 'bash' was blocked (oap.tool_not_allowed)`

### 选项 2：OAP Passport Provider（基于策略）

用于基于 [Open Agent Passport (OAP)](https://github.com/aporthq/aport-spec) 开放标准的策略执行。OAP passport 是一个 JSON 文档，声明 agent 的身份、能力和操作限制。任何读取 OAP passport 并返回 OAP 合规决策的 provider 都可以与 DeerFlow 配合使用。

```
┌─────────────────────────────────────────────────────────────┐
│                    OAP Passport (JSON)                        │
│                   （开放标准，任何 provider）                  │
│  {                                                           │
│    "spec_version": "oap/1.0",                                │
│    "status": "active",                                       │
│    "capabilities": [                                         │
│      {"id": "system.command.execute"},                       │
│      {"id": "data.file.read"},                               │
│      {"id": "data.file.write"},                              │
│      {"id": "web.fetch"},                                    │
│      {"id": "mcp.tool.execute"}                              │
│    ],                                                        │
│    "limits": {                                               │
│      "system.command.execute": {                             │
│        "allowed_commands": ["git", "npm", "node", "ls"],     │
│        "blocked_patterns": ["rm -rf", "sudo", "chmod 777"]   │
│      }                                                       │
│    }                                                         │
│  }                                                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
               任何 OAP 合规的 provider
          ┌────────────────┼────────────────┐
          │                │                │
     您自己的         APort (参考       其他未来
     评估器          实现)            实现
```

**手动创建 passport：**

OAP passport 只是一个 JSON 文件。您可以按照 [OAP 规范](https://github.com/aporthq/aport-spec/blob/main/oap/oap-spec.md) 手动创建一个，并针对 [JSON schema](https://github.com/aporthq/aport-spec/blob/main/oap/passport-schema.json) 进行验证。请参阅 [示例](https://github.com/aporthq/aport-spec/tree/main/oap/examples) 目录获取模板。

**使用 APort 作为参考实现：**

[APort Agent Guardrails](https://github.com/aporthq/aport-agent-guardrails) 是 OAP provider 的一个开源（Apache 2.0）实现。它处理 passport 创建、本地评估和可选的托管 API 评估。

```bash
pip install aport-agent-guardrails
aport setup --framework deerflow
```

这会创建：
- `~/.aport/deerflow/config.yaml` — 评估器配置（本地或 API 模式）
- `~/.aport/deerflow/aport/passport.json` — 具有能力和限制的 OAP passport

**config.yaml（使用 APort 作为 provider）：**
```yaml
guardrails:
  enabled: true
  provider:
    use: aport_guardrails.providers.generic:OAPGuardrailProvider
```

**config.yaml（使用您自己的 OAP provider）：**
```yaml
guardrails:
  enabled: true
  provider:
    use: my_oap_provider:MyOAPProvider
    config:
      passport_path: ./my-passport.json
```

任何接受 `framework` 作为 kwarg 并实现 `evaluate`/`aevaluate` 的 provider 都可以工作。OAP 标准定义 passport 格式和决策代码；DeerFlow 不关心哪个 provider 读取它们。

**passport 控制的内容：**

| Passport 字段 | 作用 | 示例 |
|---|---|---|
| `capabilities[].id` | agent 可以使用哪些工具类别 | `system.command.execute`, `data.file.write` |
| `limits.*.allowed_commands` | 允许哪些命令 | `["git", "npm", "node"]` 或 `["*"]` 表示全部 |
| `limits.*.blocked_patterns` | 始终拒绝的模式 | `["rm -rf", "sudo", "chmod 777"]` |
| `status` | 终止开关 | `active`, `suspended`, `revoked` |

**评估模式（取决于 provider）：**

OAP provider 可能支持不同的评估模式。例如，APort 参考实现支持：

| 模式 | 工作方式 | 网络 | 延迟 |
|---|---|---|---|
| **Local** | 本地评估 passport（bash 脚本）。 | 无 | ~300ms |
| **API** | 将 passport + 上下文发送到托管评估器。签名决策。 | 是 | ~65ms |

自定义 OAP provider 可以实现任何评估策略 — DeerFlow 中间件不关心 provider 如何做出决策。

**尝试它：**
1. 按上述方式安装和设置
2. 启动 DeerFlow 并问："Create a file called test.txt with content hello"
3. 然后问："Now delete it using bash rm -rf"
4. Guardrail 阻止它：`oap.blocked_pattern: Command contains blocked pattern: rm -rf`

### 选项 3：自定义 Provider（自带）

任何具有 `evaluate(request)` 和 `aevaluate(request)` 方法的 Python 类都可以工作。不需要基类或继承 — 这是一个结构化协议。

```python
# my_guardrail.py

class MyGuardrailProvider:
    name = "my-company"

    def evaluate(self, request):
        from deerflow.guardrails.provider import GuardrailDecision, GuardrailReason

        # 示例：阻止任何包含 "delete" 的 bash 命令
        if request.tool_name == "bash" and "delete" in str(request.tool_input):
            return GuardrailDecision(
                allow=False,
                reasons=[GuardrailReason(code="custom.blocked", message="delete not allowed")],
                policy_id="custom.v1",
            )
        return GuardrailDecision(allow=True, reasons=[GuardrailReason(code="oap.allowed")])

    async def aevaluate(self, request):
        return self.evaluate(request)
```

**config.yaml：**
```yaml
guardrails:
  enabled: true
  provider:
    use: my_guardrail:MyGuardrailProvider
```

确保 `my_guardrail.py` 在 Python 路径上（例如在 backend 目录中或安装为包）。

**尝试它：**
1. 在 backend 目录中创建 `my_guardrail.py`
2. 添加配置
3. 启动 DeerFlow 并问："Use bash to delete test.txt"
4. 您的 provider 阻止它

## 实现 Provider

### 必需接口

```
┌──────────────────────────────────────────────────┐
│              GuardrailProvider Protocol            │
│                                                   │
│  name: str                                        │
│                                                   │
│  evaluate(request: GuardrailRequest)              │
│      -> GuardrailDecision                         │
│                                                   │
│  aevaluate(request: GuardrailRequest)   (async)   │
│      -> GuardrailDecision                         │
└──────────────────────────────────────────────────┘

┌──────────────────────────┐    ┌──────────────────────────┐
│     GuardrailRequest      │    │    GuardrailDecision      │
│                           │    │                           │
│  tool_name: str           │    │  allow: bool              │
│  tool_input: dict         │    │  reasons: [GuardrailReason]│
│  agent_id: str | None     │    │  policy_id: str | None    │
│  thread_id: str | None    │    │  metadata: dict           │
│  is_subagent: bool        │    │                           │
│  timestamp: str           │    │  GuardrailReason:         │
│                           │    │    code: str              │
└──────────────────────────┘    │    message: str           │
                                └──────────────────────────┘
```

### DeerFlow 工具名称

这些是您的 provider 会在 `request.tool_name` 中看到的工具名称：

| 工具 | 作用 |
|---|---|
| `bash` | Shell 命令执行 |
| `write_file` | 创建/覆盖文件 |
| `str_replace` | 编辑文件（查找和替换） |
| `read_file` | 读取文件内容 |
| `ls` | 列出目录 |
| `web_search` | 网络搜索查询 |
| `web_fetch` | 获取 URL 内容 |
| `image_search` | 图片搜索 |
| `present_files` | 向用户展示文件 |
| `view_image` | 显示图片 |
| `ask_clarification` | 向用户提问 |
| `task` | 委托给子 agent |
| `mcp__*` | MCP 工具（动态） |

### OAP 原因代码

[OAP 规范](https://github.com/aporthq/aport-spec) 使用的标准代码：

| 代码 | 含义 |
|---|---|
| `oap.allowed` | 工具调用已授权 |
| `oap.tool_not_allowed` | 工具不在 allowlist 中 |
| `oap.command_not_allowed` | 命令不在 allowed_commands 中 |
| `oap.blocked_pattern` | 命令匹配被阻止的模式 |
| `oap.limit_exceeded` | 操作超出限制 |
| `oap.passport_suspended` | Passport 状态为暂停/撤销 |
| `oap.evaluator_error` | Provider 崩溃（fail-closed） |

### Provider 加载

DeerFlow 通过 `resolve_variable()` 加载 provider — 这与模型、工具和沙箱 provider 使用的机制相同。`use:` 字段是 Python 类路径：`package.module:ClassName`。

如果设置了 `config:`，则使用 `**config` kwargs 实例化 provider，并始终注入 `framework="deerflow"`。接受 `**kwargs` 以保持向前兼容：

```python
class YourProvider:
    def __init__(self, framework: str = "generic", **kwargs):
        # framework="deerflow" 告诉您使用哪个配置目录
        ...
```

## 配置参考

```yaml
guardrails:
  # 启用/禁用 guardrail 中间件（默认：false）
  enabled: true

  # 如果 provider 抛出异常则阻止工具调用（默认：true）
  fail_closed: true

  # Passport 引用 — 作为 request.agent_id 传递给 provider。
  # 文件路径、托管 agent ID 或 null（provider 从其配置中解析）。
  passport: null

  # Provider：通过类路径加载
  provider:
    use: deerflow.guardrails.builtin:AllowlistProvider
    config:  # 可选的 kwargs 传递给 provider.__init__
      denied_tools: ["bash"]
```

## 测试

```bash
cd backend
uv run python -m pytest tests/test_guardrail_middleware.py -v
```

25 个测试覆盖：
- AllowlistProvider：允许、拒绝、allowlist+denylist 两者、异步
- GuardrailMiddleware：允许通过、带 OAP 代码的拒绝、fail-closed、fail-open、passport 转发、空原因回退、空工具名称、protocol isinstance 检查
- 异步路径：awrap_tool_call 用于允许、拒绝、fail-closed、fail-open
- GraphBubbleUp：LangGraph 控制信号传播（不被捕获）
- 配置：默认值、from_dict、单例加载/重置

## 文件

```
packages/harness/deerflow/guardrails/
    __init__.py              # 公共导出
    provider.py              # GuardrailProvider 协议、GuardrailRequest、GuardrailDecision
    middleware.py           # GuardrailMiddleware（AgentMiddleware 子类）
    builtin.py              # AllowlistProvider（零依赖）

packages/harness/deerflow/config/
    guardrails_config.py    # GuardrailsConfig Pydantic 模型 + 单例

packages/harness/deerflow/agents/middlewares/
    tool_error_handling_middleware.py  # 在链中注册 GuardrailMiddleware

config.example.yaml          # 三种 provider 选项已记录
tests/test_guardrail_middleware.py  # 25 个测试
docs/GUARDRAILS.md           # 本文件
```
