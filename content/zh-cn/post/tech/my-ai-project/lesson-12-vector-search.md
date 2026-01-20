+++
title = "第十二课_向量检索服务"
date = 2026-01-21T01:12:00+08:00
draft = false
categories = ["技术", "AI", "智能产品架构师"]
+++

## 🎯 核心问题

**问题**: 如何从海量的文档切片中，精准地找到与用户问题"语义相关"的内容，而不仅仅是关键词匹配？

**重要性**:

- **RAG 的灵魂**: 检索质量直接决定了最终回答的质量。检索不到相关信息，LLM 就只能"幻觉"。
- **语义理解**: 用户问"怎么退货"，文档里可能写的是"售后流程"，关键词匹配失效，必须靠向量语义检索。
- **性能挑战**: 需要在毫秒级内从百万级向量中找到 Top-K。

## 🧠 核心概念图解

```ascii
User Query: "RAG 是什么?"
       |
       v
+-------------------+
| Embedding Model   |  (Text -> Vector)
+-------------------+
       |
       v
[0.1, -0.5, ... 0.9]  (Query Vector)
       |
       v
+-------------------------------------------------------+
| PostgreSQL (pgvector)                                 |
|                                                       |
| Row 1: [0.1, -0.4, ... 0.8] -> Distance: 0.1 (Close!) |
| Row 2: [0.9,  0.1, ... 0.1] -> Distance: 1.5 (Far)    |
| ...                                                   |
+-------------------------------------------------------+
       |
       v
Top K Results (Context)
```

## 📐 架构设计

1.  **Query Embedding**: 复用 `lib/embeddings/generate.ts`，确保查询向量与文档向量使用**同一个模型**生成（维度必须一致）。
2.  **Vector Search**: 使用 `pgvector` 的 `<=>` (Cosine Distance) 操作符。
3.  **Similarity Calculation**:
    - `pgvector` 返回的是**距离 (Distance)** (0=完全一样, 2=完全相反)。
    - 我们需要的是**相似度 (Similarity)**。
    - 公式: `Similarity = 1 - CosineDistance`。

## 🔧 核心伪代码

```typescript
function search(query):
  // 1. Generate Vector
  queryVector = embed(query)

  // 2. SQL Query with Drizzle
  // 1 - (embedding <=> queryVector) as similarity
  results = db.select({
    content: t.content,
    similarity: sql`1 - (${cosineDistance(t.embedding, queryVector)})`
  })
  .from(embeddings)
  .where(gt(similarity, threshold)) // Filter low relevance
  .orderBy(desc(similarity))        // Best matches first
  .limit(5)

  return results
```

## 🛠 关键技术点

1.  **Cosine Distance vs Similarity**:
    - 很多向量库直接返回 Score，但 `pgvector` 底层操作符 `<=>` 计算的是余弦距离。
    - 务必进行 `1 - distance` 转换，否则排序会反（距离越小越相似）。

2.  **Drizzle `sql` Operator**:
    - Drizzle ORM 对 `pgvector` 的支持需要使用 `sql` 模板字符串来构建自定义查询表达式。

3.  **Threshold Filtering**:
    - 设置合理的阈值（如 0.5 或 0.7）可以过滤掉完全不相关的噪音，防止 LLM 被错误信息误导。

## ⚠️ 踩坑记录

### 1. 数据库连接失败 (`ECONNREFUSED`)

- **现象**: 测试脚本报错 `AggregateError [ECONNREFUSED]`。
- **原因**: 本地 PostgreSQL 服务未启动，或 Docker 容器挂了。
- **解决**: 编写 `scripts/check-db-connection.ts` 快速诊断，确保 DB 服务存活 (`net start postgresql` 或 `docker start ...`)。

### 2. 维度不匹配

- **现象**: 报错 `different vector dimensions`。
- **原因**: 存入时用了 `text-embedding-3-small` (1536维)，查询时用了 `embedding-3` (512维)。
- **解决**: 必须强制保证 **存** 和 **取** 使用完全相同的模型配置。

## 📊 知识网络

- **前置知识**: Embedding 原理, SQL 基础。
- **关联课程**: Lesson 09 (Embedding 服务), Lesson 10 (向量存储)。
- **后续应用**: 混合检索 (Hybrid Search = Vector + Keyword)。

## 🎯 费曼检验

**"向量检索就像是在图书馆里找书，但不是按书名找（关键词），而是按'书的内容讲什么'来找。我们把每本书和你的问题都变成一个坐标点，然后算出哪本书的坐标离你的问题最近。"**

## 📝 实战练习

修改 `lib/rag/vector-search.ts`，增加一个 `filter` 参数，支持按 `documentId` 进行过滤搜索（即：只在特定文档内搜索）。

```typescript
// 提示：
export async function searchSimilarChunks(
  query: string,
  options: SearchOptions & { documentId?: string } // Add this
) {
  // ...
  let queryBuilder = db.select(...).from(embeddings);

  const conditions = [gt(similarity, threshold)];
  if (options.documentId) {
    conditions.push(eq(embeddings.documentId, options.documentId));
  }

  // Use and(...conditions) in where clause
}
```

## ✅ 检查清单

- [ ] 确认 `pgvector` 扩展已在 DB 中开启。
- [ ] 确认查询向量与存储向量的模型/维度一致。
- [ ] 验证了 `1 - distance` 的相似度计算逻辑。
- [ ] 测试了数据库连接中断时的错误处理。
