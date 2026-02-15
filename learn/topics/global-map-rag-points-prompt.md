# `global_map_rag_points` Prompt（中文翻译与讲解）

## 定位

- Prompt 键：`PROMPTS["global_map_rag_points"]`
- 文件：`nano_graphrag/prompt.py:368`
- 使用位置：`_map_global_communities(...)` 中 map 阶段，`nano_graphrag/_op.py:1002`

## 中文完整翻译

### 角色（Role）

你是一个有帮助的助手，负责回答关于所提供表格数据的问题。

### 目标（Goal）

生成一个“关键要点列表”形式的回答，以响应用户问题，并总结输入数据表中所有相关信息。

你应当以下方数据表中的信息作为生成回答的主要上下文。  
如果你不知道答案，或者输入数据表不足以支持作答，请直接说明。不要编造。

回答中的每个关键要点应包含以下元素：

- Description：对该要点的完整描述。
- Importance Score：0-100 的整数分数，表示该要点对回答用户问题的重要程度。若是“我不知道”类回答，分数应为 0。

回答应按如下 JSON 格式输出：

```json
{
  "points": [
    {"description": "要点1描述...", "score": score_value},
    {"description": "要点2描述...", "score": score_value}
  ]
}
```

回答应保留原始含义，以及诸如 "shall"、"may"、"will" 等情态动词的用法。  
不要包含没有证据支持的信息。

### 数据表（Data tables）

`{context_data}`

### 目标（Goal，重复段）

生成一个“关键要点列表”形式的回答，以响应用户问题，并总结输入数据表中所有相关信息。

你应当以下方数据表中的信息作为生成回答的主要上下文。  
如果你不知道答案，或者输入数据表不足以支持作答，请直接说明。不要编造。

回答中的每个关键要点应包含以下元素：

- Description：对该要点的完整描述。
- Importance Score：0-100 的整数分数，表示该要点对回答用户问题的重要程度。若是“我不知道”类回答，分数应为 0。

回答应保留原始含义，以及诸如 "shall"、"may"、"will" 等情态动词的用法。  
不要包含没有证据支持的信息。

回答应按如下 JSON 格式输出：

```json
{
  "points": [
    {"description": "要点1描述", "score": score_value},
    {"description": "要点2描述", "score": score_value}
  ]
}
```

## 讲解

## 这段 prompt 在 map 阶段做什么

它的任务不是输出“最终答案”，而是先做“候选要点提炼”：

1. 输入一组社区报告（表格形式）
2. 让模型提炼若干 `points`
3. 每个 point 给一个 0-100 的重要度 `score`

之后这些 points 会被上层逻辑聚合、排序、截断，再进入 reduce 阶段生成最终答案。

## 输入上下文长什么样

在 `_map_global_communities` 里，`context_data` 由以下列组成：

- `id`
- `content`（社区报告文本）
- `rating`（社区评分）
- `importance`（社区 occurrence）

对应：`nano_graphrag/_op.py:991`

## 输出为何要求 JSON

上层代码会把 map 输出解析为 JSON 并读取 `points` 字段：

- 解析：`nano_graphrag/_op.py:1009`
- 读取：`nano_graphrag/_op.py:1010`

所以必须保证可解析、字段稳定。

## 为什么要 score

map 产出的 `score` 是“与当前 query 的相关重要度”，后续会用于：

1. 过滤 `score <= 0` 的点
2. 按分数降序排序
3. 截断保留高价值要点

对应：`nano_graphrag/_op.py:1074` 到 `nano_graphrag/_op.py:1085`

## 为什么 prompt 里有重复 Goal 段

你会看到同一目标说明出现两次，这是模板层面的冗余写法（历史沿用）。  
通常不会破坏功能，但会增加 token 消耗；若要优化成本，可考虑去重并做 A/B 测试验证效果。

