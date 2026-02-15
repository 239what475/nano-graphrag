# 图存储与 GraphML（chunk_entity_relation）

## 文件作用

`graph_chunk_entity_relation.graphml` 是实体关系图（知识图谱）的持久化缓存，不是原始文本。

## 生成与加载

1. 图对象初始化：
   `self.chunk_entity_relation_graph = self.graph_storage_cls(namespace="chunk_entity_relation", ...)`
   位置：`nano_graphrag/graphrag.py:192`
2. 默认图后端：
   `NetworkXStorage`
   位置：`nano_graphrag/graphrag.py:132`
3. GraphML 文件路径：
   `working_dir/graph_chunk_entity_relation.graphml`
   位置：`nano_graphrag/_storage/gdb_networkx.py:81`
4. 若文件存在则加载：
   `nx.read_graphml(...)`
   位置：`nano_graphrag/_storage/gdb_networkx.py:24`, `nano_graphrag/_storage/gdb_networkx.py:84`
5. 插入完成后写回：
   `nx.write_graphml(...)`
   位置：`nano_graphrag/_storage/gdb_networkx.py:32`, `nano_graphrag/_storage/gdb_networkx.py:97`

## 在检索中的作用

- `local` 查询会使用这张图：
  `nano_graphrag/graphrag.py:242`
- `global` 查询会使用这张图：
  `nano_graphrag/graphrag.py:253`
- 社区报告生成依赖图聚类：
  `nano_graphrag/graphrag.py:337`, `nano_graphrag/_op.py:625`

## 数据字段（GraphML 中常见）

- 节点字段：
  `entity_type`, `description`, `source_id`, `clusters`
- 边字段：
  `weight`, `description`, `source_id`, `order`

这些字段来自实体/关系抽取与合并逻辑：
`nano_graphrag/_op.py:138`, `nano_graphrag/_op.py:159`, `nano_graphrag/_op.py:182`, `nano_graphrag/_op.py:230`

## 删除后影响

- 删除该文件不会破坏代码结构，但会丢失已构建图
- 需要重新 `insert(...)` 才会重建
- 示例里也有先删再重建的流程：
  `examples/no_openai_key_at_all.py:93`

