# Prompt 按场景定制（实操清单）

## 结论

你的判断是对的：`nano-graphrag` 的默认 prompt 是通用模板，面对特定领域（法律、医疗、金融、科研、工业）通常要做领域化改造，尤其是实体类型、关系定义、评分标准。

## 优先定制项（按影响排序）

1. `PROMPTS["DEFAULT_ENTITY_TYPES"]`
   - 位置：`nano_graphrag/prompt.py:324`
   - 作用：直接影响实体抽取范围
   - 建议：改成领域词表，如法律可用 `["organization","person","law","case","court","location","event"]`

2. `PROMPTS["entity_extraction"]`
   - 位置：`nano_graphrag/prompt.py:194`
   - 作用：定义实体/关系提取规则与输出格式
   - 建议：明确关系强度评分口径、实体别名处理、排除项（例如“不要抽取通用停用概念”）

3. `PROMPTS["community_report"]`
   - 位置：`nano_graphrag/prompt.py:63`
   - 作用：决定社区报告写作框架与 `rating` 口径
   - 建议：给出清晰评分 rubric（例如 0-10 各分段含义），减少 `rating` 漂移

4. `PROMPTS["local_rag_response"]` / `PROMPTS["global_reduce_rag_response"]`
   - 位置：`nano_graphrag/prompt.py:330`、`nano_graphrag/prompt.py:426`
   - 作用：控制最终回答风格、保守性、证据引用方式

## 一个高频坑：改了 prompt 但结果不变

这是因为 `insert` 会先按 doc/chunk 的 md5 key 去重。若文档内容不变，可能直接判定“已存在”并跳过后续抽取流程：

- 文档去重：`nano_graphrag/graphrag.py:287`
- 无新增文档直接返回：`nano_graphrag/graphrag.py:289`
- chunk 去重后无新增也返回：`nano_graphrag/graphrag.py:304`、`nano_graphrag/graphrag.py:310`

所以只改 prompt 不清缓存，经常看不到效果。

## 正确做法（最实用）

当你改了抽取/报告 prompt 后，建议至少清掉以下缓存再重建：

1. `kv_store_full_docs.json`
2. `kv_store_text_chunks.json`
3. `graph_chunk_entity_relation.graphml`
4. `kv_store_community_reports.json`
5. `vdb_entities.json`（若开启 local）

示例里也有类似清理流程：`examples/no_openai_key_at_all.py:89`

## 建议的迭代方式

1. 固定一份小型验证语料（10-30 段）
2. 每次只改一项（先 `DEFAULT_ENTITY_TYPES`，再 `entity_extraction`）
3. 清缓存重建后对比：
   - 实体覆盖率
   - 关系质量（错连/漏连）
   - 社区报告可读性与可验证性
4. 最后再调回答 prompt（local/global）

