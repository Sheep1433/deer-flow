# 内存系统改进 - 摘要

## 同步说明（2026-03-10）

本摘要与 `main` 分支实现同步。
TF-IDF/上下文感知检索已**计划**，但尚未合并。

## 已实现

- 使用 `tiktoken` 进行精确 token 计数（内存注入）。
- 事实注入到 `<memory>` 提示内容中。
- 事实按置信度排序，受 `max_injection_tokens` 限制。

## 计划中（尚未合并）

- 基于 TF-IDF 余弦相似度的最近对话上下文召回。
- `current_context` 参数用于 `format_memory_for_injection`。
- 加权排名（`similarity` + `confidence`）。
- 运行时提取/注入流程用于上下文感知的事实选择。

## 为什么要进行此次同步

早期文档描述的 TF-IDF 行为已实现，但这与 `main` 中的代码不匹配。
此不匹配问题在 issue `#1059` 中跟踪。

## 当前 API 形状

```python
def format_memory_for_injection(memory_data: dict[str, Any], max_tokens: int = 2000) -> str:
```

当前 `main` 中没有 `current_context` 参数。

## 验证要点

- 实现：`packages/harness/deerflow/agents/memory/prompt.py`
- 提示组装：`packages/harness/deerflow/agents/lead_agent/prompt.py`
- 回归测试：`backend/tests/test_memory_prompt_injection.py`