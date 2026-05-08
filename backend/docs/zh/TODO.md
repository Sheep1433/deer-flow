# 待办事项

## 已完成功能

- [x] 仅在首次调用文件系统或 bash 工具时才启动沙箱
- [x] 为整个流程添加澄清过程
- [x] 实现上下文摘要机制以避免上下文膨胀
- [x] 集成 MCP（Model Context Protocol）以支持可扩展工具
- [x] 添加文件上传支持（自动文档转换）
- [x] 实现自动线程标题生成
- [x] 添加计划模式（TodoList 中间件）
- [x] 添加视觉模型支持（ViewImageMiddleware）
- [x] 技能系统（SKILL.md 格式）
- [x] 在 `packages/harness/deerflow/tools/builtins/task_tool.py`（子代理轮询）中用 `asyncio.sleep()` 替换 `time.sleep(5)`

## 计划中的功能

- [ ] 池化沙箱资源以减少沙箱容器数量
- [ ] 添加认证/授权层
- [ ] 实现速率限制
- [ ] 添加指标和监控
- [ ] 上传支持更多文档格式
- [ ] 技能市场 / 远程技能安装
- [ ] 优化代理热路径中的异步并发（IM 渠道多任务场景）
- [ ] 在 `packages/harness/deerflow/sandbox/local/local_sandbox.py` 中用 `asyncio.create_subprocess_shell()` 替换 `subprocess.run()`
  - 在社区工具中用 `httpx.AsyncClient` 替换同步的 `requests`（tavily、jina_ai、firecrawl、infoquest、image_search）
  - [x] 在 title_middleware 和 memory updater 中用异步的 `model.ainvoke()` 替换同步的 `model.invoke()`
  - 考虑为剩余的阻塞文件 I/O 使用 `asyncio.to_thread()` 包装器
  - 生产环境：使用 `langgraph up`（多 worker）而非 `langgraph dev`（单 worker）

## 已解决的问题

- [x] 确保 `state.artifacts` 中没有重复文件
- [x] 长时间思考但内容为空（答案在思考过程中）