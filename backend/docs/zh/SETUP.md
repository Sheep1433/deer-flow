# 安装指南

DeerFlow 的快速安装说明。

## 配置设置

DeerFlow 使用 YAML 配置文件，应放置在**项目根目录**。

### 步骤

1. **导航到项目根目录**：
   ```bash
   cd /path/to/deer-flow
   ```

2. **复制示例配置**：
   ```bash
   cp config.example.yaml config.yaml
   ```

3. **编辑配置**：
   ```bash
   # 选项 A：设置环境变量（推荐）
   export OPENAI_API_KEY="your-key-here"

   # 选项 B：直接编辑 config.yaml
   vim config.yaml  # 或您喜欢的编辑器
   ```

4. **验证配置**：
   ```bash
   cd backend
   python -c "from deerflow.config import get_app_config; print('✓ Config loaded:', get_app_config().models[0].name)"
   ```

## 重要说明

- **位置**：`config.yaml` 应在 `deer-flow/`（项目根目录），而非 `deer-flow/backend/`
- **Git**：`config.yaml` 被自动忽略（包含 secrets）
- **优先级**：如果 `backend/config.yaml` 和 `../config.yaml` 都存在，backend 版本优先

## 配置文件位置

后端按此顺序查找 `config.yaml`：

1. `DEER_FLOW_CONFIG_PATH` 环境变量（如果已设置）
2. `backend/config.yaml`（从 backend/ 运行时是当前目录）
3. `deer-flow/config.yaml`（父目录 - **推荐位置**）

**推荐**：将 `config.yaml` 放在项目根目录（`deer-flow/config.yaml`）。

## 沙箱设置（可选但推荐）

如果您计划使用 Docker/基于容器的沙箱（在 `config.yaml` 中配置为 `sandbox.use: deerflow.community.aio_sandbox:AioSandboxProvider`），强烈建议预拉取容器镜像：

```bash
# 从项目根目录
make setup-sandbox
```

**为什么要预拉取？**
- 沙箱镜像（~500MB+）在首次使用时拉取，会导致长时间等待
- 预拉取提供清晰的进度指示
- 避免首次使用代理时的困惑

如果跳过此步骤，镜像将在首次代理执行时自动拉取，这可能需要几分钟（取决于网络速度）。

## 故障排除

### 找不到配置文件

```bash
# 检查后端查找的位置
cd deer-flow/backend
python -c "from deerflow.config.app_config import AppConfig; print(AppConfig.resolve_config_path())"
```

如果找不到配置：
1. 确保已将 `config.example.yaml` 复制到 `config.yaml`
2. 验证您在正确的目录中
3. 检查文件是否存在：`ls -la ../config.yaml`

### 权限被拒绝

```bash
chmod 600 ../config.yaml  # 保护敏感配置
```

## 另请参阅

- [配置指南](CONFIGURATION.md) - 详细配置选项
- [架构概述](../CLAUDE.md) - 系统架构