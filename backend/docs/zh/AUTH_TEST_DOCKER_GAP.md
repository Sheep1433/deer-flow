# Docker 测试差距（第七节 7.4）

本文档记录完整发布验证通过后 `backend/docs/AUTH_TEST_PLAN.md` 中唯一**未执行**的测试用例。

## 为什么存在此差距

发布验证环境（sg_dev: `10.251.229.92`）**未安装 Docker 守护进程**。TC-DOCKER 用例是容器运行时行为测试，需要实际的 Docker 引擎来启动 `docker/docker-compose.yaml` 服务。

```bash
$ ssh sg_dev "which docker; docker --version"
# (empty)
# bash: docker: command not found
```

所有其他测试计划部分均在以下环境执行：
- 本地开发机（Mac，所有服务本地运行），或
- 已部署的 sg_dev 实例（通过 SSH 隧道访问 gateway + frontend + nginx）

## 未执行的用例

| 用例 | 标题 | 覆盖内容 | 未运行原因 |
|---|---|---|---|
| TC-DOCKER-01 | `users.db` 卷持久化 | 验证 `DEER_FLOW_HOME` 绑定挂载在容器重启后存活 | 需要 `docker compose up` |
| TC-DOCKER-02 | 容器重启后会话持久化 | `AUTH_JWT_SECRET` 环境变量使 cookie 在 `docker compose down && up` 后保持有效 | 需要 `docker compose down/up` |
| TC-DOCKER-03 | 每 worker 速率限制器发散 | 确认进程内 `_login_attempts` 字典在 `gunicorn` workers（compose 文件中默认为 4 个）之间不共享状态；已知限制，已记录 | 需要多 worker 容器 |
| TC-DOCKER-04 | IM 渠道跳过 AuthMiddleware | 验证 Feishu/Slack/Telegram 分发器在容器内针对 `http://langgraph:2024` 运行，不经过 nginx | 需要 `docker logs` |
| TC-DOCKER-05 | 管理凭据暴露 | **简化后已更新**——之前是"日志抓取"，现在是"0600 凭据文件在 `DEER_FLOW_HOME`"。基于文件的行為已通过 TC-1.1 + TC-UPG-13 在 sg_dev（非 Docker）上验证，因此唯一的 Docker 特定差距是验证卷挂载是否将此文件投射到主机 | 需要容器 + 主机卷 |
| TC-DOCKER-06 | Gateway 模式 Docker 部署 | `./scripts/deploy.sh --gateway` 产生 3 容器拓扑（无 `langgraph` 容器）；与标准模式相同的认证流程 | 需要 `docker compose --profile gateway` |

## 非 Docker 测试已提供的覆盖

每个 Docker 用例中的**认证相关**行为已被在 sg_dev 或本地运行的测试用例覆盖：

| Docker 用例 | 认证行为由以下覆盖 |
|---|---|
| TC-DOCKER-01（卷持久化） | sg_dev 上的 TC-REENT-01（admin 行在 gateway 重启后存活）—— 相同的 SQLite 文件，只是中间没有容器层 |
| TC-DOCKER-02（会话持久化） | TC-API-02/03/06（cookie 往返），加上 TC-REENT-04（多 cookie）—— JWT 验证是进程状态无关的，容器重启等同于 `pkill uvicorn && uv run uvicorn` |
| TC-DOCKER-03（每 worker 速率限制） | TC-GW-04 + TC-REENT-09（单 worker 速率限制 + 5 分钟过期）。跨 worker 发散是内存字典架构的属性；没有认证代码路径不同 |
| TC-DOCKER-04（IM 渠道跳过认证） | 仅代码层面：`app/channels/manager.py` 直接使用 `langgraph_sdk`，没有 cookie 处理。通过 SDK 绕过而非 HTTP 绕过了 langgraph_auth 处理器 |
| TC-DOCKER-05（凭据暴露） | sg_dev 上的 TC-1.1（文件位于 `~/deer-flow/backend/.deer-flow/admin_initial_credentials.txt`，模式 0600，密码 22 字符）—— 唯一的 Docker 独特步骤是绑定挂载是否将此路径投影到主机，这是 `docker compose` 配置检查，不是运行时行为变更 |
| TC-DOCKER-06（gateway 模式容器） | 第七节 7.2 由 TC-GW-01..05 + 第二节（sg_dev 上的 gateway 模式认证流程）覆盖 —— 相同的 Gateway 代码，容器只是打包变更 |

## Docker 可用时的复现步骤

任何安装有 `docker` + `docker compose` 的人都可以通过按原样运行测试计划部分来复现差距。预检：

```bash
# 主机上必需
docker --version           # >=24.x
docker compose version     # plugin >=2.x

# 必需的环境变量（否则每次容器重启会话都会重置）
echo "AUTH_JWT_SECRET=$(python3 -c 'import secrets; print(secrets.token_urlsafe(32))')" \
  >> .env

# 可选：将 DEER_FLOW_HOME 固定到稳定的主机路径
echo "DEER_FLOW_HOME=$HOME/deer-flow-data" >> .env
```

然后按原样从测试计划运行 TC-DOCKER-01..06。

## 决策日志

- **不阻止发布。** 每个 Docker 用例中的认证相关行为在裸机上都有已验证的等价物。差距纯粹在于*容器打包*细节（绑定挂载、多 worker、日志收集），而非认证代码路径是否工作。
- **TC-DOCKER-05 已在原位更新**至反映简化后的现实（凭据文件 → 0600 文件，无日志泄漏）。旧的"在 docker logs 中 grep 'Password:'"预期会静默失败并给出虚假的覆盖感。