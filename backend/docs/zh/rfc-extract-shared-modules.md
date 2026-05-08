# RFC：将共享技能安装器和上传管理器提取到 Harness

## 1. 问题

Gateway（`app/gateway/routers/skills.py`、`uploads.py`）和 Client（`deerflow/client.py`）各自独立实现了相同的业务逻辑：

### 技能安装

| 逻辑 | Gateway（`skills.py`） | Client（`client.py`） |
|-------|----------------------|---------------------|
| Zip 安全检查 | `_is_unsafe_zip_member()` | 内联 `Path(info.filename).is_absolute()` |
| 符号链接过滤 | `_is_symlink_member()` | `p.is_symlink()` 提取后删除 |
| Zip 炸弹防御 | `total_size += info.file_size`（声明的） | `total_size > 100MB`（声明的） |
| macOS 元数据过滤 | `_should_ignore_archive_entry()` | 无 |
| Frontmatter 验证 | `_validate_skill_frontmatter()` | `_validate_skill_frontmatter()` |
| 重复检测 | `HTTPException(409)` | `ValueError` |

**两个实现，行为不一致**：Gateway 流式写入并跟踪真实解压大小；Client 求和声明的 `file_size`。Gateway 在提取时跳过符号链接；Client 提取所有内容后遍历并删除符号链接。

### 上传管理

| 逻辑 | Gateway（`uploads.py`） | Client（`client.py`） |
|-------|----------------------|---------------------|
| 目录访问 | `get_uploads_dir()` + `mkdir` | `_get_uploads_dir()` + `mkdir` |
| 文件名安全 | 内联 `Path(f).name` + 手动检查 | 无检查，直接使用 `src_path.name` |
| 重复处理 | 无（覆盖） | 无（覆盖） |
| 列表 | 内联 `iterdir()` | 内联 `os.scandir()` |
| 删除 | 内联 `unlink()` + 遍历检查 | 内联 `unlink()` + 遍历检查 |
| 路径遍历 | `resolve().relative_to()` | `resolve().relative_to()` |

**相同的遍历检查写了两次**——任何安全修复必须同时应用于两个位置。

## 2. 设计原则

### 依赖方向

```
app.gateway.routers.skills  ──┐
app.gateway.routers.uploads ──┤── calls ──→  deerflow.skills.installer
deerflow.client             ──┘              deerflow.uploads.manager
```

- 共享模块位于 harness 层（`deerflow.*`），纯业务逻辑，无 FastAPI 依赖
- Gateway 处理 HTTP 适配（`UploadFile` → bytes，异常 → `HTTPException`）
- Client 处理本地适配（`Path` → copy，异常 → Python 异常）
- 满足 `test_harness_boundary.py` 约束：harness 永不导入 app

### 异常策略

| 共享层异常 | Gateway 映射到 | Client |
|----------------------|-----------------|--------|
| `FileNotFoundError` | `HTTPException(404)` | 传播 |
| `ValueError` | `HTTPException(400)` | 传播 |
| `SkillAlreadyExistsError` | `HTTPException(409)` | 传播 |
| `PermissionError` | `HTTPException(403)` | 传播 |

用类型化异常匹配（`SkillAlreadyExistsError`）替换字符串类型路由（`"already exists" in str(e)`）。

## 3. 新模块

### 3.1 `deerflow.skills.installer`

```python
# 安全检查
is_unsafe_zip_member(info: ZipInfo) -> bool     # 绝对路径 / .. 遍历
is_symlink_member(info: ZipInfo) -> bool         # Unix 符号链接检测
should_ignore_archive_entry(path: Path) -> bool  # __MACOSX / dotfiles

# 提取
safe_extract_skill_archive(zip_ref, dest_path, max_total_size=512MB)
  # 流式写入，累积真实字节（vs 声明的 file_size）
  # 双重遍历检查：成员级 + resolve 级

# 目录解析
resolve_skill_dir_from_archive(temp_path: Path) -> Path
  # 自动进入单个目录，过滤 macOS 元数据

# 安装入口点
install_skill_from_archive(zip_path, *, skills_root=None) -> dict
  # 扩展名验证前的 is_file() 预检查
  # SkillAlreadyExistsError 替换 ValueError

# 异常
class SkillAlreadyExistsError(ValueError)
```

### 3.2 `deerflow.uploads.manager`

```python
# 目录管理
get_uploads_dir(thread_id: str) -> Path      # 纯路径，无副作用
ensure_uploads_dir(thread_id: str) -> Path   # 创建目录（用于写路径）

# 文件名安全
normalize_filename(filename: str) -> str
  # Path.name 提取 + 拒绝 ".." / "." / 反斜杠 / >255 字节
deduplicate_filename(name: str, seen: set) -> str
  # _N 后缀递增去重，就地改变 seen

# 路径安全
validate_path_traversal(path: Path, base: Path) -> None
  # resolve().relative_to()，失败时引发 PermissionError

# 文件操作
list_files_in_dir(directory: Path) -> dict
  # 在上下文内使用 scandir + stat（不再 re-stat）
  # follow_symlinks=False 防止元数据泄漏
  # 不存在的目录返回空列表
delete_file_safe(base_dir: Path, filename: str) -> dict
  # 首先验证遍历，然后 unlink

# URL 辅助函数
upload_artifact_url(thread_id, filename) -> str   # 百分号编码以确保 HTTP 安全
upload_virtual_path(filename) -> str               # 沙箱内部路径
enrich_file_listing(result, thread_id) -> dict     # 添加 URL，字符串化大小
```

## 4. 更改

### 4.1 Gateway 精简

**`app/gateway/routers/skills.py`**：
- 移除 `_is_unsafe_zip_member`、`_is_symlink_member`、`_safe_extract_skill_archive`、`_should_ignore_archive_entry`、`_resolve_skill_dir_from_archive_root`（约 80 行）
- `install_skill` 路由变为对 `install_skill_from_archive(path)` 的单次调用
- 异常映射：`SkillAlreadyExistsError → 409`、`ValueError → 400`、`FileNotFoundError → 404`

**`app/gateway/routers/uploads.py`**：
- 移除内联 `get_uploads_dir`（由 `ensure_uploads_dir`/`get_uploads_dir` 替换）
- `upload_files` 使用 `normalize_filename()` 而不是内联安全检查
- `list_uploaded_files` 使用 `list_files_in_dir()` + 增强
- `delete_uploaded_file` 使用 `delete_file_safe()` + 配套的 markdown 清理

### 4.2 Client 精简

**`deerflow/client.py`**：
- 移除 `_get_uploads_dir` 静态方法
- 移除 `install_skill` 中约 50 行内联 zip 处理
- `install_skill` 委托给 `install_skill_from_archive()`
- `upload_files` 使用 `deduplicate_filename()` + `ensure_uploads_dir()`
- `list_uploads` 使用 `get_uploads_dir()` + `list_files_in_dir()`
- `delete_upload` 使用 `get_uploads_dir()` + `delete_file_safe()`
- `update_mcp_config`/`update_skill` 现在重置 `_agent_config_key = None`

### 4.3 读/写路径分离

| 操作 | 函数 | 创建目录？ |
|-----------|----------|:------------:|
| 上传（写） | `ensure_uploads_dir()` | 是 |
| 列表（读） | `get_uploads_dir()` | 否 |
| 删除（读） | `get_uploads_dir()` | 否 |

读路径不再有 `mkdir` 副作用——不存在的目录返回空列表。

## 5. 安全改进

| 改进 | 之前 | 之后 |
|-------------|--------|-------|
| Zip 炸弹检测 | 声明的 `file_size` 总和 | 流式写入，累积真实字节 |
| 符号链接处理 | Gateway 跳过 / Client 提取后删除 | 统一跳过 + 记录 |
| 遍历检查 | 仅成员级 | 成员级 + `resolve().is_relative_to()` |
| 文件名反斜杠 | Gateway 检查 / Client 不检查 | 统一拒绝 |
| 文件名长度 | 无检查 | 拒绝 > 255 字节（OS 限制） |
| thread_id 验证 | 无 | 拒绝不安全的文件系统字符 |
| 列表符号链接泄漏 | `follow_symlinks=True`（默认） | `follow_symlinks=False` |
| 409 状态路由 | `"already exists" in str(e)` | `SkillAlreadyExistsError` 类型匹配 |
| Artifact URL 编码 | URL 中使用原始文件名 | `urllib.parse.quote()` |

## 6. 考虑的替代方案

| 替代方案 | 为何不采用 |
|-------------|---------|
| 将逻辑保留在 Gateway，让 Client 通过 HTTP 调用 Gateway | 为嵌入式 Client 添加网络依赖；破坏 `DeerFlowClient` 作为进程内 API 的目的 |
| 抽象基类与 Gateway/Client 子类 | 对于纯函数来说过度工程；不需要多态 |
| 将所有内容移入 `client.py` 让 Gateway 导入它 | 违反 harness/app 边界——Client 在 harness 中，但 Gateway 特定的模型（Pydantic 响应类型）应留在 app 层 |
| 合并 Gateway 和 Client 为一个模块 | 它们服务不同的消费者（HTTP vs 进程内），具有不同的适配需求 |

## 7. 破坏性更改

**无。** 所有公共 API（Gateway HTTP 端点、`DeerFlowClient` 方法）保留其现有签名和返回格式。`SkillAlreadyExistsError` 是 `ValueError` 的子类，因此现有的 `except ValueError` 处理程序仍会捕获它。

## 8. 测试

| 模块 | 测试文件 | 数量 |
|--------|-----------|:-----:|
| `skills.installer` | `tests/test_skills_installer.py` | 22 |
| `uploads.manager` | `tests/test_uploads_manager.py` | 20 |
| `client` 加固 | `tests/test_client.py`（新用例） | ~40 |
| `client` 端到端 | `tests/test_client_e2e.py`（新文件） | ~20 |

覆盖率：不安全 zip / 符号链接 / zip 炸弹 / frontmatter / 重复 / 扩展名 / macOS 过滤 / 规范化 / 去重 / 遍历 / 列表 / 删除 / 代理失效 / 上传生命周期 / 线程隔离 / URL 编码 / 配置污染。
