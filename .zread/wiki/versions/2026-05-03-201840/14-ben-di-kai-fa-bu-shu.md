本文档面向希望直接在本地机器上启动和运行 DeerFlow 的中级开发者，介绍从环境准备到服务启动的完整流程。本地开发模式适合那些需要调试后端代码、深入理解系统架构或在没有 Docker 环境下工作的开发者。

## 环境要求

DeerFlow 本地开发依赖一系列工具链，确保这些组件正确安装是顺利启动的前提条件。

### 必要依赖

| 工具 | 版本要求 | 安装说明 |
|------|----------|----------|
| Node.js | 22+ | [nodejs.org](https://nodejs.org/) 下载 LTS 或使用 nvm |
| pnpm | 最新版 | `npm install -g pnpm` 或启用 Corepack `corepack enable` |
| uv | 最新版 | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| nginx | 任意稳定版 | macOS: `brew install nginx` / Ubuntu: `sudo apt install nginx` |
| Python | 3.12+ | 通常随 Node.js 工具链一起安装 |

### 资源规划建议

根据部署场景的不同，硬件资源配置也有所差异：

| 场景 | 起步配置 | 推荐配置 | 适用场景 |
|------|----------|----------|----------|
| 本地体验 | 4 vCPU、8 GB 内存 | 8 vCPU、16 GB 内存 | 单个开发者、轻量会话、模型走外部 API |
| Docker 开发 | 4 vCPU、8 GB 内存、25 GB SSD | 8 vCPU、16 GB 内存 | 包含镜像构建和沙箱容器开销 |
| 长期运行服务 | 8 vCPU、16 GB 内存、40 GB SSD | 16 vCPU、32 GB 内存 | 多 agent 任务、报告生成、重负载场景 |

> **注意**：2 核/4 GB 配置通常无法稳定运行 DeerFlow 正常工作负载。持续运行的服务推荐使用 Linux + Docker 环境；macOS 和 Windows 更适合作为开发机或体验环境。

Sources: [README_zh.md](README_zh.md#L77-L95), [Makefile](Makefile#L39-L48), [CONTRIBUTING.md](CONTRIBUTING.md#L155-L174)

## 快速开始

### 第一步：检查依赖工具

在开始配置之前，先验证本地环境是否满足所有依赖要求：

```bash
make check
```

该命令会检查 Node.js、pnpm、uv 和 nginx 的安装状态。如果任何工具缺失，命令会输出具体的安装指引。

Sources: [Makefile](Makefile#L50-L52), [scripts/check.py](scripts/check.py#L1-L171)

### 第二步：生成配置文件

DeerFlow 使用 YAML 配置文件管理所有运行时参数。首次设置时，运行配置生成命令：

```bash
make config
```

该命令会执行 `scripts/configure.py`，从示例模板生成以下文件：

- `config.yaml` — 主配置文件（模型、工具、沙箱等）
- `.env` — 环境变量文件（API keys）
- `frontend/.env` — 前端环境变量

> **警告**：如果 `config.yaml` 已存在，该命令会中止以避免覆盖现有配置。如需重新配置，请先删除现有文件。

Sources: [Makefile](Makefile#L53-L55), [scripts/configure.py](scripts/configure.py#L1-L59)

### 第三步：配置语言模型

编辑生成的 `config.yaml`，至少配置一个模型。以下是常见模型的配置示例：

```yaml
models:
  # OpenAI 模型
  - name: gpt-4
    display_name: GPT-4
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
    max_tokens: 4096
    temperature: 0.7

  # 通过 OpenRouter 使用 Gemini
  - name: openrouter-gemini-2.5-flash
    display_name: Gemini 2.5 Flash (OpenRouter)
    use: langchain_openai:ChatOpenAI
    model: google/gemini-2.5-flash-preview
    api_key: $OPENAI_API_KEY
    base_url: https://openrouter.ai/api/v1

  # 本地 Ollama 模型（保留思考内容）
  - name: qwen3-local
    display_name: Qwen3 32B (Ollama)
    use: langchain_ollama:ChatOllama
    model: qwen3:32b
    base_url: http://localhost:11434
    num_predict: 8192
    temperature: 0.7
    reasoning: true
    supports_thinking: true
```

**配置 API Key**：在 `.env` 文件中设置环境变量（推荐方式）：

```bash
TAVILY_API_KEY=your-tavily-api-key
OPENAI_API_KEY=your-openai-api-key
# 其他 provider 的 key 按需补充
```

Sources: [README_zh.md](README_zh.md#L45-L84), [config.example.yaml](config.example.yaml#L1-L100), [.env.example](.env.example#L1-L48)

### 第四步：安装依赖

配置完成后，安装项目依赖：

```bash
make install
```

该命令依次执行：

1. `cd backend && uv sync` — 同步 Python 依赖
2. `cd frontend && pnpm install` — 安装 Node.js 依赖
3. 安装 pre-commit hooks

Sources: [Makefile](Makefile#L65-L78)

### 第五步：启动开发服务

启动完整开发环境（支持热更新）：

```bash
make dev
```

该命令调用 `scripts/serve.sh --dev`，依次启动以下服务：

| 服务 | 端口 | 说明 |
|------|------|------|
| Gateway | 8001 | FastAPI REST API + Agent 运行时 |
| Frontend | 3000 | Next.js 开发服务器 |
| Nginx | 2026 | 反向代理，统一入口 |

Sources: [Makefile](Makefile#L90-L93), [scripts/serve.sh](scripts/serve.sh#L180-L230)

## 服务架构

DeerFlow 本地开发环境采用多服务协作架构，各组件通过 nginx 反向代理进行统一路由。

```mermaid
flowchart TB
    subgraph Client["客户端"]
        Browser["浏览器"]
        IM["IM 渠道"]
    end

    subgraph Gateway["Nginx 反向代理 :2026"]
        Router["请求路由"]
    end

    subgraph Services["本地服务"]
        subgraph Backend["后端服务"]
            Gateway_API["Gateway API<br/>:8001"]
            LangGraph["LangGraph Server<br/>:2024"]
        end
        
        Frontend["Frontend<br/>:3000"]
    end

    Browser -->|HTTP| Gateway
    IM -->|Webhook| Gateway
    Gateway -->|/api/*| Gateway_API
    Gateway -->|/api/langgraph/*| Gateway_API
    Gateway -->|/ [静态资源]| Frontend
    Gateway_API --> LangGraph
```

**请求路由规则**：

- `/api/langgraph/*` → Gateway API（重写为 `/api/*` 后转发）
- `/api/*` → Gateway API（REST 端点）
- `/` → Frontend（Next.js）
- `/health` → Gateway API（健康检查）

Sources: [docker/nginx/nginx.local.conf](docker/nginx/nginx.local.conf#L1-L246), [scripts/serve.sh](scripts/serve.sh#L180-L230)

## 交互式配置向导

对于新用户，推荐使用交互式配置向导简化设置过程：

```bash
make setup
```

该命令启动基于终端的向导程序，引导完成以下配置步骤：

1. **选择语言模型** — 选择 LLM provider 并输入 API key
2. **配置搜索工具** — 设置网络搜索和网页抓取服务
3. **配置执行环境** — 选择沙箱模式（Local/Docker）
4. **写入配置文件** — 自动生成 `config.yaml` 和 `.env`

向导会在完成后显示配置摘要，并提供启动命令。

Sources: [Makefile](Makefile#L45-L48), [scripts/setup_wizard.py](scripts/setup_wizard.py#L1-L166)

## 手动服务控制

如果需要单独控制各个服务，可以手动启动每个组件。

### 后端服务（终端 1）

```bash
cd backend
make dev
```

这将启动 Gateway API 服务（端口 8001），支持热更新。

### 前端服务（终端 2）

```bash
cd frontend
pnpm dev
```

这将启动 Next.js 开发服务器（端口 3000），支持热更新。

### Nginx 反向代理（终端 3）

```bash
nginx -g 'daemon off;' -c "$(pwd)/docker/nginx/nginx.local.conf" -p "$(pwd)"
```

启动 nginx 反向代理（端口 2026），将请求路由到 Gateway 和 Frontend。

Sources: [CONTRIBUTING.md](CONTRIBUTING.md#L175-L190), [backend/Makefile](backend/Makefile#L1-L19), [frontend/Makefile](frontend/Makefile#L1-L21)

## 配置管理

### 配置升级

当 `config.example.yaml` 添加新字段时，运行升级命令合并更改：

```bash
make config-upgrade
```

该命令会自动将新字段合并到现有 `config.yaml`，并创建 `.bak` 备份文件。

Sources: [Makefile](Makefile#L56-L58), [backend/docs/CONFIGURATION.md](backend/docs/CONFIGURATION.md#L1-L25)

### 配置验证

运行健康检查验证配置和系统依赖：

```bash
make doctor
```

该脚本会检查：

- Python/Node.js/pnpm/uv/nginx 版本
- `config.yaml` 语法正确性
- 模型 API key 配置
- 可选组件（搜索工具、沙箱等）

Sources: [Makefile](Makefile#L49-L51), [scripts/doctor.py](scripts/doctor.py#L1-L200)

### 配置文件优先级

后端按以下顺序查找配置文件：

1. `DEER_FLOW_CONFIG_PATH` 环境变量（如果设置）
2. `backend/config.yaml`（从 backend 目录运行）
3. 项目根目录 `config.yaml`（**推荐位置**）

Sources: [backend/docs/SETUP.md](backend/docs/SETUP.md#L1-L92)

## 开发模式与生产模式

DeerFlow 提供两种运行模式，适用于不同场景：

| 特性 | 开发模式 (`make dev`) | 生产模式 (`make start`) |
|------|----------------------|------------------------|
| 热更新 | Gateway 和 Frontend 支持 | 关闭 |
| 前端构建 | `pnpm dev`（实时编译） | `pnpm preview`（预构建） |
| 日志级别 | DEBUG | INFO |
| 适用场景 | 调试、特性开发 | 部署、压力测试 |

```bash
# 开发模式
make dev

# 生产模式
make start

# 后台运行模式
make dev-daemon
make start-daemon

# 停止所有服务
make stop

# 清理临时文件
make clean
```

Sources: [Makefile](Makefile#L90-L120), [scripts/serve.sh](scripts/serve.sh#L1-L100)

## Windows 特殊说明

在 Windows 上运行本地开发流程时，**必须使用 Git Bash** 执行服务脚本。基于 bash 的脚本不支持直接在原生 `cmd.exe` 或 PowerShell 中执行。

```bash
# 在 Git Bash 中运行
./scripts/serve.sh --dev

# 或使用 Makefile（Makefile 会自动检测 Windows 环境并调用 Git Bash）
make dev
```

Makefile 包含 Windows 检测逻辑，会通过 `scripts/run-with-git-bash.cmd` 包装脚本执行。

Sources: [Makefile](Makefile#L12-L23), [scripts/serve.sh](scripts/serve.sh#L1-L30)

## 常见问题排查

### 配置文件未找到

```bash
# 检查配置文件位置
ls -la config.yaml
ls -la .env

# 手动生成配置
make config
```

### 端口占用

如果启动时提示端口被占用：

```bash
# 查看占用端口的进程
lsof -i :8001  # Gateway
lsof -i :3000  # Frontend
lsof -i :2026  # Nginx

# 停止所有服务后重试
make stop
make dev
```

### 依赖安装失败

```bash
# 清理缓存后重试
cd backend && rm -rf .venv && uv sync
cd frontend && rm -rf node_modules && pnpm install
```

### nginx 启动失败

检查 nginx 配置文件语法：

```bash
nginx -t -c "$(pwd)/docker/nginx/nginx.local.conf"
```

Sources: [scripts/serve.sh](scripts/serve.sh#L60-L75)

## 下一步

完成本地开发环境配置后，建议继续阅读以下文档：

- [整体架构](4-zheng-ti-jia-gou) — 深入了解 DeerFlow 系统架构
- [Lead Agent 主代理](5-lead-agent-zhu-dai-li) — 理解核心代理组件
- [Docker 部署](15-docker-bu-shu) — 如果需要容器化部署
- [配置指南](3-pei-zhi-zhi-nan) — 完整的配置项说明