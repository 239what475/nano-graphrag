# `local_query` vs `global_query` 对照

## 入口

- 分流位置：`nano_graphrag/graphrag.py:236`
- `mode="local"` -> `local_query(...)`
- `mode="global"` -> `global_query(...)`

## 一句话区别

- `local_query`：先找“与问题最相关的实体”，围绕这些实体拉关系/文本/社区，再回答。
- `global_query`：先看“全局社区报告”，做 map-reduce 汇总后再回答。

## `local_query` 流程（实体中心）

1. 用 `entities_vdb.query` 找 top-k 相关实体：
   `nano_graphrag/_op.py:853`
2. 回图中取这些实体节点，并计算节点度：
   `nano_graphrag/_op.py:856`, `nano_graphrag/_op.py:859`
3. 基于实体再取：
   - 相关社区：`nano_graphrag/_op.py:865`
   - 相关文本块：`nano_graphrag/_op.py:868`
   - 相关关系边：`nano_graphrag/_op.py:871`
4. 拼成四段上下文（Reports / Entities / Relationships / Sources）：
   `nano_graphrag/_op.py:915`
5. 套 `PROMPTS["local_rag_response"]` 生成最终答案：
   `nano_graphrag/_op.py:959`

## `global_query` 流程（社区中心）

1. 从图上取社区 schema，并按 level 过滤：
   `nano_graphrag/_op.py:1027`
2. 按 `occurrence` 排序，截断候选社区数：
   `nano_graphrag/_op.py:1035`
3. 读取社区报告，并按 `rating` 阈值过滤：
   `nano_graphrag/_op.py:1043`, `nano_graphrag/_op.py:1050`
4. 把社区分组后走 map 阶段，产出 points（description + score）：
   `nano_graphrag/_op.py:970`
5. 聚合所有 points，按 score 排序截断，形成 analyst reports：
   `nano_graphrag/_op.py:1062` 到 `nano_graphrag/_op.py:1094`
6. 套 `PROMPTS["global_reduce_rag_response"]` 生成最终答案：
   `nano_graphrag/_op.py:1097`

## 参数侧重点

- `local` 重点参数：
  - `top_k`
  - `local_max_token_for_text_unit`
  - `local_max_token_for_local_context`
  - `local_max_token_for_community_report`
  - `local_community_single_one`

- `global` 重点参数：
  - `global_min_community_rating`
  - `global_max_consider_community`
  - `global_max_token_for_community_report`

定义：`nano_graphrag/base.py:9`

## 选型建议（经验）

1. 问题针对具体人物/实体关系：优先 `local`
2. 问题偏主题总结、全局趋势、总体观点：优先 `global`
3. 对召回稳定性要求高时，可先 `global` 看大局，再 `local` 针对关键实体追问

