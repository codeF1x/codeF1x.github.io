+++
title = "第十课_向量存储与 PostgreSQL pgvector 集成"
date = 2026-01-21T10:00:00+08:00
draft = false
categories = ["技术", "AI", "智能产品架构师"]
+++

## 🎯 核心问题

**本课解决什么问题？**

在 RAG (Retrieval-Augmented Generation) 系统中,我们需要:

1. 将文档的向量表示持久化存储到数据库
2. 支持高效的向量相似度搜索
3. 管理大规模向量数据（文档 ID、分块索引、元数据等）

**为什么重要？**

- 向量是 RAG 系统的核心,存储不当会导致检索失败
- PostgreSQL + pgvector 是开源、成熟、高性能的向量数据库方案
- 正确的 schema 设计直接影响查询性能和系统可扩展性

---

## 🧠 核心概念图解

```
┌─────────────────────────────────────────────────────────────┐
│                    RAG 向量存储流程                          │
└─────────────────────────────────────────────────────────────┘

文档输入
   │
   ├─► 1. 分块 (Chunking)
   │      ├─ chunk 0: "人工智能是..."
   │      ├─ chunk 1: "深度学习..."
   │      └─ chunk 2: "神经网络..."
   │
   ├─► 2. 向量化 (Embedding)
   │      ├─ [0.1, 0.2, ..., 0.5]  (512维)
   │      ├─ [0.3, 0.1, ..., 0.7]
   │      └─ [0.2, 0.4, ..., 0.3]
   │
   └─► 3. 批量存储 (Batch Store)
          │
          ▼
   ┌──────────────────────────────────────┐
   │   PostgreSQL + pgvector              │
   ├──────────────────────────────────────┤
   │ embeddings 表                        │
   │ ├─ id: 1                             │
   │ ├─ document_id: "doc-123"            │
   │ ├─ chunk_index: 0                    │
   │ ├─ content: "人工智能是..."          │
   │ ├─ embedding: vector(512)            │
   │ ├─ provider: "zhipu"                 │
   │ ├─ model: "embedding-3"              │
   │ └─ dimensions: 512                   │
   └──────────────────────────────────────┘
          │
          ├─► 索引: document_id_idx (快速查找文档)
          └─► 索引: provider_idx (按提供商筛选)

查询时:
   用户问题 → 向量化 → 相似度搜索 → 返回最相关的 chunks
```

---

## 📐 架构设计

### 三层架构

```
┌─────────────────────────────────────────┐
│         API 层 (route.ts)               │
│  - POST /api/embeddings/store           │
│  - DELETE /api/embeddings/store         │
│  - 输入验证 (Zod)                       │
│  - 错误处理                             │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│      服务层 (store.ts)                  │
│  - storeVectors()                       │
│  - deleteVectorsByDocumentId()          │
│  - 维度验证                             │
│  - 批量插入优化                         │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│      数据层 (schema.ts + Drizzle)       │
│  - embeddings 表定义                    │
│  - 索引配置                             │
│  - pgvector 类型                        │
└─────────────────────────────────────────┘
```

**为什么这样设计？**

1. **API 层**: 统一入口,验证输入,标准化响应
2. **服务层**: 业务逻辑,可复用,易测试
3. **数据层**: ORM 抽象,类型安全,迁移管理

---

## 🔧 核心伪代码

### 批量存储向量

```typescript
function storeVectors(records: VectorRecord[]) {
  // 1. 验证维度一致性
  firstDim = records[0].embedding.length
  for each record in records:
    if record.embedding.length != firstDim:
      throw Error("维度不一致")

  // 2. 批量插入数据库
  insertedIds = db.insert(embeddings).values(
    records.map(r => ({
      documentId: r.documentId,
      chunkIndex: r.chunkIndex,
      content: r.content,
      embedding: r.embedding,  // 直接传数组!
      provider: r.provider,
      model: r.model,
      dimensions: r.dimensions,
    }))
  )

  // 3. 返回结果
  return { success: true, count: insertedIds.length, ids: insertedIds }
}
```

### 灵活 Schema 设计

```sql
CREATE TABLE embeddings (
  id SERIAL PRIMARY KEY,
  document_id TEXT NOT NULL,      -- 文档 ID
  chunk_index INTEGER NOT NULL,   -- 分块索引
  content TEXT NOT NULL,           -- 原文
  embedding VECTOR(512),           -- 向量
  provider TEXT NOT NULL,          -- 提供商 (zhipu/openai)
  model TEXT NOT NULL,             -- 模型名称
  dimensions INTEGER NOT NULL,     -- 实际维度
  metadata JSONB,                  -- 自定义元数据
  created_at TIMESTAMP DEFAULT NOW()
);

-- 索引优化
CREATE INDEX ON embeddings (document_id);
CREATE INDEX ON embeddings (provider);
```

---

## 🛠 关键技术点

### 1. 为什么选择 PostgreSQL + pgvector？

| 特性       | PostgreSQL + pgvector | 专用向量数据库 (Pinecone/Weaviate) |
| ---------- | --------------------- | ---------------------------------- |
| 成本       | 免费开源              | 按量付费                           |
| 部署       | 自托管/云服务         | 仅云服务                           |
| 学习曲线   | 熟悉 SQL              | 新 API                             |
| 数据一致性 | ACID 事务             | 最终一致性                         |
| 混合查询   | SQL + 向量            | 仅向量                             |

**结论**: 对于中小规模应用,PostgreSQL + pgvector 是最佳选择。

### 2. pgvector 向量格式

```typescript
// ❌ 错误 - 不要 JSON.stringify
embedding: JSON.stringify([0.1, 0.2, 0.3]);

// ✅ 正确 - 直接传数组
embedding: [0.1, 0.2, 0.3];
```

**原因**: Drizzle ORM 会自动将 JavaScript 数组转换为 pgvector 格式。

### 3. 批量插入优化

```typescript
// ❌ 低效 - 逐条插入
for (const record of records) {
  await db.insert(embeddings).values(record); // N 次数据库往返
}

// ✅ 高效 - 批量插入
await db.insert(embeddings).values(records); // 1 次数据库往返
```

**性能差异**: 100 条记录,批量插入快 50-100 倍!

### 4. 灵活的维度设计

```typescript
// 支持多种 embedding 提供商
embeddings: {
  provider: "zhipu" | "openai" | "google",
  model: "embedding-3" | "text-embedding-3-small",
  dimensions: 512 | 1536 | 768,  // 记录实际维度
}
```

**好处**:

- 可以混用不同提供商
- 方便 A/B 测试不同模型
- 未来切换提供商无需迁移数据

---

## ⚠️ 踩坑记录

### 坑1: OpenAI SDK 模块加载错误

**错误信息**: `Missing credentials. Please pass an apiKey`

**原因**: 在模块顶层直接创建客户端,环境变量未加载就会报错

```typescript
// ❌ 错误 - 模块加载时立即执行
const client = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY, // 可能为 undefined
});
```

**解决方案**: 懒加载

```typescript
// ✅ 正确 - 使用时才创建
let client: OpenAI | null = null;

function getClient() {
  if (!client) {
    if (!process.env.OPENAI_API_KEY) {
      throw new Error("Missing API key");
    }
    client = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
  }
  return client;
}
```

### 坑2: pgvector 格式错误

**错误信息**: `Vector contents must start with "["`

**原因**: 使用了 `JSON.stringify()` 导致格式错误

**解决**: 直接传数组,让 Drizzle ORM 处理转换

### 坑3: 僵尸进程导致测试失败

**现象**: 修改代码后测试仍然失败,错误不变

**原因**: 旧的 Next.js 服务器仍在运行

**解决**:

```powershell
# 杀掉所有 node 进程
taskkill /F /IM node.exe

# 或者只杀占用 3000 端口的进程
Stop-Process -Id (Get-NetTCPConnection -LocalPort 3000).OwningProcess -Force
```

### 坑4: DeepSeek 不提供 Embedding API

**发现**: DeepSeek 只有 chat 和 reasoner 模型,**没有 embedding**

**解决**: 改用智谱 AI 的 `embedding-3` 模型

---

## 📊 知识网络

```
本课 (向量存储)
    ↑ 依赖
    │
    ├─ 第9课: Embedding 生成服务
    │   └─ 提供向量数据
    │
    ├─ 第8课: 文档分块
    │   └─ 提供文本内容
    │
    └─ PostgreSQL 基础
        └─ 数据库操作

    ↓ 支撑
    │
    ├─ 第11课: 向量相似度搜索 (下一课)
    │   └─ 使用存储的向量进行检索
    │
    └─ 第12课: RAG 完整流水线
        └─ 组合所有模块
```

---

## 🎯 费曼检验

**用一句话解释核心概念:**

> 向量存储就像给每个文档片段贴上"指纹"(向量),然后存到数据库里,这样以后就能通过"指纹相似度"快速找到相关内容。

**类比**:

- 向量 = 指纹 (唯一标识)
- pgvector = 指纹数据库
- 相似度搜索 = 指纹匹配

---

## 📝 实战练习

### 练习1: 存储测试数据

```bash
# 1. 启动开发服务器
npm run dev

# 2. 运行快速测试
npx tsx scripts/test-vector-quick.ts

# 预期输出:
# ✓ 成功存储 3 个向量
# ✓ IDs: [1, 2, 3]
# ✓ 成功删除 3 个向量
```

### 练习2: 查询数据库

```sql
-- 查看存储的向量
SELECT
  id,
  document_id,
  chunk_index,
  provider,
  model,
  dimensions,
  length(content) as content_length
FROM embeddings
ORDER BY created_at DESC
LIMIT 10;

-- 统计各提供商的向量数量
SELECT
  provider,
  model,
  COUNT(*) as count,
  AVG(dimensions) as avg_dimensions
FROM embeddings
GROUP BY provider, model;
```

### 练习3: 完整流程

```typescript
// 1. 分块
const chunks = chunkText(document, { chunkSize: 500 });

// 2. 向量化
const result = await generateEmbeddingsBatch(
  chunks.map((c) => c.content),
  { provider: "zhipu" },
);

// 3. 存储
const storeResult = await storeVectors(
  result.embeddings.map((emb, i) => ({
    documentId: "my-doc",
    chunkIndex: i,
    content: chunks[i].content,
    embedding: emb,
    provider: "zhipu",
    model: "embedding-3",
    dimensions: 512,
  })),
);

console.log(`存储了 ${storeResult.count} 个向量`);
```

---

## ✅ 检查清单

完成本课后,你应该能够:

- [ ] 理解向量存储在 RAG 系统中的作用
- [ ] 设计灵活的 embeddings 表 schema
- [ ] 使用 Drizzle ORM 进行批量插入
- [ ] 正确处理 pgvector 向量格式
- [ ] 创建索引优化查询性能
- [ ] 使用 drizzle-kit 管理数据库迁移
- [ ] 实现完整的存储 API (POST/DELETE/GET)
- [ ] 编写测试验证功能正确性
- [ ] 避免常见的坑 (懒加载、格式、僵尸进程)

---

## 📚 延伸阅读

- [pgvector 官方文档](https://github.com/pgvector/pgvector)
- [Drizzle ORM 文档](https://orm.drizzle.team/)
- [PostgreSQL 向量索引优化](https://github.com/pgvector/pgvector#indexing)
- [向量数据库对比](https://benchmark.vectorview.ai/)

---

## 🎓 下一步

完成向量存储后,下一课将学习:

**第11课: 向量相似度搜索**

- 使用 pgvector 的相似度函数
- 实现 k-NN 搜索
- 优化搜索性能
- 混合检索 (向量 + 关键词)

准备好了吗? 🚀
