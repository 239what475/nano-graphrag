# Asyncio 与 Event Loop（项目语境）

## 一句话结论

`nano-graphrag` 的 `insert/query` 是对 `ainsert/aquery` 的同步封装，内部通过 event loop 执行协程。

## 关键调用链

1. 同步入口：
   `insert()` / `query()`
   位置：`nano_graphrag/graphrag.py:228`, `nano_graphrag/graphrag.py:232`
2. 获取 loop：
   `always_get_an_event_loop()`
   位置：`nano_graphrag/_utils.py`（函数定义）
3. 同步驱动异步：
   `loop.run_until_complete(self.ainsert(...))`
   位置：`nano_graphrag/graphrag.py:230`
4. 真正异步逻辑：
   `async def ainsert(...)` / `async def aquery(...)`
   位置：`nano_graphrag/graphrag.py:277`, `nano_graphrag/graphrag.py:236`

## 概念对齐

- `await`：在 async 函数内部等待协程
- `run_until_complete(...)`：在 sync 函数中“阻塞等待协程完成”
- 它更接近 Rust 的 `block_on`，不是线程 `join`

## 关于线程模型

- 默认是单线程 event loop 上的协程并发（`asyncio.gather`）
- 不是默认多线程并行计算
- IO 密集任务能并发提速；纯 CPU 密集任务不会自动多核加速

## `__post_init__` 相关

- `GraphRAG` 是 `@dataclass`，实例化时会自动调用 `__post_init__`
  位置：`nano_graphrag/graphrag.py:52`, `nano_graphrag/graphrag.py:140`
- 若手写 `__init__`，需自行决定是否调用 `__post_init__`

## `_insert_start` 相关

`_insert_start` 会调用图存储的 `index_start_callback()`：

- 调用点：`nano_graphrag/graphrag.py:350`
- 默认 `NetworkXStorage` 未重写该方法，因此是 no-op（基类空实现）
  位置：`nano_graphrag/base.py:64`
- 换成 `Neo4jStorage` 时有真实初始化逻辑
  位置：`nano_graphrag/_storage/gdb_neo4j.py:66`

