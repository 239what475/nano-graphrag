# `local_query` 详细流程

## 入口与定位

- 定义：`nano_graphrag/_op.py:935`
- 调用：`GraphRAG.aquery(mode="local")`，`nano_graphrag/graphrag.py:242`

`local_query` 的核心是“实体中心检索”：

1. 先从实体向量库召回与问题最相关的实体
2. 再围绕这些实体扩展社区、关系、文本证据
3. 最终拼成结构化上下文交给 LLM 回答

## 主流程（按执行顺序）

### 1) 构建 local 上下文

`local_query` 第一件事是调用：

```python
context = await _build_local_query_context(...)
```

位置：`nano_graphrag/_op.py:946`

如果 `only_need_context=True`，直接返回这个上下文，不走生成：`nano_graphrag/_op.py:955`

如果上下文为空，返回 `fail_response`：`nano_graphrag/_op.py:957`

### 2) 用 local prompt 生成答案

使用 `PROMPTS["local_rag_response"]`，把 `context_data + response_type` 填进去，然后让 `best_model_func` 回答。

位置：`nano_graphrag/_op.py:959`

## `_build_local_query_context` 做了什么

定义：`nano_graphrag/_op.py:844`

### A. 先做实体召回

```python
results = await entities_vdb.query(query, top_k=query_param.top_k)
```

位置：`nano_graphrag/_op.py:853`

这一步拿到的是与 query 语义最接近的实体（来自 `vdb_entities.json`）。

### B. 取实体节点详情 + 度数

- 从图里拉节点属性：`get_nodes_batch`
- 计算每个实体度数：`node_degrees_batch`
- 组装 `node_datas`（含 `entity_name` 和 `rank`）

位置：`nano_graphrag/_op.py:856` 到 `nano_graphrag/_op.py:863`

这里 `rank` 实际是节点度数（连接度），不是向量分数。

### C. 并行扩展三类上下文

1. 相关社区：`_find_most_related_community_from_entities`
2. 相关文本：`_find_most_related_text_unit_from_entities`
3. 相关关系：`_find_most_related_edges_from_entities`

位置：`nano_graphrag/_op.py:865` 到 `nano_graphrag/_op.py:873`

### D. 组装四段 CSV 上下文

最终返回四段：

1. `Reports`（社区报告）
2. `Entities`（实体表）
3. `Relationships`（关系表）
4. `Sources`（文本块）

位置：`nano_graphrag/_op.py:915`

## 三个子检索函数的策略

## 1) `_find_most_related_community_from_entities`

定义：`nano_graphrag/_op.py:700`

逻辑：

1. 从每个命中实体的 `clusters` 收集社区 ID
2. 按出现频次统计（实体越多指向该社区，优先级越高）
3. 次级排序看社区 `rating`
4. 按 token 预算截断社区报告文本
5. 可选只留 1 个社区（`local_community_single_one=True`）

## 2) `_find_most_related_text_unit_from_entities`

定义：`nano_graphrag/_op.py:748`

逻辑：

1. 先拿命中实体自身的 `source_id`（直接证据 chunk）
2. 再看实体的一跳邻居，统计每个 chunk 的“关系支持计数”
3. 给每个候选 chunk 打两个排序维度：
   - `order`：来自第几个召回实体（越靠前越优先）
   - `relation_counts`：被关系邻居支持的程度
4. 排序键为 `(order, -relation_counts)`，然后按 token 预算截断

## 3) `_find_most_related_edges_from_entities`

定义：`nano_graphrag/_op.py:807`

逻辑：

1. 收集命中实体相关边，先去重（无向归一）
2. 拉边属性与边度数
3. 排序键 `(rank, weight)`，降序
4. 按关系描述 token 预算截断

## local prompt 在这一步的职责

`PROMPTS["local_rag_response"]`（`nano_graphrag/prompt.py:330`）要求模型：

1. 基于数据表作答
2. 信息不足就说不知道
3. 不编造
4. 输出符合 `response_type` 的长度和格式（Markdown）

它本质是“证据约束的答案生成器”。

## 关键参数（`QueryParam`）

定义：`nano_graphrag/base.py:9`

local 相关：

1. `top_k`
2. `level`
3. `local_max_token_for_text_unit`
4. `local_max_token_for_local_context`
5. `local_max_token_for_community_report`
6. `local_community_single_one`

这些参数直接决定召回范围与上下文预算。

## 常见误解

1. 误以为 local 只看向量相似度  
   - 实际只在第一步用实体向量召回，后续 heavily 依赖图关系与社区。
2. 误以为 `rank` 是 embedding 相似度  
   - 这里多数 `rank` 来自图度数（节点度/边度）。
3. 误以为 local 一定比 global 更准  
   - local 更偏“具体实体问题”；全局主题问题通常 global 更稳。

