# API 参考文档

本文档提供 DeerFlow 后端 API 的完整参考。

## 概述

DeerFlow 后端暴露两组 API：

1. **LangGraph API** - Agent 交互、线程和流式处理（`/api/langgraph/*`）
2. **Gateway API** - 模型、MCP、技能、上传和产物（`/api/*`）

所有 API 均通过 Nginx 反向代理访问，端口为 2026。

## LangGraph API

基础 URL：`/api/langgraph`

LangGraph API 由 LangGraph 服务器提供，遵循 LangGraph SDK 约定。

### 线程

#### 创建线程

```http
POST /api/langgraph/threads
Content-Type: application/json
```

**请求体：**
```json
{
  "metadata": {}
}
```

**响应：**
```json
{
  "thread_id": "abc123",
  "created_at": "2024-01-15T10:30:00Z",
  "metadata": {}
}
```

#### 获取线程状态

```http
GET /api/langgraph/threads/{thread_id}/state
```

**响应：**
```json
{
  "values": {
    "messages": [...],
    "sandbox": {...},
    "artifacts": [...],
    "thread_data": {...},
    "title": "Conversation Title"
  },
  "next": [],
  "config": {...}
}
```

### 运行

#### 创建运行

执行 agent 并传入输入。

```http
POST /api/langgraph/threads/{thread_id}/runs
Content-Type: application/json
```

**请求体：**
```json
{
  "input": {
    "messages": [
      {
        "role": "user",
        "content": "Hello, can you help me?"
      }
    ]
  },
  "config": {
    "recursion_limit": 100,
    "configurable": {
      "model_name": "gpt-4",
      "thinking_enabled": false,
      "is_plan_mode": false
    }
  },
  "stream_mode": ["values", "messages-tuple", "custom"]
}
```

**流模式兼容性：**
- 可使用：`values`、`messages-tuple`、`custom`、`updates`、`events`、`debug`、`tasks`、`checkpoints`
- 不可使用：`tools`（已弃用/在当前 `langgraph-api` 中无效，会触发 schema 验证错误）

**递归限制：**

`config.recursion_limit` 限制 LangGraph 在单次运行中执行的图步骤数。`/api/langgraph/*` 端点直接连接到 LangGraph 服务器，因此继承 LangGraph 的原生默认值为 **25**，这对于计划模式或子 agent 密集型运行来说太低——在子 agent 结果返回第一轮后，agent 通常会抛出 `GraphRecursionError`，然后主 agent 才能综合最终答案。

DeerFlow 自身的 Gateway 和 IM 渠道路径通过在 `build_run_config`（参见 `backend/app/gateway/services.py`）中默认设置为 `100` 来缓解此问题，但直接调用 LangGraph API 的客户端必须在请求体中显式设置 `recursion_limit`。`100` 与 Gateway 默认值匹配，是一个安全的起始点；如果运行深度嵌套的子 agent 图，可以增加此值。

**可配置选项：**
- `model_name`（字符串）：覆盖默认模型
- `thinking_enabled`（布尔值）：为支持的模型启用扩展思考
- `is_plan_mode`（布尔值）：启用 TodoList middleware 进行任务跟踪

**响应：** 服务器发送事件（SSE）流

```
event: values
data: {"messages": [...], "title": "..."}

event: messages
data: {"content": "Hello! I'd be happy to help.", "role": "assistant"}

event: end
data: {}
```

#### 获取运行历史

```http
GET /api/langgraph/threads/{thread_id}/runs
```

**响应：**
```json
{
  "runs": [
    {
      "run_id": "run123",
      "status": "success",
      "created_at": "2024-01-15T10:30:00Z"
    }
  ]
}
```

#### 流式运行

实时流式响应。

```http
POST /api/langgraph/threads/{thread_id}/runs/stream
Content-Type: application/json
```

请求体与创建运行相同。返回 SSE 流。

---

## Gateway API

基础 URL：`/api`

### 模型

#### 获取模型列表

获取配置中的所有可用 LLM 模型。

```http
GET /api/models
```

**响应：**
```json
{
  "models": [
    {
      "name": "gpt-4",
      "display_name": "GPT-4",
      "supports_thinking": false,
      "supports_vision": true
    },
    {
      "name": "claude-3-opus",
      "display_name": "Claude 3 Opus",
      "supports_thinking": false,
      "supports_vision": true
    },
    {
      "name": "deepseek-v3",
      "display_name": "DeepSeek V3",
      "supports_thinking": true,
      "supports_vision": false
    }
  ]
}
```

#### 获取模型详情

```http
GET /api/models/{model_name}
```

**响应：**
```json
{
  "name": "gpt-4",
  "display_name": "GPT-4",
  "model": "gpt-4",
  "max_tokens": 4096,
  "supports_thinking": false,
  "supports_vision": true
}
```

### MCP 配置

#### 获取 MCP 配置

获取当前 MCP 服务器配置。

```http
GET /api/mcp/config
```

**响应：**
```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "***"
      },
      "description": "GitHub operations"
    },
    "filesystem": {
      "enabled": false,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"],
      "description": "File system access"
    }
  }
}
```

#### 更新 MCP 配置

更新 MCP 服务器配置。

```http
PUT /api/mcp/config
Content-Type: application/json
```

**请求体：**
```json
{
  "mcpServers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "$GITHUB_TOKEN"
      },
      "description": "GitHub operations"
    }
  }
}
```

**响应：**
```json
{
  "success": true,
  "message": "MCP configuration updated"
}
```

### 技能

#### 获取技能列表

获取所有可用技能。

```http
GET /api/skills
```

**响应：**
```json
{
  "skills": [
    {
      "name": "pdf-processing",
      "display_name": "PDF Processing",
      "description": "Handle PDF documents efficiently",
      "enabled": true,
      "license": "MIT",
      "path": "public/pdf-processing"
    },
    {
      "name": "frontend-design",
      "display_name": "Frontend Design",
      "description": "Design and build frontend interfaces",
      "enabled": false,
      "license": "MIT",
      "path": "public/frontend-design"
    }
  ]
}
```

#### 获取技能详情

```http
GET /api/skills/{skill_name}
```

**响应：**
```json
{
  "name": "pdf-processing",
  "display_name": "PDF Processing",
  "description": "Handle PDF documents efficiently",
  "enabled": true,
  "license": "MIT",
  "path": "public/pdf-processing",
  "allowed_tools": ["read_file", "write_file", "bash"],
  "content": "# PDF Processing\n\nInstructions for the agent..."
}
```

#### 启用技能

```http
POST /api/skills/{skill_name}/enable
```

**响应：**
```json
{
  "success": true,
  "message": "Skill 'pdf-processing' enabled"
}
```

#### 禁用技能

```http
POST /api/skills/{skill_name}/disable
```

**响应：**
```json
{
  "success": true,
  "message": "Skill 'pdf-processing' disabled"
}
```

#### 安装技能

从 `.skill` 文件安装技能。

```http
POST /api/skills/install
Content-Type: multipart/form-data
```

**请求体：**
- `file`：要安装的 `.skill` 文件

**响应：**
```json
{
  "success": true,
  "message": "Skill 'my-skill' installed successfully",
  "skill": {
    "name": "my-skill",
    "display_name": "My Skill",
    "path": "custom/my-skill"
  }
}
```

### 文件上传

#### 上传文件

上传一个或多个文件到线程。

```http
POST /api/threads/{thread_id}/uploads
Content-Type: multipart/form-data
```

**请求体：**
- `files`：要上传的一个或多个文件

**响应：**
```json
{
  "success": true,
  "files": [
    {
      "filename": "document.pdf",
      "size": 1234567,
      "path": ".deer-flow/threads/abc123/user-data/uploads/document.pdf",
      "virtual_path": "/mnt/user-data/uploads/document.pdf",
      "artifact_url": "/api/threads/abc123/artifacts/mnt/user-data/uploads/document.pdf",
      "markdown_file": "document.md",
      "markdown_path": ".deer-flow/threads/abc123/user-data/uploads/document.md",
      "markdown_virtual_path": "/mnt/user-data/uploads/document.md",
      "markdown_artifact_url": "/api/threads/abc123/artifacts/mnt/user-data/uploads/document.md"
    }
  ],
  "message": "Successfully uploaded 1 file(s)"
}
```

**支持的文档格式**（自动转换为 Markdown）：
- PDF（`.pdf`）
- PowerPoint（`.ppt`、`.pptx`）
- Excel（`.xls`、`.xlsx`）
- Word（`.doc`、`.docx`）

#### 获取已上传文件列表

```http
GET /api/threads/{thread_id}/uploads/list
```

**响应：**
```json
{
  "files": [
    {
      "filename": "document.pdf",
      "size": 1234567,
      "path": ".deer-flow/threads/abc123/user-data/uploads/document.pdf",
      "virtual_path": "/mnt/user-data/uploads/document.pdf",
      "artifact_url": "/api/threads/abc123/artifacts/mnt/user-data/uploads/document.pdf",
      "extension": ".pdf",
      "modified": 1705997600.0
    }
  ],
  "count": 1
}
```

#### 删除文件

```http
DELETE /api/threads/{thread_id}/uploads/{filename}
```

**响应：**
```json
{
  "success": true,
  "message": "Deleted document.pdf"
}
```

### 线程清理

在 LangGraph 线程本身删除后，删除 `.deer-flow/threads/{thread_id}` 下由 DeerFlow 管理的本地线程文件。

```http
DELETE /api/threads/{thread_id}
```

**响应：**
```json
{
  "success": true,
  "message": "Deleted local thread data for abc123"
}
```

**错误行为：**
- `422` 表示无效的线程 ID
- `500` 返回通用的 `{"detail": "Failed to delete local thread data."}` 响应，而完整的异常详情保留在服务器日志中

### 产物

#### 获取产物

下载或查看 agent 生成的产物。

```http
GET /api/threads/{thread_id}/artifacts/{path}
```

**路径示例：**
- `/api/threads/abc123/artifacts/mnt/user-data/outputs/result.txt`
- `/api/threads/abc123/artifacts/mnt/user-data/uploads/document.pdf`

**查询参数：**
- `download`（布尔值）：如果为 `true`，使用 Content-Disposition 头强制下载

**响应：** 具有适当 Content-Type 的文件内容

---

## 错误响应

所有 API 以一致的格式返回错误：

```json
{
  "detail": "Error message describing what went wrong"
}
```

**HTTP 状态码：**
- `400` - 错误请求：输入无效
- `404` - 未找到：资源不存在
- `422` - 验证错误：请求验证失败
- `500` - 内部服务器错误：服务器端错误

---

## 认证

目前，DeerFlow 未实现认证。所有 API 均可无需凭据即可访问。

注意：这是关于 DeerFlow API 认证的。MCP 出站连接仍可为配置的 HTTP/SSE MCP 服务器使用 OAuth。

对于生产部署，建议：
1. 使用 Nginx 进行基本认证或 OAuth 集成
2. 部署在 VPN 或专用网络后
3. 实现自定义认证中间件

---

## 速率限制

默认情况下未实现速率限制。对于生产部署，请在 Nginx 中配置速率限制：

```nginx
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;

location /api/ {
    limit_req zone=api burst=20 nodelay;
    proxy_pass http://backend;
}
```

---

## WebSocket 支持

LangGraph 服务器支持 WebSocket 连接以进行实时流式处理。连接到：

```
ws://localhost:2026/api/langgraph/threads/{thread_id}/runs/stream
```

---

## SDK 使用

### Python（LangGraph SDK）

```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:2026/api/langgraph")

# 创建线程
thread = await client.threads.create()

# 运行 agent
async for event in client.runs.stream(
    thread["thread_id"],
    "lead_agent",
    input={"messages": [{"role": "user", "content": "Hello"}]},
    config={"configurable": {"model_name": "gpt-4"}},
    stream_mode=["values", "messages-tuple", "custom"],
):
    print(event)
```

### JavaScript/TypeScript

```typescript
// 使用 fetch 调用 Gateway API
const response = await fetch('/api/models');
const data = await response.json();
console.log(data.models);

// 使用 EventSource 进行流式处理
const eventSource = new EventSource(
  `/api/langgraph/threads/${threadId}/runs/stream`
);
eventSource.onmessage = (event) => {
  console.log(JSON.parse(event.data));
};
```

### cURL 示例

```bash
# 获取模型列表
curl http://localhost:2026/api/models

# 获取 MCP 配置
curl http://localhost:2026/api/mcp/config

# 上传文件
curl -X POST http://localhost:2026/api/threads/abc123/uploads \
  -F "files=@document.pdf"

# 启用技能
curl -X POST http://localhost:2026/api/skills/pdf-processing/enable

# 创建线程并运行 agent
curl -X POST http://localhost:2026/api/langgraph/threads \
  -H "Content-Type: application/json" \
  -d '{}'

curl -X POST http://localhost:2026/api/langgraph/threads/abc123/runs \
  -H "Content-Type: application/json" \
  -d '{
    "input": {"messages": [{"role": "user", "content": "Hello"}]},
    "config": {
      "recursion_limit": 100,
      "configurable": {"model_name": "gpt-4"}
    }
  }'
```

> `/api/langgraph/*` 端点绕过 DeerFlow 的 Gateway，继承 LangGraph 原生的 `recursion_limit` 默认值 25，这对于计划模式或子 agent 运行来说太低。请显式设置 `config.recursion_limit`——参见[创建运行](#create-run)部分了解更多详情。
