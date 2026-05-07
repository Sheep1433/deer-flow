# DeerFlow Docker 部署笔记

## 环境信息
- macOS (Darwin)
- Docker Desktop
- MiniMax API

## 部署步骤

### 1. 生成配置文件
```bash
make config
```

### 2. 配置模型 (config.yaml)
修改 `config.yaml`，添加 MiniMax 模型配置：
```yaml
models:
  # MiniMax model (China endpoint)
  - name: minimax-m2.5
    display_name: MiniMax M2.5
    use: langchain_openai:ChatOpenAI
    model: MiniMax-M2.5
    api_key: $MINIMAX_API_KEY
    base_url: https://api.minimaxi.com/v1
    request_timeout: 600.0
    max_retries: 2
    max_tokens: 4096
    temperature: 1.0
    supports_vision: true
```

### 3. 配置 API Key (.env)
```bash
MINIMAX_API_KEY=sk-cp-noxX_bhB-wUwNA-FDXe7fRZoZGXpboiiRHcmZ5EMy369uJMvX2E4La35Mp5SF4B1YabYcFyF-jBOPTULYSn6lIZ9-q9230nyBH0twhJ100Te__Aw5orCWrc
```

### 4. 拉取基础镜像 (解决网络问题)
如果 Docker 拉取官方镜像失败，使用 mirror 仓库：
```bash
# 拉取 nginx
docker pull mirror.gcr.io/library/nginx:alpine
docker tag mirror.gcr.io/library/nginx:alpine nginx:alpine

# 拉取 node
docker pull mirror.gcr.io/library/node:22-alpine
docker tag mirror.gcr.io/library/node:22-alpine node:22-alpine

# 拉取 python
docker pull mirror.gcr.io/library/python:3.12-slim-bookworm
docker tag mirror.gcr.io/library/python:3.12-slim-bookworm python:3.12-slim-bookworm

# 拉取 docker:cli
docker pull mirror.gcr.io/library/docker:cli
docker tag mirror.gcr.io/library/docker:cli docker:cli
```

### 5. 启动 Docker 服务
```bash
make docker-start
```

### 6. 验证服务
```bash
# 检查容器状态
docker ps

# 检查日志
docker logs deer-flow-gateway -f

# 访问前端 (不需要认证)
curl http://localhost:2026/

# API 需要认证 (401 是正常的)
curl http://localhost:2026/api/models
```

## 常见问题

### Docker 构建失败 (apt-get / npm / corepack 超时)
原因: Docker 配置了代理但代理无法访问目标地址
症状: `apt-get update` 502、`corepack install` 超时、npm 无法下载

解决:
1. 取消代理环境变量后重试:
```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy
make docker-start
```

2. 或在 Docker Desktop 中关闭代理: Settings → Resources → Proxy → 关闭 "Enable manual proxy configuration"

3. 使用国内镜像:
```bash
export NPM_REGISTRY=https://registry.npmmirror.com
export APT_MIRROR=mirrors.aliyun.com
make docker-start
```

### API 返回 401 Unauthorized
原因: Gateway API 需要身份验证
解决: 这是预期行为，前端通过浏览器访问时会自动处理认证

### MiniMax API Key 无效 (invalid api key 2049)
原因: 使用了错误的 API endpoint
- MiniMax 国际版: `api.minimax.io` (海外)
- MiniMax 中国版: `api.minimaxi.com` (中国)

解决: 确认你的 API key 所属区域，使用对应的 base_url。测试命令:
```bash
# 测试中国区
curl -X POST https://api.minimaxi.com/v1/chat/completions \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{"model": "MiniMax-M2.7", "messages": [{"role": "user", "content": "hi"}]}'
```

修改 config.yaml 中的 base_url 后需要重启服务:
```bash
make docker-stop && make docker-start
```

## 清理镜像 (可选)
```bash
docker rmi mirror.gcr.io/library/python:3.12-slim-bookworm \
  mirror.gcr.io/library/docker:cli \
  mirror.gcr.io/library/nginx:alpine \
  mirror.gcr.io/library/node:22-alpine \
  mirror.gcr.io/library/alpine:latest
```

## 停止服务
```bash
make docker-stop
```