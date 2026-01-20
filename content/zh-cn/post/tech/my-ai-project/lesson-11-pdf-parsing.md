+++
title = "第十一课_PDF 解析与数据清洗"
date = 2026-01-21T01:11:00+08:00
draft = false
categories = ["技术", "AI", "智能产品架构师"]
+++

## 🎯 核心问题

**问题**: 如何在 Node.js 环境下稳定、高效地解析 PDF 文档，并清洗掉因 PDF 布局导致的"乱码"和冗余空格，为 RAG 系统提供高质量的 Embedding 输入？

**重要性**:

- **数据质量**: 垃圾进，垃圾出 (Garbage In, Garbage Out)。解析失败或乱码会直接污染向量数据库。
- **成本控制**: 二进制乱码会导致 Token 消耗爆炸 (44k vs 几百)，造成巨大的 API 成本浪费。
- **系统稳定性**: 依赖冲突和未捕获的异常会导致服务崩溃。

## 🧠 核心概念图解

```ascii
+----------------+      +------------------+      +------------------+
|  PDF Document  | ---> |  Binary Parser   | ---> |  Raw Text Stream |
+----------------+      |  (pdfjs-dist)    |      +------------------+
                        +------------------+               |
                                                           v
+----------------+      +------------------+      +------------------+
| Clean Text     | <--- | Semantic Cleaner | <--- |  Layout Noise    |
| (Ready for DB) |      | (Regex/Heuristic)|      |  (Spaces/Garbage)|
+----------------+      +------------------+      +------------------+
```

## 📐 架构设计

### 解析流水线 (Pipeline)

1.  **Input Validation**: 检查 MIME 类型，拒绝非 PDF 文件。
2.  **Safe Parsing**: 使用 `pdfjs-dist` (Legacy Build) 在沙箱/非 Worker 模式下解析。
3.  **Error Handling**: 严格捕获解析错误，**禁止**回退到 `buffer.toString()`。
4.  **Text Cleaning**:
    - 去除控制字符。
    - 合并被 PDF 布局打断的中文句子 (去除冗余空格)。
    - 规范化换行符。

## 🔧 核心伪代码

```typescript
function processPdf(buffer):
  try:
    // 1. Load with pdfjs-dist (Legacy)
    doc = pdfjs.getDocument({ data: buffer, disableWorker: true })

    fullText = ""
    for page in doc.pages:
      // 2. Extract text items
      text = page.getTextContent()
      fullText += text

    // 3. Clean Text (Critical!)
    // Remove spaces between Chinese characters: "中 国" -> "中国"
    cleanedText = fullText.replace(/([\u4e00-\u9fa5])\s+([\u4e00-\u9fa5])/g, '$1$2')

    return cleanedText

  catch error:
    // 4. Fail Safe
    log("PDF Parse Failed")
    throw error // Do NOT return raw buffer string!
```

## 🛠 关键技术点

1.  **`pdfjs-dist` Legacy Build**:
    - Node.js 环境下必须使用 `pdfjs-dist/legacy/build/pdf.mjs`。
    - 现代 ESM 环境与旧版 CommonJS 的兼容性处理。

2.  **Worker 禁用 (`disableWorker: true`)**:
    - 在服务端环境禁用 Worker 线程，避免复杂的路径解析和多线程开销，提高稳定性。

3.  **正则清洗 (Regex Cleaning)**:
    - 利用 Unicode 范围 `[\u4e00-\u9fa5]` 识别中文字符。
    - 模式匹配：`汉字 + 空格(1个或多个) + 汉字` -> 替换为 `汉字 + 汉字`。

## ⚠️ 踩坑记录

### 1. `TypeError: Class constructors cannot be invoked without 'new'`

- **原因**: `pdf-parse` 等老库在 ESM 模式下导出不兼容。
- **解决**: 迁移到 `pdfjs-dist` 并使用动态导入。

### 2. 二进制乱码陷阱 (%PDF-1.7)

- **现象**: 解析"成功"但内容是 `%PDF...`，Token 消耗 4w+。
- **原因**: `catch` 块中错误地使用了 `buffer.toString()` 作为兜底，将二进制流当文本处理。
- **解决**: 解析失败直接抛出错误，绝不返回原始 Buffer 字符串。

### 3. 中文乱码/空格

- **现象**: "R A G 架 构"
- **原因**: PDF 内部通过定位排版，文字间没有实际空格，但提取时被视为独立 Item 拼接了空格。
- **解决**: 后处理正则清洗。

## 📊 知识网络

- **前置知识**: Node.js Buffer, ESM/CJS 模块规范, 正则表达式。
- **关联课程**: Lesson 09 (Embedding 服务) - 清洗后的数据直接输入 Embedding。
- **后续应用**: RAG 检索质量优化 (Chunking 策略)。

## 🎯 费曼检验

**"PDF 解析就像是从一张拍扁的照片里还原文字。机器看到的不是连贯的句子，而是散落在坐标上的一个个字。我们的工作就是把这些散落的字捡起来，拼成通顺的句子，并把照片底片（二进制数据）扔掉，千万别把底片当成文字读给别人听。"**

## 📝 实战练习

编写一个工具函数 `cleanPdfText`，实现以下功能：

1.  去除首尾空白。
2.  将多个连续换行符替换为两个换行符。
3.  **核心**: 去除两个汉字之间的所有空白字符（空格、制表符、换行）。

```typescript
export function cleanPdfText(text: string): string {
  // 实现你的代码
  return text
    .trim()
    .replace(/\n{3,}/g, "\n\n")
    .replace(/([\u4e00-\u9fa5])\s+([\u4e00-\u9fa5])/g, "$1$2");
}
```

## ✅ 检查清单

- [ ] 确认使用了 `pdfjs-dist` 的 legacy build。
- [ ] 确认禁用了 Worker 线程。
- [ ] 检查了错误处理逻辑，确保没有 `buffer.toString()` 回退。
- [ ] 验证了中文文档解析后的空格清洗效果。
- [ ] 监控了 Token 消耗，确保没有异常飙升。
