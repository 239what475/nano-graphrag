# `_merge_edges_then_upsert` 详细讲解

## 函数定位

- 定义：`nano_graphrag/_op.py:230`
- 调用：`nano_graphrag/_op.py:398`
- 作用：把同一对实体的多条关系记录合并成一条边，并写回图存储。

## 这个函数要解决什么问题

实体关系是按 chunk 抽取的，同一对实体可能在多个 chunk 里反复出现，描述和强度也可能不同。
如果直接逐条写边，会产生重复边或信息覆盖混乱。

`_merge_edges_then_upsert` 做的是“边级归并 + 一次性 upsert”。

## 入参含义

1. `src_id`, `tgt_id`：边两端实体 ID（已在上游做了无向归一）
2. `edges_data`：本轮抽到的同一条边的多条记录
3. `knwoledge_graph_inst`：图存储实例
4. `global_config` + `tokenizer_wrapper`：用于描述摘要

## 上游前提（很关键）

在调用前，上游会先把边键做规范化：

```python
maybe_edges[tuple(sorted(k))].extend(v)
```

位置：`nano_graphrag/_op.py:389`

这保证 `(A,B)` 和 `(B,A)` 会合并到同一桶里，后续这个函数拿到的是“同一条无向边”的聚合输入。

## 执行步骤（按代码顺序）

### 1) 读取历史边（若已存在）

```python
if await knwoledge_graph_inst.has_edge(src_id, tgt_id):
    already_edge = await knwoledge_graph_inst.get_edge(src_id, tgt_id)
```

位置：`nano_graphrag/_op.py:242` 到 `nano_graphrag/_op.py:243`

若存在，就收集历史字段作为合并候选：

- 历史 `weight`
- 历史 `source_id`（先按 `<SEP>` 拆开）
- 历史 `description`
- 历史 `order`（没有就按 1）

位置：`nano_graphrag/_op.py:244` 到 `nano_graphrag/_op.py:249`

### 2) 合并规则

#### `order`: 取最小值

```python
order = min([dp.get("order", 1) for dp in edges_data] + already_order)
```

位置：`nano_graphrag/_op.py:252`

语义：`order=1` 表示更直接关系，合并后保留“最直接”的阶数。
注释里也写到 `order` 主要来自 DSPy 关系抽取：`nano_graphrag/_op.py:251`，
普通 prompt 解析路径默认没有这个字段，会回退到 1（`nano_graphrag/_op.py:252`）。

#### `weight`: 累加

```python
weight = sum([dp["weight"] for dp in edges_data] + already_weights)
```

位置：`nano_graphrag/_op.py:253`

语义：同一关系多次被提及，边权更高（用于后续关系排序）。

#### `description`: 去重后拼接

```python
description = GRAPH_FIELD_SEP.join(
    sorted(set([dp["description"] for dp in edges_data] + already_description))
)
```

位置：`nano_graphrag/_op.py:254` 到 `nano_graphrag/_op.py:256`

#### `source_id`: 去重后拼接

```python
source_id = GRAPH_FIELD_SEP.join(
    set([dp["source_id"] for dp in edges_data] + already_source_ids)
)
```

位置：`nano_graphrag/_op.py:257` 到 `nano_graphrag/_op.py:259`

含义：记录这条关系由哪些 chunk 支撑。

### 3) 防御性补节点

在写边前，会检查两端节点是否存在；不存在就创建 `UNKNOWN` 节点：

```python
for need_insert_id in [src_id, tgt_id]:
    if not (await knwoledge_graph_inst.has_node(need_insert_id)):
        await knwoledge_graph_inst.upsert_node(..., {"entity_type": '"UNKNOWN"', ...})
```

位置：`nano_graphrag/_op.py:260` 到 `nano_graphrag/_op.py:269`

这是为了防止“边引用了不存在节点”导致图结构不完整。

### 4) 描述过长时摘要

```python
description = await _handle_entity_relation_summary(
    (src_id, tgt_id), description, global_config, tokenizer_wrapper
)
```

位置：`nano_graphrag/_op.py:270` 到 `nano_graphrag/_op.py:272`

和节点合并一致：长描述会压缩，降低后续上下文成本。

### 5) 最终 upsert 边

```python
await knwoledge_graph_inst.upsert_edge(
    src_id, tgt_id,
    edge_data=dict(weight=weight, description=description, source_id=source_id, order=order),
)
```

位置：`nano_graphrag/_op.py:273` 到 `nano_graphrag/_op.py:278`

## 与后续检索的关系

- 在 local 检索里，边会按 `(rank, weight)` 排序，`weight` 参与优先级：`nano_graphrag/_op.py:833`
- 关系结果展示会带上 `weight`：`nano_graphrag/_op.py:899`

所以这里的 `weight` 累加不是装饰字段，后续会直接影响检索上下文的构成。

## 一个最小例子

假设同一边 `("SCROOGE", "MARLEY")`：

- 本轮抽取两条关系：
  - `weight=0.8`, `description="former business partners"`, `source_id=chunk-a`
  - `weight=0.6`, `description="linked by shared debt"`, `source_id=chunk-b`
- 历史边：
  - `weight=1.0`, `description="former business partners"`, `source_id=chunk-c`, `order=2`

合并后大致：

1. `weight = 0.8 + 0.6 + 1.0 = 2.4`
2. `description` 去重拼接（两条描述）
3. `source_id` 包含 `chunk-a/chunk-b/chunk-c`
4. `order = min(1(默认), 1(默认), 2) = 1`

## 注意点

1. `weight` 是累加型，可能超过 1（即使单条抽取时常在 0~1 区间）。
2. `source_id` 合并使用 `set`，顺序不稳定，但不影响语义。
3. Neo4j 后端内部是 `MERGE (s)-[r:RELATED]->(t)`，但上游已通过排序规范化边方向，保证同一无向关系不会分裂成两条反向边。参考：`nano_graphrag/_storage/gdb_neo4j.py:378`。

