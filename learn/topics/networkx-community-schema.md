# `NetworkXStorage.community_schema` 讲解

## 函数定位

- 定义：`nano_graphrag/_storage/gdb_networkx.py:170`
- 返回类型：`dict[str, SingleCommunitySchema]`
- `SingleCommunitySchema` 字段定义：`nano_graphrag/base.py:37`

## 这个函数在做什么

一句话：把“节点上已有的 `clusters` 标签”聚合成“社区视图”。

它不跑聚类算法；聚类在 `_leiden_clustering` 已经完成。`community_schema` 只负责汇总：

1. 每个社区有哪些节点
2. 每个社区有哪些边
3. 每个社区覆盖了哪些文本 chunk（`chunk_ids`）
4. 各层级社区的父子关系（`sub_communities`）
5. 一个归一化重要度指标 `occurrence`

## 输入依赖

它依赖每个节点已经有 `clusters` 字段（JSON 字符串），字段由聚类阶段写入：

- 写入位置：`nano_graphrag/_storage/gdb_networkx.py:226`
- 若节点无 `clusters`，会直接跳过：`nano_graphrag/_storage/gdb_networkx.py:185`

## 核心步骤

### 1) 初始化聚合容器

`results` 是 `defaultdict`，每个社区先放：

- `level`
- `title`
- `edges`（set）
- `nodes`（set）
- `chunk_ids`（set）
- `occurrence`（float）
- `sub_communities`（list）

位置：`nano_graphrag/_storage/gdb_networkx.py:171`

### 2) 遍历所有节点，把节点贡献到所属社区

对每个节点：

1. 读取 `clusters = json.loads(node_data["clusters"])`（可能有多层）
2. 取该节点相关边 `this_node_edges`
3. 对该节点所属的每个 cluster：
   - 记录社区层级与标题
   - 加入当前节点
   - 把节点相关边加入社区（边端点先排序去方向）
   - 把节点的 `source_id` 拆成 `chunk_ids` 加入社区
   - 更新全局最大 `chunk_ids` 数，用于后面归一化

对应：`nano_graphrag/_storage/gdb_networkx.py:184` 到 `nano_graphrag/_storage/gdb_networkx.py:204`

### 3) 计算子社区关系

按层级从低到高比较相邻两层：

- 若下一层社区 `c` 的节点集合是当前层社区 `comm` 节点集合的子集，
  则 `c` 是 `comm` 的子社区。

对应：`nano_graphrag/_storage/gdb_networkx.py:205` 到 `nano_graphrag/_storage/gdb_networkx.py:216`

### 4) 序列化与归一化

最后把 set 转成 list，并计算：

`occurrence = len(v["chunk_ids"]) / max_num_ids`

对应：`nano_graphrag/_storage/gdb_networkx.py:218` 到 `nano_graphrag/_storage/gdb_networkx.py:223`

这个 `occurrence` 是“该社区覆盖 chunk 数”相对“最大社区覆盖 chunk 数”的归一化比例。

## 输出长什么样

返回值是：

- key：`cluster_id`（字符串）
- value：`SingleCommunitySchema`
  - `level`, `title`
  - `nodes`, `edges`
  - `chunk_ids`
  - `sub_communities`
  - `occurrence`

## 下游谁会用它

1. 生成社区报告：`generate_community_report(...)`
   - 入口取 schema：`nano_graphrag/_op.py:635`
2. global 查询先按 `occurrence` 选社区
   - `nano_graphrag/_op.py:1027`
   - 排序：`nano_graphrag/_op.py:1035`

## 注意点

1. `community_schema` 只聚合，不聚类；聚类来源是前一步 `_leiden_clustering`。
2. 若节点缺 `clusters` 会被忽略，可能导致社区不完整。
3. `edges` 通过 `tuple(sorted(e))` 去方向，逻辑上按无向边处理。

