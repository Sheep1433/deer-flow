# Apple Container 支持

DeerFlow 现在支持 Apple Container 作为 macOS 上的首选容器运行时，并自动回退到 Docker。

## 概述

从此版本开始，DeerFlow 在可用时自动检测并在 macOS 上使用 Apple Container，在以下情况下回退到 Docker：
- Apple Container 未安装
- 在非 macOS 平台上运行

这在 Apple Silicon Mac 上提供更好的性能，同时保持所有平台的兼容性。

## 好处

### 在配备 Apple Container 的 Apple Silicon Mac 上：
- **更好的性能**：无需 Rosetta 2 转换的本机 ARM64 执行
- **更低的资源使用**：比 Docker Desktop 更轻量
- **原生集成**：使用 macOS Virtualization.framework

### 回退到 Docker：
- 完全向后兼容
- 适用于所有平台（macOS、Linux、Windows）
- 无需配置更改

## 要求

### 对于 Apple Container（仅限 macOS）：
- macOS 15.0 或更高版本
- Apple Silicon（M1/M2/M3/M4）
- 已安装 Apple Container CLI

### 安装：
```bash
# 从 GitHub releases 下载
# https://github.com/apple/container/releases

# 验证安装
container --version

# 启动服务
container system start
```

### 对于 Docker（所有平台）：
- Docker Desktop 或 Docker Engine

## 工作原理

### 自动检测

`AioSandboxProvider` 自动检测可用的容器运行时：

1. 在 macOS 上：尝试 `container --version`
   - 成功 → 使用 Apple Container
   - 失败 → 回退到 Docker

2. 在其他平台上：直接使用 Docker

### 运行时差异

两个运行时使用几乎相同的命令语法：

**容器启动：**
```bash
# Apple Container
container run --rm -d -p 8080:8080 -v /host:/container -e KEY=value image

# Docker
docker run --rm -d -p 8080:8080 -v /host:/container -e KEY=value image
```

**容器清理：**
```bash
# Apple Container（带 --rm 标志）
container stop <id>  # 由于 --rm 自动删除

# Docker（带 --rm 标志）
docker stop <id>     # 由于 --rm 自动删除
```

### 实现细节

实现位于 `backend/packages/harness/deerflow/community/aio_sandbox/aio_sandbox_provider.py`：

- `_detect_container_runtime()`：在启动时检测可用运行时
- `_start_container()`：使用检测到的运行时，为 Apple Container 跳过 Docker 特定选项
- `_stop_container()`：使用适当的停止命令

## 配置

无需配置更改！系统自动工作。

但是，您可以通过检查日志来验证正在使用的运行时：

```
INFO:deerflow.community.aio_sandbox.aio_sandbox_provider:Detected Apple Container: container version 0.1.0
INFO:deerflow.community.aio_sandbox.aio_sandbox_provider:Starting sandbox container using container: ...
```

或对于 Docker：
```
INFO:deerflow.community.aio_sandbox.aio_sandbox_provider:Apple Container not available, falling back to Docker
INFO:deerflow.community.aio_sandbox.aio_sandbox_provider:Starting sandbox container using docker: ...
```

## 容器镜像

两个运行时都使用 OCI 兼容镜像。默认镜像适用于两者：

```yaml
sandbox:
  use: deerflow.community.aio_sandbox:AioSandboxProvider
  image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest  # 默认镜像
```

确保您的镜像适用于适当的架构：
- Apple Container on Apple Silicon 的 ARM64
- Intel Mac 上 Docker 的 AMD64
- 两种架构的多架构镜像

### 预拉取镜像（推荐）

**重要**：容器镜像通常很大（500MB+），在首次使用时拉取，这可能导致长时间等待且没有清晰的反馈。

**最佳实践**：在设置期间预拉取镜像：

```bash
# 从项目根目录
make setup-sandbox
```

此命令将：
1. 从 `config.yaml` 读取配置的镜像（或使用默认）
2. 检测可用运行时（Apple Container 或 Docker）
3. 带进度指示拉取镜像
4. 验证镜像可以投入使用

**手动预拉取**：

```bash
# 使用 Apple Container
container image pull enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest

# 使用 Docker
docker pull enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
```

如果跳过预拉取，镜像将在首次代理执行时自动拉取，这可能需要几分钟取决于您的网络速度。

## 清理脚本

项目包含处理两种运行时的统一清理脚本：

**脚本**：`scripts/cleanup-containers.sh`

**用法：**
```bash
# 清理所有 DeerFlow 沙箱容器
./scripts/cleanup-containers.sh deer-flow-sandbox

# 自定义前缀
./scripts/cleanup-containers.sh my-prefix
```

**Makefile 集成**：

`Makefile` 中的所有清理命令自动处理两种运行时：
```bash
make stop   # 停止所有服务并清理容器
make clean  # 完整清理包括日志
```

## 测试

测试容器运行时检测：

```bash
cd backend
python test_container_runtime.py
```

这将：
1. 检测可用运行时
2. 可选启动测试容器
3. 验证连接
4. 清理

## 故障排查

### 在 macOS 上未检测到 Apple Container

1. 检查是否已安装：
   ```bash
   which container
   container --version
   ```

2. 检查服务是否正在运行：
   ```bash
   container system start
   ```

3. 检查日志中的检测：
   ```bash
   # 在应用日志中查找检测消息
   grep "container runtime" logs/*.log
   ```

### 容器未清理

1. 手动检查正在运行的容器：
   ```bash
   # Apple Container
   container list

   # Docker
   docker ps
   ```

2. 手动运行清理脚本：
   ```bash
   ./scripts/cleanup-containers.sh deer-flow-sandbox
   ```

### 性能问题

- Apple Container 在 Apple Silicon 上应该更快
- 如果遇到问题，您可以强制使用 Docker，方法是临时重命名 `container` 命令：
   ```bash
   # 临时解决方法 - 不建议永久使用
   sudo mv /opt/homebrew/bin/container /opt/homebrew/bin/container.bak
   ```

## 参考

- [Apple Container GitHub](https://github.com/apple/container)
- [Apple Container 文档](https://github.com/apple/container/blob/main/docs/)
- [OCI 镜像规范](https://github.com/opencontainers/image-spec)
