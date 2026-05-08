# 内存系统改进

本文档跟踪内存注入行为和路线图状态。

## 状态（截至 2026-03-10）

在 `main` 中已实现：
- 通过 `tiktoken` 在 `format_memory_for_injection` 中进行精确 token 计数。
- 事实注入到提示内存上下文中。
- 事实按置信度排序（降序）。
- 注入遵守 `max_injection_tokens` 预算。

计划中 / 尚未合并：
- 基于 TF-IDF 相似度的事实检索。
- 用于上下文感知评分的 `current_context` 输入。
- 可配置的相似度/置信度权重（`similarity_weight`、`confidence_weight`）。
- 每次模型调用前用于上下文感知检索的中间件/运行时连接。

## 当前行为

今天的函数：

```python
def format_memory_for_injection(memory_data: dict[str, Any], max_tokens: int = 2000) -> str:
```

当前注入格式：
- 来自 `user.*.summary` 的 `User Context` 部分
- 来自 `history.*.summary` 的 `History` 部分
- 来自 `facts[]` 的 `Facts` 部分，按置信度排序，在 token 预算内追加

Token 计数：
- 优先使用 `tiktoken`（`cl100k_base`）
- 如果 tokenizer 导入失败，回退到 `len(text) // 4`

## 已知差距

之前版本的本文档描述 TF-IDF/上下文感知检索好像已经发布。
这对于 `main` 并不准确，并造成了混淆。

Issue 参考：`#1059`

## 路线图（计划中）

计划的评分策略：

```text
final_score = (similarity * 0.6) + (confidence * 0.4)
```

计划的集成方式：
1. 从过滤后的用户/最终助手回合中提取最近的对话上下文。
2. 计算每个事实与当前上下文之间的 TF-IDF 余弦相似度。
3. 按加权分数排名并在 token 预算内注入。
4. 如果上下文不可用，回退到仅置信度排名。

## 验证

当前回归覆盖包括：
- 事实包含在内存注入输出中
- 置信度排序
- Token 预算限制的事实包含

测试：
- `backend/tests/test_memory_prompt_injection.py`