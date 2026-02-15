# `_pack_single_community_describe` 讲解

## 函数定位

- 定义：`nano_graphrag/_op.py:464`
- 调用：`generate_community_report` 内部，`nano_graphrag/_op.py:647`
- 作用：把一个社区整理成可喂给 LLM 的上下文文本（报告 + 实体表 + 关系表），并在 token 预算内尽量保留关键信息。

## 为什么需要它

`community_report` 的 LLM prompt 需要输入社区信息，但社区可能很大（节点、边很多）。
这个函数的核心是“预算内打包”：

1. 固定模板结构
2. 信息排序
3. 按 token 预算截断
4. 可选引用子社区报告减少冗余

## 输出格式

最终返回一个字符串，结构固定为：

```text
-----Reports-----
```csv
...
```
-----Entities-----
```csv
...
```
-----Relationships-----
```csv
...
```
```

模板位置：`nano_graphrag/_op.py:488`

## 主要步骤

### 1) 取社区原始数据

- 节点按名字排序：`nodes_in_order`
- 边按 `source+target` 排序：`edges_in_order`
- 批量拉取节点和边属性：`get_node/get_edge`

位置：`nano_graphrag/_op.py:476` 到 `nano_graphrag/_op.py:484`

### 2) 计算模板固定开销，得到可用预算

先计算“空模板”占用 token，然后从 `max_token_size` 里减掉：

- `base_template_tokens`
- `remaining_budget = max_token_size - base_template_tokens`

位置：`nano_graphrag/_op.py:500` 到 `nano_graphrag/_op.py:504`

### 3) 是否优先用子社区报告

触发条件：

- 社区很大（节点 > 100 或边 > 100）
- 当前社区有 `sub_communities`
- `already_reports` 不为空（说明更细层报告已生成）

或者被 `addon_params.force_to_use_sub_communities` 强制开启。

位置：`nano_graphrag/_op.py:511` 到 `nano_graphrag/_op.py:523`

若启用，会调用 `_pack_single_community_by_sub_communities(...)`：

- 先把子社区报告按 `occurrence` 排序
- 按 token 预算截断
- 生成 `Reports` 的 CSV 描述
- 同时返回“已覆盖的节点/边集合”，供后续去重

位置：`nano_graphrag/_op.py:417`

### 4) 组装实体/关系表并按重要性排序

- 实体字段：`id, entity, type, description, degree`
- 关系字段：`id, source, target, description, rank`
- `degree/rank` 都来自图度数（越大通常越重要）

位置：`nano_graphrag/_op.py:535` 到 `nano_graphrag/_op.py:558`

关键点：会过滤掉已被子社区报告覆盖的节点/边，减少重复。

### 5) 动态分配预算并截断

- 先扣掉表头 token 开销
- 剩余预算在实体表和关系表之间按“条目占比”分配
- 分别调用 `truncate_list_by_token_size(...)` 截断

位置：`nano_graphrag/_op.py:560` 到 `nano_graphrag/_op.py:586`

### 6) 回填模板返回

把 `reports/entities/relationships` 三段 CSV 填进模板后返回。

位置：`nano_graphrag/_op.py:594` 到 `nano_graphrag/_op.py:600`

## 你关心的“策略味道”

这个函数不是“绝对最优”，是工程上可控的启发式：

1. 大社区优先借用子社区报告（层次化摘要）
2. 节点/边按度数排序（重要性代理）
3. 预算按数据规模比例切分（简洁可解释）

## 两个实现细节提醒

1. 默认参数 `already_reports: dict = {}` 和 `global_config: dict = {}` 是可变默认值写法，不是最佳实践；当前调用路径基本不会触发问题，但风格上建议改成 `None` 再初始化。
2. 函数签名中的 `knwoledge_graph_inst` 是拼写遗留（应为 `knowledge`），功能无影响。

