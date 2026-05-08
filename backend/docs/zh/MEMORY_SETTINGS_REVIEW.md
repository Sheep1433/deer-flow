# 内存设置审查

使用本指南可以用最少的步骤在本地审查内存设置添加/编辑流程。

## 快速审查

1. 使用任意您已有的工作开发设置在本地启动 DeerFlow。

   示例：

   ```bash
   make dev
   ```

   或

   ```bash
   make docker-start
   ```

   如果您已经在本地运行 DeerFlow，可以重用现有设置。

2. 加载示例内存 fixture。

   ```bash
   python scripts/load_memory_sample.py
   ```

3. 打开 `设置 > 内存`。

   默认本地 URL：
   - 应用：`http://localhost:2026`
   - 本地仅前端回退：`http://localhost:3000`

## 最小手动测试

1. 点击 `添加事实`。
2. 创建新事实：
   - 内容：`Reviewer-added memory fact`
   - 类别：`testing`
   - 置信度：`0.88`
3. 确认新事实立即出现并显示 `Manual` 作为来源。
4. 编辑示例事实 `This sample fact is intended for edit testing.` 并将其更改为：
   - 内容：`This sample fact was edited during manual review.`
   - 类别：`testing`
   - 置信度：`0.91`
5. 确认编辑的事实立即更新。
6. 刷新页面并确认新添加的事实和编辑的事实都持续存在。

## 可选的一致性检查

- 搜索 `Reviewer-added` 并确认新事实被匹配。
- 搜索 `workflow` 并确认类别文本可搜索。
- 在 `All`、`Facts` 和 `Summaries` 之间切换。
- 删除一次性示例事实 `Delete fact testing can target this disposable sample entry.` 并确认列表立即更新。
- 清除所有内存并确认页面进入空状态。

## Fixture 文件

- 示例 fixture：`backend/docs/memory-settings-sample.json`
- 默认本地运行时目标：`backend/.deer-flow/memory.json`

加载脚本在覆盖现有运行时内存文件之前会自动创建时间戳备份。