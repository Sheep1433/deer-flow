# DeerFlow

Deep Exploration and Efficient Research Flow — an open-source **super agent harness** that orchestrates sub-agents, memory, and sandboxes to accomplish complex tasks. Built by ByteDance, powered by LangGraph and LangChain.

## Project Overview

DeerFlow is an AI Agent框架 that allows you to:
- Create a lead agent that coordinates sub-agents
- Execute code/tasks in isolated sandboxes
- Maintain persistent memory across sessions
- Integrate with IM platforms (Telegram, Slack, Feishu, etc.)
- Extend via skills and MCP servers

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Python 3.12+, FastAPI, LangGraph, LangChain |
| Frontend | Next.js 16, React 19, Tailwind CSS 4, Shadcn UI |
| Runtime | LangGraph Server (port 2024) |
| API Gateway | FastAPI (port 8001), Nginx (port 2026) |
| Frontend Dev | Next.js (port 3000) |

## Directory Structure

```
deer-flow/
├── backend/
│   ├── app/
│   │   ├── gateway/          # FastAPI REST API (port 8001)
│   │   └── channels/         # IM integrations
│   ├── packages/harness/     # Core agent framework
│   │   └── deerflow/
│   │       ├── agents/       # Lead agent + middleware chain
│   │       ├── sandbox/      # Sandboxed execution (Local/Docker/K8s)
│   │       ├── subagents/    # Subagent delegation
│   │       ├── tools/        # Built-in tools
│   │       ├── memory/       # Persistent memory system
│   │       ├── skills/       # Skills loader
│   │       └── mcp/          # MCP integration
│   └── tests/
├── frontend/                 # Next.js 16 App Router
│   └── src/
│       ├── app/            # Pages
│       ├── components/     # React components
│       └── core/           # Business logic (threads, API, artifacts)
├── skills/                  # Agent skills (Markdown + YAML frontmatter)
│   └── public/             # Built-in skills
├── config.yaml             # Main configuration
├── extensions_config.json  # MCP servers & skills config
└── docs/
```

## Key Concepts

### Lead Agent
主代理，由 `make_lead_agent()` 创建，注册在 `langgraph.json` 中作为入口点。通过 18 个中间件组件处理请求：ThreadData、Uploads、Sandbox、Guardrail、Memory 等。

### Subagents
子代理，用于委托复杂任务给后台 agent 执行。最多 3 个并发子代理，15 分钟超时。通过 SSE 事件流返回结果。

### Memory System
持久化记忆系统，跨会话存储用户上下文、事实和历史。数据存储在 `backend/.deer-flow/users/{user_id}/memory.json`。

### Skills
技能系统，基于 Markdown + YAML frontmatter 的结构化能力模块。位于 `skills/{public,custom}/`，按需加载。

### Sandbox
沙箱系统，为每个线程提供隔离的执行环境。支持 Local、Docker、Kubernetes 三种模式。虚拟路径：`/mnt/user-data/{workspace,uploads,outputs}`。

### Tools
内置工具包括：
- **沙箱工具**: bash, ls, read_file, write_file, str_replace
- **内置工具**: present_files, ask_clarification, view_image, task
- **社区工具**: Tavily (搜索), Jina AI (抓取), Firecrawl, DuckDuckGo

## Development Commands

```bash
# 安装依赖
make setup

# 启动本地开发环境（Gateway + Frontend + Nginx）
make dev

# 仅启动后端 Gateway
make gateway

# Docker 开发环境
make docker-start

# 运行测试
cd backend && make test
```

## Configuration

主要配置文件：
- `config.yaml` — 模型、工具、沙箱、技能、内存、渠道等
- `extensions_config.json` — MCP 服务器和技能状态
- `.env` — API keys 和 trace 配置

配置值以 `$` 开头会解析为环境变量。

## Key Files

| 文件 | 用途 |
|------|------|
| `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | Lead Agent 工厂函数 |
| `backend/app/gateway/app.py` | FastAPI 应用入口 |
| `backend/langgraph.json` | LangGraph Agent 注册配置 |
| `frontend/src/core/` | 前端业务逻辑（threads, API, artifacts） |

## Workflow

1. 用户通过前端 (port 3000) 或 IM 渠道发起请求
2. 请求经过 Nginx (port 2026) 路由到 Gateway API (8001)
3. Lead Agent 协调 subagents、sandboxes、tools 和 memory 执行任务
4. 结果通过 SSE 流式返回前端或 IM 渠道

## 注意事项

- 后端使用 `uv` 作为包管理器
- 前端使用 `pnpm` 作为包管理器
- Node.js 版本需 22+
- Python 版本需 3.12+