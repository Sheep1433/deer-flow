# MCP（模型上下文协议）配置

DeerFlow 支持可配置的 MCP 服务器和技能扩展其功能，这些功能从项目根目录下的专用 `extensions_config.json` 文件加载。

## 设置

1. 将 `extensions_config.example.json` 复制到项目根目录的 `extensions_config.json`。
   ```bash
   # 复制示例配置
   cp extensions_config.example.json extensions_config.json
   ```

2. 通过设置 `"enabled": true` 来启用所需的 MCP 服务器或技能。
3. 根据需要配置每个服务器的命令、参数和环境变量。
4. 重启应用程序以加载和注册 MCP 工具。

## OAuth 支持（HTTP/SSE MCP 服务器）

对于 `http` 和 `sse` MCP 服务器，DeerFlow 支持 OAuth 令牌获取和自动令牌刷新。

- 支持的授权类型：`client_credentials`、`refresh_token`
- 在 `extensions_config.json` 中为每个服务器配置 `oauth` 块
- 密钥应通过环境变量提供（例如：`$MCP_OAUTH_CLIENT_SECRET`）

示例：

```json
{
   "mcpServers": {
      "secure-http-server": {
         "enabled": true,
         "type": "http",
         "url": "https://api.example.com/mcp",
         "oauth": {
            "enabled": true,
            "token_url": "https://auth.example.com/oauth/token",
            "grant_type": "client_credentials",
            "client_id": "$MCP_OAUTH_CLIENT_ID",
            "client_secret": "$MCP_OAUTH_CLIENT_SECRET",
            "scope": "mcp.read",
            "refresh_skew_seconds": 60
         }
      }
   }
}
```

## 自定义工具拦截器

您可以注册自定义拦截器，在每次 MCP 工具调用之前运行。这对于注入每个请求的头部（例如，来自 LangGraph 执行上下文的用户认证令牌）、日志记录或指标非常有用。

使用 `mcpInterceptors` 字段在 `extensions_config.json` 中声明拦截器：

```json
{
  "mcpInterceptors": [
    "my_package.mcp.auth:build_auth_interceptor"
  ],
  "mcpServers": { ... }
}
```

每个条目是一个 Python 导入路径，格式为 `module:variable`（通过 `resolve_variable` 解析）。该变量必须是一个**无参数构建器函数**，返回一个与 `MultiServerMCPClient` 的 `tool_interceptors` 接口兼容的异步拦截器，或 `None` 以跳过。

从 LangGraph 元数据注入认证头部的拦截器示例：

```python
def build_auth_interceptor():
    async def interceptor(request, handler):
        from langgraph.config import get_config
        metadata = get_config().get("metadata", {})
        headers = dict(request.headers or {})
        if token := metadata.get("auth_token"):
            headers["X-Auth-Token"] = token
        return await handler(request.override(headers=headers))
    return interceptor
```

- 接受单个字符串值，并将其规范化为一个元素的列表。
- 无效路径或构建器失败会记录为警告，不会阻塞其他拦截器。
- 构建器返回值必须是 `callable`；非可调用值会被跳过并记录警告。

## 工作原理

MCP 服务器公开的工具在运行时自动被发现并集成到 DeerFlow 的代理系统中。启用后，这些工具无需额外的代码更改即可供代理使用。

## 示例功能

MCP 服务器可以提供以下访问：

- **文件系统**
- **数据库**（例如 PostgreSQL）
- **外部 API**（例如 GitHub、Brave Search）
- **浏览器自动化**（例如 Puppeteer）
- **自定义 MCP 服务器实现**

## 了解更多

有关模型上下文协议的详细文档，请访问：  
https://modelcontextprotocol.io
