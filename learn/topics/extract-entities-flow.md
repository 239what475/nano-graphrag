# `extract_entities` 逻辑拆解

## 函数定位

- 入口函数：`nano_graphrag/_op.py:282`
- 默认由 `GraphRAG` 调用：`nano_graphrag/graphrag.py:323`

## 输入与输出

输入参数（核心）：

1. `chunks`: `chunk_id -> {content, ...}` 的字典
2. `knwoledge_graph_inst`: 图存储实例（默认 `NetworkXStorage`）
3. `entity_vdb`: 实体向量库（可选，`enable_local=False` 时可能为 `None`）
4. `tokenizer_wrapper` + `global_config`: 控制 tokenizer、模型函数和阈值

输出：

- 成功：返回更新后的 `knwoledge_graph_inst`
- 失败：若没有抽到任何实体，返回 `None`（`nano_graphrag/_op.py:402`）

## 执行流程（按时间顺序）

### 1) 准备 prompt 与配置

- 取用于抽取的模型函数 `best_model_func`：`nano_graphrag/_op.py:290`
- 读取抽取 prompt 模板 `PROMPTS["entity_extraction"]`：`nano_graphrag/_op.py:295`
- 组装分隔符（tuple/record/completion）与实体类型：`nano_graphrag/_op.py:296`
- 将 `chunks` 转成有序列表：`nano_graphrag/_op.py:293`

### 2) 每个 chunk 调 LLM 抽取（并发）

内部函数 `_process_single_content(...)` 会对单个 chunk 做：

1. 用 `entity_extraction` prompt 让 LLM 输出结构化记录：`nano_graphrag/_op.py:314`
2. 可选“补抽”循环（gleaning）：
   - 继续补充：`PROMPTS["entiti_continue_extraction"]`，`nano_graphrag/_op.py:321`
   - 是否继续：`PROMPTS["entiti_if_loop_extraction"]`，`nano_graphrag/_op.py:328`
3. 把输出按 record/completion 分隔符切开：`nano_graphrag/_op.py:335`
4. 逐条解析：
   - 实体走 `_handle_single_entity_extraction`：`nano_graphrag/_op.py:350`
   - 关系走 `_handle_single_relationship_extraction`：`nano_graphrag/_op.py:357`
5. 返回该 chunk 的局部结果 `maybe_nodes/maybe_edges`：`nano_graphrag/_op.py:375`

外层用 `asyncio.gather` 并发所有 chunk：`nano_graphrag/_op.py:378`

## 并发上限说明

虽然这里并发提交所有 chunk，但 `best_model_func` 在 `GraphRAG.__post_init__` 中已被限流包装：

- 包装位置：`nano_graphrag/graphrag.py:219`
- 限流实现：`nano_graphrag/_utils.py:276`

这能避免一次性把 LLM API 压爆。

### 3) 合并跨 chunk 的重复实体/关系

并发结果汇总后，按 name/edge key 合并：

- 节点：同名实体聚合，类型按“出现次数最多”选，描述去重拼接，`source_id` 合并
  - 逻辑：`_merge_nodes_then_upsert`，`nano_graphrag/_op.py:182`
- 边：同一对实体（无向，先排序）聚合，`weight` 求和，`order` 取最小
  - 逻辑：`_merge_edges_then_upsert`，`nano_graphrag/_op.py:230`
  - 无向处理：`tuple(sorted(k))`，`nano_graphrag/_op.py:389`

如果关系引用了不存在节点，会先补一个 `UNKNOWN` 类型节点：`nano_graphrag/_op.py:260`

### 4) 描述过长时做摘要

节点/边合并后的 description 可能很长，会走 `_handle_entity_relation_summary(...)`：

- 触发判定：超过 `entity_summary_to_max_tokens`：`nano_graphrag/_op.py:123`
- 使用 `cheap_model_func` 生成摘要：`nano_graphrag/_op.py:117`, `nano_graphrag/_op.py:134`
- 摘要 prompt：`PROMPTS["summarize_entity_descriptions"]`（`nano_graphrag/prompt.py:297`）

也就是：抽取用“大模型”，摘要压缩用“便宜模型”。

### 5) 写入图 + （可选）写入实体向量库

- 节点写图：`upsert_node(...)`，`nano_graphrag/_op.py:222`
- 边写图：`upsert_edge(...)`，`nano_graphrag/_op.py:273`
- 若 `entity_vdb` 存在，把 `entity_name + description` 做 embedding 并 upsert：
  `nano_graphrag/_op.py:405`

注意：这一步是“实体 embedding”，不是 chunk embedding。chunk embedding 只在 naive 模式里做。

## 少量关键代码

```python
# 1) 并发处理每个 chunk
results = await asyncio.gather(
    *[_process_single_content(c) for c in ordered_chunks]
)
```

```python
# 2) 合并后写图（节点）
await knwoledge_graph_inst.upsert_node(
    entity_name,
    node_data=node_data,
)
```

```python
# 3) 可选写实体向量
if entity_vdb is not None:
    await entity_vdb.upsert(data_for_vdb)
```

## 为什么这么设计

1. 先抽结构（实体+关系），再检索时可结合图结构做多跳与社区推理
2. 先合并再 embedding，减少重复实体向量计算
3. 用分隔符格式约束 LLM 输出，简化后处理

