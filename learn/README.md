# Learn Notes

这个目录用于沉淀我们在 `nano-graphrag` 项目中的学习内容，目标是：

- 按主题快速查知识点（长期可维护）
- 按日期回放学习过程（可追溯）
- 统一记录格式（便于持续补充）

## 目录结构

```text
learn/
  README.md                  # 当前说明 + 索引
  topics/                    # 按主题沉淀
    asyncio-event-loop.md
    storage-graphml.md
  sessions/                  # 按日期记录学习过程
    2026-02-15.md
  templates/                 # 记录模板
    topic-template.md
    session-template.md
```

## 使用约定

- 新知识优先写到 `topics/`（避免重复）
- 当天讨论过程写到 `sessions/YYYY-MM-DD.md`
- 会话记录里只放结论链接，不重复贴大段解释
- 涉及源码时尽量写文件定位，如 `nano_graphrag/graphrag.py:228`

## 主题索引

- Asyncio 与事件循环：
  `learn/topics/asyncio-event-loop.md`
- 图存储与 GraphML：
  `learn/topics/storage-graphml.md`
- 实体抽取主流程：
  `learn/topics/extract-entities-flow.md`
- 节点合并细节：
  `learn/topics/merge-nodes-then-upsert.md`
- 边合并细节：
  `learn/topics/merge-edges-then-upsert.md`
- 默认实体抽取 Prompt：
  `learn/topics/default-entity-extraction-prompt.md`
- NetworkX 社区聚合：
  `learn/topics/networkx-community-schema.md`
- 社区描述打包：
  `learn/topics/pack-single-community-describe.md`
- 默认社区报告 Prompt：
  `learn/topics/default-community-report-prompt.md`
- Prompt 场景化定制：
  `learn/topics/prompt-customization-by-domain.md`
- local vs global 查询：
  `learn/topics/local-vs-global-query.md`
- global 查询流程：
  `learn/topics/global-query-flow.md`
- global map Prompt：
  `learn/topics/global-map-rag-points-prompt.md`
- global reduce Prompt：
  `learn/topics/global-reduce-rag-response-prompt.md`
- local 查询流程：
  `learn/topics/local-query-flow.md`

## 会话索引

- 2026-02-15：
  `learn/sessions/2026-02-15.md`
