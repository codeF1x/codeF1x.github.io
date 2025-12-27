+++
title = "第五课_第一阶段的补充"
date = 2025-12-27T00:43:00+08:00
draft = false
categories = ["技术", "AI",'智能产品架构师']
+++

今日开发实战文档。

---

# 📝 AI 全栈开发实战文档：流式专家诊断系统

## 1. 今日工作概述 (Work Summary)

今天我们的核心目标是优化用户体验与架构健壮性，完成了以下三个维度的升级：

- **交互升级**：将阻塞式的 `generateObject` 升级为流式的 `streamObject`，实现了 JSON 数据的“打字机”式生长。
- **逻辑增强**：引入 **Few-Shot Prompting（少样本提示）** 和 **CoT（思维链）**，显著提升了 AI 诊断的准确性与可信度。
- **UI 工业化**：集成 **shadcn/ui** 规范，构建了基于“无头组件”思想的 AI 原生 UI 基座。
- 在 Vercel 上部署 demo：添加流量限制功能

---

## 2. 核心代码实现 (Key Code)

### 2.1 后端：流式与逻辑补全 (`app/api/analyze/route.ts`)

使用了 `streamObject` 配合 `messages` 数组实现 Few-Shot。

```tsx
import { streamObject } from "ai";
import { deepseek } from "@ai-sdk/deepseek";
import { analysisSchema } from "./schema";

export async function POST(req: Request) {
  const { text } = await req.json();

  const result = await streamObject({
    model: deepseek("deepseek-chat"),
    schema: analysisSchema, // Zod 定义的结构化 Schema
    output: "object",
    messages: [
      { role: "system", content: "你是一位资深硬件维修专家..." },
      // Few-Shot 示例：校准 AI 判定标准
      { role: "user", content: "型号 U2723QE" },
      {
        role: "assistant",
        content: JSON.stringify({
          sentiment: "neutral",
          score: 0.9,
          reasoning: "仅包含型号...",
          summary: "提供设备型号",
        }),
      },
      // 真实用户输入
      { role: "user", content: text },
    ] as const,
    // 流结束后的异步持久化
    onFinish: async ({ object }) => {
      if (object) {
        await db
          .insert(analysisTasks)
          .values({ content: text, result: object });
      }
    },
  });

  return result.toTextStreamResponse();
}
```

### 2.2 前端：流式 JSON 渲染 (`app/page.tsx`)

利用 `ai/react` 库中的 `useObject` Hook 实时处理不完整的 JSON 块。

```tsx
"use client";

import { experimental_useObject as useObject } from "@ai-sdk/react";
import { useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

export default function AnalysisPage() {
  const [text, setText] = useState("");

  // v6.x 标准用法：实时同步后端生长的对象
  const { object, submit, isLoading } = useObject({
    api: "/api/analyze",
    schema: analysisSchema,
  });

  return (
    <div className="p-8">
      <textarea value={text} onChange={(e) => setText(e.target.value)} />
      <button onClick={() => submit({ text })} disabled={isLoading}>
        开始诊断
      </button>

      {/* 魔法时刻：partialObject 的字段会逐个出现 */}
      {object && (
        <Card>
          <CardHeader>
            <CardTitle>
              置信度: {((object.score ?? 0) * 100).toFixed(0)}%
            </CardTitle>
          </CardHeader>
          <CardContent>
            <p className="animate-pulse">{object.summary}</p>
            {object.reasoning && (
              <div className="mt-4 italic">{object.reasoning}</div>
            )}
          </CardContent>
        </Card>
      )}
    </div>
  );
}
```

---

## 3. 核心名词解释 (Glossary)

| **名词**                   | **解释**                                                                                                                                                               |
| -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Few-Shot Prompting**     | **少样本提示**。在提示词中加入几个“输入-输出”的示例，像教学生一样告诉 AI 什么是标准答案，从而极大地稳定 AI 的输出质量。                                                |
| **CoT (Chain of Thought)** | **思维链**。引导 AI 在给出最终答案前，先写下推理过程。这不仅能减少 AI “一本正经胡说八道”的概率，还能增强用户的信任感。                                                 |
| **Streaming JSON**         | **流式 JSON**。一种技术手段，允许前端在完整的 JSON 字符串还没传输完时，就通过特殊的解析算法提前读取已生成的字段，消灭 Loading 焦虑。                                   |
| **Headless Components**    | **无头组件**。指那些只提供交互逻辑（如：下拉菜单如何开关、如何支持键盘）而不提供样式的组件。`shadcn/ui` 里的 `Radix UI` 就是典型的无头组件，它让你能 100% 自定义外观。 |
| **Type Widening**          | **类型拓宽**。TypeScript 的一种行为，会将 `'system'` 这种字面量看作普通的 `string`。在 AI SDK 中需要使用 `as const` 来锁定类型，确保类型检查通过。                     |

---

### 4.部署到 Vercel，并添加流量限制

流量限制（Rate Limiting）功能，确保在部署到 Vercel 后能够有效防止 API 被滥用。

**主要改动：**

1. **引入 Upstash Redis：**  安装了    和  ，这是 Vercel 环境下实现分布式限流的标准方案。

   ```
   @upstash/ratelimit
   ```

   ```
   @upstash/redis
   ```

2. 去 redis 官网注册一个 redis 实例，并获取到`UPSTASH_REDIS_REST_URL`，`UPSTASH_REDIS_REST_TOKEN`

   然后把`UPSTASH_REDIS_REST_TOKEN`，`UPSTASH_REDIS_REST_URL` 填入 vercel 中的环境变量中。本地的.env.local 拉取一次

3. **创建限流工具：**  在    中配置了限流策略（默认每 10 秒允许 10 次请求，按 IP 维度限制）。

   lib/ratelimit.ts

   ```tsx
   import { Ratelimit } from "@upstash/ratelimit";
   import { Redis } from "@upstash/redis";

   // Create a new ratelimiter, that allows 10 requests per 10 seconds
   export const ratelimit = new Ratelimit({
     redis: Redis.fromEnv(),
     limiter: Ratelimit.slidingWindow(10, "10 s"),
     analytics: true,
     /**
      * Optional prefix for the keys used in redis. This is useful if you want to share a redis
      * instance with other applications and want to avoid key collisions. The default prefix is
      * "@upstash/ratelimit"
      */
     prefix: "@upstash/ratelimit",
   });
   ```

4. **后端集成：**  更新了  。现在每个请求都会先检查频率，如果超过限制将返回    状态码。

   ```
   app/api/analyze/route.ts
   ```

   ```tsx
   // 0. 流量限制 (Rate Limiting)
   const ip = (await headers()).get("x-forwarded-for") ?? "127.0.0.1";
   const { success, limit, reset, remaining } = await ratelimit.limit(
     `ratelimit_analyze_${ip}`
   );

   if (!success) {
     return new Response(
       JSON.stringify({ error: "请求过于频繁，请稍后再试" }),
       {
         status: 429,
         headers: {
           "Content-Type": "application/json",
           "X-RateLimit-Limit": limit.toString(),
           "X-RateLimit-Remaining": remaining.toString(),
           "X-RateLimit-Reset": reset.toString(),
         },
       }
     );
   }
   ```

**部署前准备：**  在部署到 Vercel 之前，请确保在 Vercel 控制台或

```
.env.local
```

中配置以下环境变量：

- `UPSTASH_REDIS_REST_URL`
- `UPSTASH_REDIS_REST_TOKEN`

你可以在  [**Upstash 控制台**](https://console.upstash.com/)  免费创建一个 Redis 实例来获取这些密钥。

详情请查看  **Walkthrough**。

## 🚀 接下来我们的计划

我们已经打通了 **“输入 -> 流式分析 -> 持久化 -> UI 反馈”** 的闭环。

**明天的重点是：模块 2 —— 记忆引擎（RAG）**

1. **激活向量字段**：在数据库中开启 `vector` 扩展。
2. **Embedding 流程**：学会如何将分析记录转化成 AI 能“搜索”的数学向量。
3. **语义召回**：让 AI 能够检索历史记录，真正变成一个“有记忆”的私人诊断专家。
