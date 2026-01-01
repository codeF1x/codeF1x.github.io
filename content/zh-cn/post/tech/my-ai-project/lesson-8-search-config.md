# 第八课_本地向量搜索优化

## 1. 概述

本项目对本地向量搜索功能进行了深度优化。核心目标是提升在大量文档分块（Chunks）场景下的检索性能，减少主线程与浏览器 IndexedDB 引擎之间的通信开销，并确保内存使用的稳定性。

## 2. 核心改进

### 2.1 数据库架构设计

在 IndexedDB 中，我们设计了一个名为 `vectors` 的对象存储空间（Object Store），其结构如下：

- **主键 (Primary Key)**: `id` (字符串，通常为分块的唯一标识符)。
- **数据项 (Data Item)**: 包含 `docId` (所属文档 ID)、`text` (原始文本)、`vector` (Float32Array 向量数据) 等。
- **索引设计**:
    - **by_doc (复合索引)**: 键路径为 `["docId", "id"]`。
    - **设计初衷**:
        1. **精确过滤**: 能够快速定位属于特定 `docId` 的所有向量。
        2. **稳定分页**: 利用 `id` 作为复合键的第二部分，确保在分页检索时顺序是确定的且不会遗漏或重复记录。
        3. **性能优化**: 复合索引允许 IndexedDB 引擎在索引树中直接完成范围扫描，而无需回表查询主键。

### 2.2 数据库架构升级 (`lib/db.ts`)

为了支持高效的分页和过滤，我们将数据库版本提升至 `2`，并对索引进行了重构：

- **复合索引**：将原有的单一 `docId` 索引升级为复合索引 `["docId", "id"]`。
- **升级逻辑**：在 `onupgradeneeded` 中实现了平滑升级，自动删除旧索引并创建支持分页的复合索引。

**核心伪代码：**

```tsx
// 数据库升级逻辑
onupgradeneeded = (event) => {
  const store = request.transaction.objectStore("vectors");
  if (oldVersion < 2) {
    // 删除旧的单列索引
    if (store.indexNames.contains("by_doc")) store.deleteIndex("by_doc");
    // 创建复合索引 [文档ID + 记录ID]，用于精确匹配和有序分页
    store.createIndex("by_doc", ["docId", "id"], { unique: false });
  }
};
```

### 2.3 搜索算法重构 (`lib/search.ts`)

搜索逻辑从“逐条遍历”演进为“批量检索”：

- **方案演进**：
    - **v0.1 (Cursor)**: 使用 `openCursor` 逐条读取。缺点是产生“乒乓效应”，主线程与 IndexedDB 频繁通信，I/O 开销大。
    - **v0.2 (Batch getAll)**: 采用 `store.getAll() + IDBKeyRange + 分页`。
- **实现细节**：
    - 使用 `IDBKeyRange.bound` 配合复合索引实现“基于查找（Seek-based）”的分页。
    - 设定 `BATCH_SIZE = 100`，每批次获取 100 条数据进行相似度计算。
    - 显著减少了 IPC（进程间通信）次数，提升了整体吞吐量。

**核心伪代码：**

```tsx
// 批量分页检索逻辑
let lastId = "";
while (hasMore) {
  // 构建范围：[当前文档ID, 上次最后一条ID] -> [当前文档ID, 极大值]
  const range = lastId
    ? IDBKeyRange.bound([docId, lastId], [docId, []], true) // true 表示排除起始点
    : IDBKeyRange.bound([docId], [docId, []]);

  // 一次性拉取 100 条数据，减少“乒乓效应”
  const batch = await index.getAll(range, BATCH_SIZE);

  // 内存中批量计算相似度
  for (const item of batch) {
    const score = calculateSimilarity(query, item.vector);
    results.push({ ...item, score });
  }

  // 更新游标位置，准备下一轮拉取
  lastId = batch[batch.length - 1]?.id;
  if (batch.length < BATCH_SIZE) hasMore = false;
}
```

### 2.4 向量数学库 (`lib/vector-math.ts`)

- 提取了核心的 `calculateCosineSimilarity` 函数，专门用于计算已归一化向量的余弦相似度，保证了计算逻辑的纯粹与高效。

## 3. 云端同步与持久化 (`hooks/use-sync.ts` & `db/schema.ts`)

- **云端同步**：新增 `use-sync.ts` 钩子，支持将本地计算的向量序列化并同步至远程 Postgres 数据库（通过 Drizzle ORM）。
- **服务端架构**：在 `db/schema.ts` 中定义了 `embeddings` 表，支持 1536 维向量存储，为 RAG（检索增强生成）阶段的语义搜索打下基础。

## 4. 性能考量与未来展望

### 4.1 为什么选择 `getAll` 而不是 `openCursor`？

虽然 `openCursor` 在内存占用上理论上更优，但在现代浏览器中，主线程与 IndexedDB 存储进程之间的通信延迟往往是瓶颈。批量获取数据（Batching）能极大缓解这一问题。

### 4.2 未来优化方向 (v0.3)

目前的 `getAll` 方案在处理“降序排列”或“获取最新记录”时仍有局限（IndexedDB 的 `getAll` 仅支持升序）。

- **前沿情报**：最新的 IndexedDB 规范引入了 `getAllRecords()` API，未来可支持：
    1. **原生降序**：批量获取倒序数据。
    2. **键值合一**：同时获取 Key 和 Value，进一步减少调用次数。

## 5. 总结

通过本次优化，系统在保持本地化处理优势的同时，具备了处理大规模向量数据的能力，并为后续的云端协同和 RAG 流程提供了坚实的技术支撑。