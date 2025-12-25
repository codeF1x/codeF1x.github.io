+++
title = "第二课_Vercel 标准技术栈环境配置"
date = 2025-12-26T00:41:00+08:00
draft = false
categories = ["技术", "AI",'智能产品架构师']
+++

既然进入第一阶段实战，配置一套符合 **2025 年智能产品架构师标准** 的编码环境是所有工作的起点。我们将基于 **Vercel 标准技术栈**，构建一个支持流式传输、结构化输出和向量检索的全栈基础。

---

## 🛠️ 第一步：初始化 Next.js 15 工程底层

选择 **Next.js 15 (App Router)** 是因为其原生支持 **React 服务器组件 (RSC)** 和 **流式传输 (Streaming)**，这与 AI 的响应模式天然契合。

1. 创建项目：Bash

   在终端执行以下命令，确保开启 TypeScript 和 Tailwind CSS 支持。

   `npx create-next-app@latest my-ai-arch --typescript --tailwind --app`

2. 安装核心 AI 依赖：Bash

   安装 Vercel AI SDK 及其 DeepSeek 提供者（您选定的高性价比方案）。

   `npm install ai @ai-sdk/deepseek zod`

   - **TypeScript**: 确保在处理 AI 的概率性数据时保持类型安全。
   - **Zod**: 用于配合 `generateObject` 强制约束模型输出。

---

### 🗄️ 第二步：配置数据持久层 (Postgres + pgvector)

### **配置 Vercel Postgres**

为了实现阶段二的 RAG 功能，我们采用**“单体数据库”策略**，直接在 Postgres 中集成向量能力。

**现在开始：**  使用  **Vercel Postgres**。它能让你在 5 分钟内完成与 Next.js 15 的连接，并直接进入第一阶段的实战任务——**智能情感分析器**(第一阶段的项目名称)。

**后续进阶：**  当你进入阶段二，需要处理几十 GB 的非结构化数据清洗（ETL）时，再启动本地  **Docker**  进行大规模离线处理，以规避云端的存储成本。

| **维度**              | **Vercel Postgres (推荐方案)**                                 | **本地 Docker 部署**                                       |
| --------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| **集成难度**          | **极低**。与 Next.js 15 和 Vercel AI SDK 深度集成。            | **中等**。需要自行管理容器、端口映射及扩展。               |
| **AI 原生支持**       | **原生支持 pgvector**。一键开启向量功能。                      | **需手动配置**。必须选用特定的镜像（如 ankane/pgvector）。 |
| **Serverless 友好度** | **极高**。原生适配 Edge Runtime，冷启动优化好。                | **一般**。本地环境与线上 Edge 环境可能存在行为差异。       |
| **成本**              | 基础版免费，但在处理超大规模向量数据时会有积分限制。           | **完全免费**。适合处理海量数据抓取和测试。                 |
| **架构师价值**        | 学习  **“单体数据库策略”**，降低基础设施复杂度和数据同步成本。 |                                                            |

[\*\*https://vercel.com/](https://vercel.com/)\*\*    官网地址

在 Vercel Dashboard 中选择 "Storage" 选项卡，创建一个新的 Postgres 实例。选择 Neon 。

选择 Neon 理由：

**Vercel Postgres 的底层引擎**：Vercel 官方提供的 Postgres 服务实际上就是由 **Neon** 驱动的。选择它意味着您正在使用 Vercel AI 应用中最标准的数据库方案。

**原生支持 pgvector**：它是您实现阶段二 **RAG（检索增强生成）** 的核心组件。它允许您在同一个数据库中同时存储业务数据和高维向量数据。

**单体数据库策略 (Single Database Strategy)**：正如我们之前讨论的，这种选择能帮您避开维护独立向量数据库（如 Pinecone）的复杂度，极大降低基础设施成本 and 数据同步开销。

**自动扩缩容与 Edge 友好**：作为 Serverless 数据库，它能根据 AI 任务的负载自动扩写，且与 Next.js 15 的 Edge Runtime 配合极佳，能有效降低全球访问延迟。

**如何实现本地代码和云端数据的链接**

1. **在本地全局安装 Vercel CLI**

```bash
npm i -g vercel
```

1. **身份验证与项目绑定 🔑**

   1. 安装完成执行

   ```bash
   vercel login
   ```

   再访问 url (bash 中 visit 后面的 url)

   验证后 在您的项目根目录执行

```bash
vercel link
```

1. 它会问：Set up “~/.../my-ai-arch”? 选择 **Yes**。
2. 它会问：Which scope... 选择您的个人账号。
3. 它会问：Link to existing project? 选择 **Yes**。
4. 它会问：What’s the name of your existing project? 输入您在 Vercel 网页端创建的项目名称。

**去 Settings -> Environment Variables 页面添加 DEEPSEEK_API_KEY 这样拉取的 env 中就包含了 DEEPSEEK_API_KEY**

拉取 vercel 中的 .env.local

```bash
vercel env pull .env.local
```

### **安装 Drizzle ORM**

在“智能产品架构师”的技术栈选型中，选择 **Drizzle ORM** 是为了实现 **“单体数据库策略”**：即在同一个数据库中同时处理业务数据和 AI 向量数据。

Drizzle 被选中的核心原因在于它**轻量、无运行时开销**，对 **Serverless 环境（如 Vercel Edge Runtime）** 极其友好，且原生支持 **pgvector** 语法。

---

### 1. 安装核心依赖 🛠️

在项目根目录执行以下命令，安装运行时的 ORM 核心包及 Postgres 驱动：

`npm install drizzle-orm postgres`

接着安装开发环境所需的工具包（用于管理 Schema 和数据库同步）：

`npm install -D drizzle-kit`

---

### 2. 初始化数据库连接 (`db/index.ts`)

在本地项目中创建 `db/index.ts`，这是代码连接云端数据库的入口。它会读取您刚才通过 `vercel env pull` 拿到本地的 `DATABASE_URL`。

```jsx
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import * as schema from './schema';

// 禁用预处理语句以适配 Serverless/边缘环境
const client = postgres(process.env.DATABASE_URL!, { prepare: false });
export const db = drizzle(client, { schema });
```

---

### 3. 定义 AI 原生 Schema (`db/schema.ts`)

这是架构设计的核心。我们需要定义一个能存储 **AI 任务结果** 和 **向量数据** 的表结构。

```jsx
import {
  pgTable,
  serial,
  text,
  timestamp,
  jsonb,
  vector,
} from "drizzle-orm/pg-core";

// 任务分析表：用于存储 generateObject 生成的结构化数据
export const analysisTasks = pgTable("analysis_tasks", {
  id: serial("id").primaryKey(),
  content: text("content").notNull(),
  result: jsonb("result"), // 存储 AI 返回的 JSON 对象
  createdAt: timestamp("created_at").defaultNow(),
});

// 向量存储表：用于 RAG 阶段的语义搜索
export const embeddings = pgTable("embeddings", {
  id: serial("id").primaryKey(),
  content: text("content").notNull(),
  // 1536 维是 OpenAI/DeepSeek 常见的向量维度
  embedding: vector("embedding", { dimensions: 1536 }),
});
```

---

### 4. 同步到云端 (Push) 🚀

现在，利用 Drizzle Kit 将本地的设计图直接“推”送到云端的 Vercel Postgres：

`npx drizzle-kit push`

`npx drizzle-kit push` 执行完会报错，是因为 `drizzle-kit` 需要一个配置文件来充当“地图”，告诉它你的 **Schema（设计图）** 在哪里，以及如何连接到 **Vercel Postgres** 数据库

所以我们需要在项目跟目录下加一个 `drizzle.config.ts`

```jsx
import { defineConfig } from 'drizzle-kit';
import { config } from 'dotenv';

// 加载 .env.local 中的环境变量
config({ path: '.env.local' });

export default defineConfig({
  schema: './db/schema.ts',      // 告知 Schema 文件的路径
  out: './drizzle',              // 迁移文件的输出目录
  dialect: 'postgresql',         // 数据库方言
  dbCredentials: {
    url: process.env.DATABASE_URL!, // 从环境变量读取云端连接字符串
  },
  strict: true,
  verbose: true,
});
```

引入 dotenv 依赖

```bash
npm install dotenv
```

然后再执行 `npx drizzle-kit push`

**提示**：执行此操作前，请确保您已经在 Vercel 控制台的 SQL 面板执行过 `CREATE EXTENSION IF NOT EXISTS vector;`，否则推送包含 `vector` 类型的数据表时会报错。

`CREATE EXTENSION IF NOT EXISTS vector;` 执行位置在 Storage-你的 neon 实例-Open In Neon-SQL Editor 中。

执行结果：

```bash
[✓] Pulling schema from database...

 Warning  You are about to execute current statements:

CREATE TABLE "analysis_tasks" (
        "id" serial PRIMARY KEY NOT NULL,
        "content" text NOT NULL,
        "result" jsonb,
        "created_at" timestamp DEFAULT now()
);

CREATE TABLE "embeddings" (
        "id" serial PRIMARY KEY NOT NULL,
        "content" text NOT NULL,
        "embedding" vector(1536)
);

❯ No, abort
  Yes, I want to execute all statements


 选择 yes .
```

当你点击 **Yes** 后，你的 **Next.js 15** 项目与 **Vercel Postgres** 之间已经彻底打通了。你目前构建的是一种**“推理系统（Reasoning Systems）”**的物理架构：

- **传统层**：处理文本、时间等确定性数据。
- **智能层**：处理 JSON 结构化指令。
- **语义层**：处理高维向量空间数据

---

### 📝 检查点 (Checkpoint)

完成以上步骤后，您的本地环境已具备**读写云端 AI 数据**的能力：

- **Drizzle ORM** 已配置为 Serverless 模式。
- **Schema** 已兼容结构化 JSON 和高维向量。
- **管道** 已通过 `DATABASE_URL` 彻底打通。

**数据库底座已打好，现在我们正式进入“实战任务 1”：编写第一个 `app/api/analyze/route.ts` 接口，利用 `generateObject` 把 AI 分析结果存入数据库，好吗？** 💡

---

## 🔑 第三步：环境变量与安全配置

在项目根目录检查 `.env.local`，并配置关键服务的 API 密钥。

- **DEEPSEEK_API_KEY**: 用于调用 DeepSeek V3/R1 模型。
- **DATABASE_URL**: 您的 Postgres 连接字符串。
- **NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY**: 若需用户鉴权，推荐使用 Clerk 方案。

---

### 🎨 第四步：构建 AI 原生 UI 基座

为了承接 **生成式 UI (GenUI)**，我们需要一套原子化的组件库。

1. 集成 shadcn/ui：Bash

   这是目前构建 AI 应用最快的组件组合。

   `npx shadcn-ui@latest init`

2. 准备“无头”组件：

   设计一组高度自包含的组件（如 Card, Skeleton），它们将作为 AI 动态填充数据的容器。

---

### 📝 环境核对清单 (Checklist)

| **模块**      | **状态检测**                 | **架构目标**                     |
| ------------- | ---------------------------- | -------------------------------- |
| **Runtime**   | Node.js 20+ & Next.js 15     | 支持 Edge Runtime 降低延迟。     |
| **AI Stream** | `ai/rsc` 或 `useChat` 已就绪 | 实现逐字渲染（Token-by-Token）。 |
| **Safety**    | Zod Schema 与类型定义同步    | 将概率性输出约束在可控系统中。   |
| **Storage**   | `pgvector` 扩展已激活        | 为后续语义搜索打下基础。         |

环境配置完成后，您想先尝试编写第一个“智能情感分析器”的后端 API 路由，还是先配置 Drizzle ORM 来连接您的向量数据库？

为了实现阶段二的 RAG 功能，我们采用**“单体数据库”策略**，直接在 Postgres 中集成向量能力。

**现在开始：**  使用  **Vercel Postgres**。它能让你在 5 分钟内完成与 Next.js 15 的连接，并直接进入第一阶段的实战任务——**智能情感分析器**(第一阶段的项目名称)。

**后续进阶：**  当你进入阶段二，需要处理几十 GB 的非结构化数据清洗（ETL）时，再启动本地  **Docker**  进行大规模离线处理，以规避云端的存储成本。

| **维度**              | **Vercel Postgres (推荐方案)**                                 | **本地 Docker 部署**                                       |
| --------------------- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| **集成难度**          | **极低**。与 Next.js 15 和 Vercel AI SDK 深度集成。            | **中等**。需要自行管理容器、端口映射及扩展。               |
| **AI 原生支持**       | **原生支持 pgvector**。一键开启向量功能。                      | **需手动配置**。必须选用特定的镜像（如 ankane/pgvector）。 |
| **Serverless 友好度** | **极高**。原生适配 Edge Runtime，冷启动优化好。                | **一般**。本地环境与线上 Edge 环境可能存在行为差异。       |
| **成本**              | 基础版免费，但在处理超大规模向量数据时会有积分限制。           | **完全免费**。适合处理海量数据抓取和测试。                 |
| **架构师价值**        | 学习  **“单体数据库策略”**，降低基础设施复杂度和数据同步成本。 |                                                            |
