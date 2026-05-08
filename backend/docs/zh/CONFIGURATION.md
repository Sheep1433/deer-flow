# 配置指南

本指南说明如何为您的环境配置 DeerFlow。

## 配置版本控制

`config.example.yaml` 包含一个 `config_version` 字段，用于跟踪架构变更。当示例版本高于您本地的 `config.yaml` 时，应用程序会在启动时发出警告：

```
WARNING - Your config.yaml (version 0) is outdated — the latest version is 1.
Run `make config-upgrade` to merge new fields into your config.
```

- 您配置中**缺少 `config_version`** 视为版本 0。
- 运行 `make config-upgrade` 可自动合并缺失字段（保留您现有的值，会创建 `.bak` 备份）。
- 更改配置架构时，在 `config.example.yaml` 中更新 `config_version`。

## 配置章节

### 模型

配置可供 agent 使用的 LLM 模型：

```yaml
models:
  - name: gpt-4                    # 内部标识符
    display_name: GPT-4            # 人类可读名称
    use: langchain_openai:ChatOpenAI  # LangChain 类路径
    model: gpt-4                   # API 的模型标识符
    api_key: $OPENAI_API_KEY       # API 密钥（使用环境变量）
    max_tokens: 4096               # 每次请求的最大 token 数
    temperature: 0.7               # 采样温度
```

**支持的提供商**：
- OpenAI (`langchain_openai:ChatOpenAI`)
- Anthropic (`langchain_anthropic:ChatAnthropic`)
- DeepSeek (`langchain_deepseek:ChatDeepSeek`)
- Claude Code OAuth (`deerflow.models.claude_provider:ClaudeChatModel`)
- Codex CLI (`deerflow.models.openai_codex_provider:CodexChatModel`)
- 任何 LangChain 兼容的提供商

CLI 支持的提供商示例：

```yaml
models:
  - name: gpt-5.4
    display_name: GPT-5.4 (Codex CLI)
    use: deerflow.models.openai_codex_provider:CodexChatModel
    model: gpt-5.4
    supports_thinking: true
    supports_reasoning_effort: true

  - name: claude-sonnet-4.6
    display_name: Claude Sonnet 4.6 (Claude Code OAuth)
    use: deerflow.models.claude_provider:ClaudeChatModel
    model: claude-sonnet-4-6
    max_tokens: 4096
    supports_thinking: true
```

**CLI 支持的提供商的认证行为**：
- `CodexChatModel` 从 `~/.codex/auth.json` 加载 Codex CLI 认证
- Codex Responses 端点当前拒绝 `max_tokens` 和 `max_output_tokens`，因此 `CodexChatModel` 不暴露请求级别的 token 上限
- `ClaudeChatModel` 接受 `CLAUDE_CODE_OAUTH_TOKEN`、`ANTHROPIC_AUTH_TOKEN`、`CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR`、`CLAUDE_CODE_CREDENTIALS_PATH`，或明文 `~/.claude/.credentials.json`
- 在 macOS 上，DeerFlow 不会自动探测 Keychain。需要时使用 `scripts/export_claude_code_oauth.py` 显式导出 Claude Code 认证

要将 OpenAI 的 `/v1/responses` 端点与 LangChain 一起使用，继续使用 `langchain_openai:ChatOpenAI` 并设置：

```yaml
models:
  - name: gpt-5-responses
    display_name: GPT-5 (Responses API)
    use: langchain_openai:ChatOpenAI
    model: gpt-5
    api_key: $OPENAI_API_KEY
    use_responses_api: true
    output_version: responses/v1
```

对于 OpenAI 兼容的网关（例如 Novita 或 OpenRouter），继续使用 `langchain_openai:ChatOpenAI` 并设置 `base_url`：

```yaml
models:
  - name: novita-deepseek-v3.2
    display_name: Novita DeepSeek V3.2
    use: langchain_openai:ChatOpenAI
    model: deepseek/deepseek-v3.2
    api_key: $NOVITA_API_KEY
    base_url: https://api.novita.ai/openai
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

  - name: minimax-m2.5
    display_name: MiniMax M2.5
    use: langchain_openai:ChatOpenAI
    model: MiniMax-M2.5
    api_key: $MINIMAX_API_KEY
    base_url: https://api.minimax.io/v1
    max_tokens: 4096
    temperature: 1.0  # MiniMax 要求温度在 (0.0, 1.0] 范围内
    supports_vision: true

  - name: minimax-m2.5-highspeed
    display_name: MiniMax M2.5 Highspeed
    use: langchain_openai:ChatOpenAI
    model: MiniMax-M2.5-highspeed
    api_key: $MINIMAX_API_KEY
    base_url: https://api.minimax.io/v1
    max_tokens: 4096
    temperature: 1.0  # MiniMax 要求温度在 (0.0, 1.0] 范围内
    supports_vision: true
  - name: openrouter-gemini-2.5-flash
    display_name: Gemini 2.5 Flash (OpenRouter)
    use: langchain_openai:ChatOpenAI
    model: google/gemini-2.5-flash-preview
    api_key: $OPENAI_API_KEY
    base_url: https://openrouter.ai/api/v1
```

如果您的 OpenRouter 密钥在不同的环境变量名中，请将 `api_key` 显式指向该变量（例如 `api_key: $OPENROUTER_API_KEY`）。

**思考模型**：
某些模型支持复杂推理的"思考"模式：

```yaml
models:
  - name: deepseek-v3
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled
```

**通过 OpenAI 兼容网关使用 Gemini 并启用思考**：

当通过 OpenAI 兼容代理（Vertex AI OpenAI 兼容端点、AI Studio 或第三方网关）路由 Gemini 并启用思考时，API 会将 `thought_signature` 附加到响应中返回的每个 tool-call 对象上。每个后续重放这些助手消息的请求**必须**将这些签名回显到 tool-call 条目上，否则 API 返回：

```
HTTP 400 INVALID_ARGUMENT: function call `<tool>` in the N. content block is
missing a `thought_signature`.
```

标准的 `langchain_openai:ChatOpenAI` 在序列化消息时会静默删除 `thought_signature`。请改用 `deerflow.models.patched_openai:PatchedChatOpenAI` —— 它会将 tool-call 签名（从 `AIMessage.additional_kwargs["tool_calls"]` 中获取）重新注入到每个传出 payload 中：

```yaml
models:
  - name: gemini-2.5-pro-thinking
    display_name: Gemini 2.5 Pro (Thinking)
    use: deerflow.models.patched_openai:PatchedChatOpenAI
    model: google/gemini-2.5-pro-preview   # 您的网关期望的模型名称
    api_key: $GEMINI_API_KEY
    base_url: https://<your-openai-compat-gateway>/v1
    max_tokens: 16384
    supports_thinking: true
    supports_vision: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled
```

对于**不带思考**访问的 Gemini（例如通过 OpenRouter，思考未激活），普通的 `langchain_openai:ChatOpenAI` 配合 `supports_thinking: false` 就足够了，不需要补丁。

### 工具组

将工具组织成逻辑组：

```yaml
tool_groups:
  - name: web          # 网页浏览和搜索
  - name: file:read    # 只读文件操作
  - name: file:write   # 写文件操作
  - name: bash         # Shell 命令执行
```

### 工具

配置可供 agent 使用的特定工具：

```yaml
tools:
  - name: web_search
    group: web
    use: deerflow.community.tavily.tools:web_search_tool
    max_results: 5
    # api_key: $TAVILY_API_KEY  # 可选
```

**内置工具**：
- `web_search` - 网络搜索（DuckDuckGo、Tavily、Exa、InfoQuest、Firecrawl）
- `web_fetch` - 获取网页内容（Jina AI、Exa、InfoQuest、Firecrawl）
- `ls` - 列出目录内容
- `read_file` - 读取文件内容
- `write_file` - 写入文件内容
- `str_replace` - 文件中的字符串替换
- `bash` - 执行 bash 命令

### 沙箱

DeerFlow 支持多种沙箱执行模式。在 `config.yaml` 中配置您喜欢的模式：

**本地执行**（直接在主机上运行沙箱代码）：
```yaml
sandbox:
   use: deerflow.sandbox.local:LocalSandboxProvider # 本地执行
   allow_host_bash: false # 默认；除非明确重新启用，否则禁用主机 bash
```

**Docker 执行**（在隔离的 Docker 容器中运行沙箱代码）：
```yaml
sandbox:
   use: deerflow.community.aio_sandbox:AioSandboxProvider # 基于 Docker 的沙箱
```

**使用 Kubernetes 的 Docker 执行**（通过 provisioner 服务在 Kubernetes Pod 中运行沙箱代码）：

此模式在您**主机机器的集群**上的隔离 Kubernetes Pod 中运行每个沙箱。需要 Docker Desktop K8s、OrbStack 或类似的本地 K8s 设置。

```yaml
sandbox:
   use: deerflow.community.aio_sandbox:AioSandboxProvider
   provisioner_url: http://provisioner:8002
```

使用 Docker 开发（`make docker-start`）时，DeerFlow 仅在此 provisioner 模式配置时才启动 `provisioner` 服务。在本地或普通 Docker 沙箱模式下，会跳过 `provisioner`。

有关详细配置、先决条件和故障排查，请参阅 [Provisioner 设置指南](../../docker/provisioner/README.md)。

在本地执行或基于 Docker 的隔离之间选择：

**选项 1：本地沙箱**（默认，设置更简单）：
```yaml
sandbox:
  use: deerflow.sandbox.local:LocalSandboxProvider
  allow_host_bash: false
```

`allow_host_bash` 默认为 `false`。DeerFlow 的本地沙箱是一种主机端便利模式，不是安全的 shell 隔离边界。如果您需要 `bash`，请优先使用 `AioSandboxProvider`。仅在为完全可信的单用户本地工作流设置 `allow_host_bash: true`。

**选项 2：Docker 沙箱**（隔离，更安全）：
```yaml
sandbox:
  use: deerflow.community.aio_sandbox:AioSandboxProvider
  port: 8080
  auto_start: true
  container_prefix: deer-flow-sandbox

  # 可选：额外的挂载
  mounts:
    - host_path: /path/on/host
      container_path: /path/in/container
      read_only: false
```

当您配置 `sandbox.mounts` 时，DeerFlow 在 agent prompt 中暴露这些 `container_path` 值，以便 agent 可以直接发现和操作挂载的目录，而不是假设所有内容必须位于 `/mnt/user-data` 下。

对于使用 localhost 的裸机 Docker 沙箱运行，DeerFlow 默认将沙箱 HTTP 端口绑定到 `127.0.0.1`，因此不会在每个主机接口上暴露。对于通过 `host.docker.internal` 连接的使用 Docker-outside-of-Docker 部署，为兼容性保留宽泛的传统绑定。如果您的部署需要不同的绑定地址，请显式设置 `DEER_FLOW_SANDBOX_BIND_HOST`。

### 技能

为专业工作流配置技能目录：

```yaml
skills:
  # 主机路径（可选，默认：../skills）
  path: /custom/path/to/skills

  # 容器挂载路径（默认：/mnt/skills）
  container_path: /mnt/skills
```

**技能如何工作**：
- 技能存储在 `deer-flow/skills/{public,custom}/`
- 每个技能有一个包含元数据的 `SKILL.md` 文件
- 技能自动被发现和加载
- 在本地和 Docker 沙箱中均可通过路径映射访问

**每个 Agent 的技能过滤**：
自定义 agent 可以通过在 `config.yaml`（位于 `workspace/agents/<agent_name>/config.yaml`）中定义 `skills` 字段来限制加载哪些技能：
- **省略或 `null`**：加载所有全局启用的技能（默认后备）。
- **`[]`（空列表）**：为此特定 agent 禁用所有技能。
- **`["skill-name"]`**：仅加载明确指定的技能。

### 标题生成

自动对话标题生成：

```yaml
title:
  enabled: true
  max_words: 6
  max_chars: 60
  model_name: null  # 使用列表中的第一个模型
```

### GitHub API 令牌（GitHub 深度研究技能的可选配置）

默认的 GitHub API 速率限制相当严格。对于频繁的项目研究，我们建议配置具有只读权限的个人访问令牌（PAT）。

**配置步骤**：
1. 在 `.env` 文件中取消注释 `GITHUB_TOKEN` 行并添加您的个人访问令牌
2. 重启 DeerFlow 服务以应用更改

## 环境变量

DeerFlow 支持使用 `$` 前缀进行环境变量替换：

```yaml
models:
  - api_key: $OPENAI_API_KEY  # 从环境读取
```

**常见环境变量**：
- `OPENAI_API_KEY` - OpenAI API 密钥
- `ANTHROPIC_API_KEY` - Anthropic API 密钥
- `DEEPSEEK_API_KEY` - DeepSeek API 密钥
- `NOVITA_API_KEY` - Novita API 密钥（OpenAI 兼容端点）
- `TAVILY_API_KEY` - Tavily 搜索 API 密钥
- `DEER_FLOW_CONFIG_PATH` - 自定义配置文件路径
- `GATEWAY_ENABLE_DOCS` - 设置为 `false` 以禁用 Swagger UI（`/docs`）、ReDoc（`/redoc`）和 OpenAPI 架构（`/openapi.json`）端点（默认：`true`）

## 配置位置

配置文件应放置在**项目根目录**（`deer-flow/config.yaml`），而不是 backend 目录中。

## 配置优先级

DeerFlow 按以下顺序搜索配置：

1. 代码中通过 `config_path` 参数指定的路径
2. `DEER_FLOW_CONFIG_PATH` 环境变量中的路径
3. 当前工作目录中的 `config.yaml`（通常运行时为 `backend/`）
4. 父目录中的 `config.yaml`（项目根目录：`deer-flow/`）

## 最佳实践

1. **将 `config.yaml` 放在项目根目录** - 不要放在 `backend/` 目录中
2. **永远不要提交 `config.yaml`** - 它已在 `.gitignore` 中
3. **使用环境变量存储密钥** - 不要硬编码 API 密钥
4. **保持 `config.example.yaml` 更新** - 记录所有新选项
5. **在本地测试配置更改** - 部署前先测试
6. **生产环境使用 Docker 沙箱** - 更好的隔离和安全性

## 故障排查

### "找不到配置文件"
- 确保 `config.yaml` 存在于**项目根目录**（`deer-flow/config.yaml`）
- 后端默认搜索父目录，因此根目录位置是首选
- 或者设置 `DEER_FLOW_CONFIG_PATH` 环境变量到自定义位置

### "API 密钥无效"
- 验证环境变量设置正确
- 检查 `$` 前缀是否用于环境变量引用

### "技能未加载"
- 检查 `deer-flow/skills/` 目录是否存在
- 验证技能是否有有效的 `SKILL.md` 文件
- 如果使用自定义路径，请检查 `skills.path` 配置

### "Docker 沙箱启动失败"
- 确保 Docker 正在运行
- 检查端口 8080（或配置的端口）是否可用
- 验证 Docker 镜像可访问

## 示例

请参阅 `config.example.yaml` 获取所有配置选项的完整示例。
