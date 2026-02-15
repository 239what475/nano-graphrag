# `_merge_nodes_then_upsert` 详细讲解

## 函数定位

- 定义：`nano_graphrag/_op.py:182`
- 调用：`nano_graphrag/_op.py:392`
- 作用：把“同名实体在多个 chunk 的抽取结果”合并成一个节点，然后写回图存储。

## 这个函数解决的核心问题

LLM 是按 chunk 抽取的，同一个实体（如 `SCROOGE`）会在很多 chunk 里反复出现。
如果不合并，图里会有大量重复节点，后续检索和社区聚类都会变差。

这个函数就是“同名实体归并器”。

## 入参含义

1. `entity_name`：当前要合并的实体名（已做 uppercase 规范化）
2. `nodes_data`：这个实体在“本轮新 chunk”里抽到的多条记录
3. `knwoledge_graph_inst`：图存储实例（NetworkX/Neo4j）
4. `global_config` + `tokenizer_wrapper`：用于描述摘要时的模型和 token 控制

## 执行步骤（按代码顺序）

### 1) 先读旧节点（如果历史上已有）

```python
already_node = await knwoledge_graph_inst.get_node(entity_name)
```

位置：`nano_graphrag/_op.py:193`

如果存在，就把旧字段并入“候选池”：

- 旧 `entity_type`
- 旧 `source_id`（先按 `<SEP>` 拆开）
- 旧 `description`

位置：`nano_graphrag/_op.py:195` 到 `nano_graphrag/_op.py:199`

### 2) 合并 `entity_type`（多数投票）

```python
entity_type = sorted(
    Counter([dp["entity_type"] for dp in nodes_data] + already_entitiy_types).items(),
    key=lambda x: x[1],
    reverse=True,
)[0][0]
```

位置：`nano_graphrag/_op.py:201` 到 `nano_graphrag/_op.py:207`

含义：哪个类型出现次数最多就用哪个。

### 3) 合并 `description`（去重 + 拼接）

```python
description = GRAPH_FIELD_SEP.join(
    sorted(set([dp["description"] for dp in nodes_data] + already_description))
)
```

位置：`nano_graphrag/_op.py:208` 到 `nano_graphrag/_op.py:210`

这里 `GRAPH_FIELD_SEP` 是 `<SEP>`：`nano_graphrag/prompt.py:6`。

### 4) 合并 `source_id`（去重 + 拼接）

```python
source_id = GRAPH_FIELD_SEP.join(
    set([dp["source_id"] for dp in nodes_data] + already_source_ids)
)
```

位置：`nano_graphrag/_op.py:211` 到 `nano_graphrag/_op.py:213`

语义是“这个实体来自哪些 chunk”。

### 5) 描述太长时做摘要压缩

```python
description = await _handle_entity_relation_summary(
    entity_name, description, global_config, tokenizer_wrapper
)
```

位置：`nano_graphrag/_op.py:214` 到 `nano_graphrag/_op.py:216`

说明：

- 超过阈值才触发摘要
- 摘要用 `cheap_model_func`，不是抽取用的 `best_model_func`

### 6) upsert 到图节点

```python
await knwoledge_graph_inst.upsert_node(entity_name, node_data=node_data)
```

位置：`nano_graphrag/_op.py:222`

### 7) 返回值的用途

函数返回 `node_data`，并额外带上 `entity_name`：

```python
node_data["entity_name"] = entity_name
return node_data
```

位置：`nano_graphrag/_op.py:226` 到 `nano_graphrag/_op.py:227`

这个返回值后续会被用来写 `entity_vdb`（实体向量库）。

## 一个最小例子

假设：

- `entity_name = "SCROOGE"`
- `nodes_data` 有两条：
  - `{"entity_type": "\"PERSON\"", "description": "A", "source_id": "chunk-1"}`
  - `{"entity_type": "\"PERSON\"", "description": "B", "source_id": "chunk-2"}`
- 旧图里已有：
  - `entity_type="\"ORGANIZATION\""`
  - `description="B<SEP>C"`
  - `source_id="chunk-2<SEP>chunk-3"`

大致结果：

- `entity_type`：`"PERSON"`（多数投票）
- `source_id`：`chunk-1<SEP>chunk-2<SEP>chunk-3`（去重后）
- `description`：`A<SEP>B<SEP>B<SEP>C` 的“候选串”可能会再被摘要压缩（取决于长度阈值）

## 你需要特别注意的两个点

1. `source_id` 用 `set` 去重，顺序不保证稳定（通常不影响功能，但字符串顺序可能变化）。
2. `entity_type` 由频次决定，不是“首次出现优先”。

