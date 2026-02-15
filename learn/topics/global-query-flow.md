# `global_query` 详细流程

## 入口与定位

- 定义：`nano_graphrag/_op.py:1017`
- 调用：`GraphRAG.aquery(mode="global")`，`nano_graphrag/graphrag.py:253`

`global_query` 的设计是“社区中心 + map-reduce”：

1. 先筛选和排序社区报告
2. 按 token 预算分组做 map
3. 聚合 point 做 reduce 输出最终答案

## 输入参数（真正会用到）

- `knowledge_graph_inst`：用于获取社区 schema
- `community_reports`：读取预先生成的社区报告
- `query_param`：控制 level、过滤阈值、token 预算等
- `global_config`：提供模型函数与 JSON 解析函数
- `tokenizer_wrapper`：用于 token 截断

说明：函数签名里有 `entities_vdb` / `text_chunks_db`，但当前实现中 `global_query` 并未使用它们（接口层面的保留参数）。

## 主流程（逐步）

### 1) 获取社区 schema 并按层级过滤

```python
community_schema = await knowledge_graph_inst.community_schema()
community_schema = {k: v for k, v in community_schema.items() if v["level"] <= query_param.level}
```

位置：`nano_graphrag/_op.py:1027`

如果过滤后为空，直接返回 `fail_response`：`nano_graphrag/_op.py:1031`

### 2) 按 occurrence 先做一轮候选排序

```python
sorted_community_schemas = sorted(..., key=lambda x: x[1]["occurrence"], reverse=True)
sorted_community_schemas = sorted_community_schemas[: query_param.global_max_consider_community]
```

位置：`nano_graphrag/_op.py:1035`

这一步是“结构先验”：优先覆盖更多 source chunk 的社区。

### 3) 读取社区报告并按 rating 过滤

```python
community_datas = await community_reports.get_by_ids([...])
community_datas = [c for c in community_datas if c is not None]
community_datas = [c for c in community_datas if c["report_json"].get("rating", 0) >= query_param.global_min_community_rating]
```

位置：`nano_graphrag/_op.py:1043`

然后再按 `(occurrence, rating)` 排序：`nano_graphrag/_op.py:1052`

### 4) map 阶段：把社区分组并并发提炼要点

调用：`_map_global_communities(...)`，`nano_graphrag/_op.py:1059`

内部逻辑（`nano_graphrag/_op.py:970`）：

1. 把社区报告按 `global_max_token_for_community_report` 分组
2. 每组构造 CSV：`id, content, rating, importance`
3. 用 `PROMPTS["global_map_rag_points"]` 让 LLM 输出 JSON points：
   - 每个 point: `description + score(0-100)`
4. 各组并发执行（`asyncio.gather`）

因此 map 阶段是“多分析师并行提炼”。代码里用组索引当 analyst id。

### 5) 聚合 map 结果并清洗

```python
for i, mc in enumerate(map_communities_points):
    for point in mc:
        ...
        final_support_points.append({"analyst": i, "answer": point["description"], "score": point.get("score", 1)})
```

位置：`nano_graphrag/_op.py:1062`

后续处理：

1. 去掉 `score <= 0` 的点
2. 按 `score` 降序排序
3. 按 token 预算截断答案文本

位置：`nano_graphrag/_op.py:1074` 到 `nano_graphrag/_op.py:1085`

### 6) 构造 reduce 输入并生成最终答案

把 points 组装成：

```text
----Analyst i----
Importance Score: X
<point text>
```

位置：`nano_graphrag/_op.py:1087`

若 `only_need_context=True`，这里直接返回 `points_context`：`nano_graphrag/_op.py:1095`

否则用 `PROMPTS["global_reduce_rag_response"]` 做最终融合生成：`nano_graphrag/_op.py:1097`

## 参数如何影响行为

关键参数定义在 `QueryParam`：`nano_graphrag/base.py:9`

1. `level`：允许参与的社区层级上限
2. `global_max_consider_community`：最多考虑多少社区
3. `global_min_community_rating`：rating 过滤阈值
4. `global_max_token_for_community_report`：map 分组预算和后续 points 截断预算
5. `global_special_community_map_llm_kwargs`：map 阶段模型参数（默认 JSON 输出）

## 为什么这是 map-reduce

1. Map：每个社区组独立提炼关键点（并发）
2. Reduce：把所有关键点再综合成最终回答（单次融合）

这能降低“单次提示词过长导致信息丢失”的风险，也更适合大图场景。

## 常见误解

1. 误以为 global 会直接检索原始 chunk
   - 实际上它主要看“社区报告”，不是直接按 chunk 向量召回。
2. 误以为 rating 是硬指标
   - rating 是 LLM 社区报告产物的软信号，仅用于辅助过滤/排序。
3. 误以为 entities_vdb 会参与 global
   - 当前实现中 global_query 没用 `entities_vdb`。

