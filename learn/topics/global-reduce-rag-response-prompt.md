# `global_reduce_rag_response` Prompt（中文翻译与讲解）

## 定位

- Prompt 键：`PROMPTS["global_reduce_rag_response"]`
- 文件：`nano_graphrag/prompt.py:425`
- 使用位置：`global_query` 的 reduce 阶段，`nano_graphrag/_op.py:1097`

## 中文完整翻译

### 角色（Role）

你是一个有帮助的助手，负责通过综合多位分析师的视角来回答关于某数据集的问题。

### 目标（Goal）

生成一个符合目标长度与格式的回答，响应用户问题，并总结多位分析师的全部报告（这些分析师分别关注了数据集的不同部分）。

请注意：下面提供的分析师报告按**重要性降序**排列。

如果你不知道答案，或者提供的报告不足以支持作答，请直接说明。不要编造。

最终回答应移除分析师报告中的无关信息，并将清洗后的信息合并为一个全面答案，且应根据目标长度与格式，解释所有关键要点及其影响。

可根据目标长度与格式添加合适的小节和评论。请使用 Markdown 风格。

回答应保留原始含义，以及诸如 "shall"、"may"、"will" 等情态动词的用法。

不要包含没有证据支持的信息。

### 目标回答长度与格式（Target response length and format）

`{response_type}`

### 分析师报告（Analyst Reports）

`{report_data}`

### 目标（Goal，重复段）

生成一个符合目标长度与格式的回答，响应用户问题，并总结多位分析师的全部报告（这些分析师分别关注了数据集的不同部分）。

请注意：下面提供的分析师报告按**重要性降序**排列。

如果你不知道答案，或者提供的报告不足以支持作答，请直接说明。不要编造。

最终回答应移除分析师报告中的无关信息，并将清洗后的信息合并为一个全面答案，且应根据目标长度与格式，解释所有关键要点及其影响。

回答应保留原始含义，以及诸如 "shall"、"may"、"will" 等情态动词的用法。

不要包含没有证据支持的信息。

### 目标回答长度与格式（Target response length and format）

`{response_type}`

可根据目标长度与格式添加合适的小节和评论。请使用 Markdown 风格。

## 讲解

## 这个 prompt 在 reduce 阶段做什么

它的职责是“最终融合回答”，不是再做证据检索。

上游 map 阶段已经产出很多 `point`（带 score），并组装成：

```text
----Analyst i----
Importance Score: X
<point text>
```

然后 reduce prompt 负责把这些分析师片段合并成面向用户的最终答案。

对应调用：`nano_graphrag/_op.py:1097`

## 它与 map prompt 的分工

1. map (`global_map_rag_points`)：提炼候选要点并打分（结构化中间结果）
2. reduce (`global_reduce_rag_response`)：把高分要点整合成最终自然语言答案

所以 map 偏“抽取和评分”，reduce 偏“综合与表达”。

## 为什么强调“按重要性降序”

上游已经按 `score` 排序和截断（`nano_graphrag/_op.py:1077`），  
reduce 里再次强调排序，是为了让模型优先吸收高价值点，减少长上下文稀释。

## 为什么强调“移除无关信息”

map 阶段可能有冗余点。reduce 要做信息清洗与去重，避免最终回答啰嗦或冲突。

## 为什么会重复 Goal 段

你会看到 prompt 中目标段重复一次。这是模板冗余（历史沿用），通常不影响功能，但会增加 token 消耗。  
若你要优化成本，可尝试去重后做 A/B 对比评估质量变化。

