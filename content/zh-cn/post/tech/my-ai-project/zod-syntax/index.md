+++
title = "zod 语法"
date = 2025-12-26T00:44:00+08:00
draft = false
categories = ["技术", "AI",'智能产品架构师']
+++

学习地址： [官网](https://zod.dev/)

## [**介绍**](https://zod.dev/?id=introduction)

Zod 是一个以 TypeScript 为先的验证库。使用 Zod，您可以定义*模式* ，用于验证数据，从简单的`字符串`到复杂的嵌套对象。

```jsx
import * as z from "zod";

const User = z.object({ name: z.string() });
// some untrusted data...
const input = {
  /* stuff */
};
// the parsed result is validated and type safe!
const data = User.parse(input);
// so you can use it with confidence :)
console.log(data.name);
```

## [**特点**](https://zod.dev/?id=features)

- 零外部依赖
- 适用于 Node.js 和所有现代浏览器
- Tiny: 2kb 核心捆绑包（压缩）
- 不可变 API：方法返回一个新的实例
- 简洁界面
- 适用于 TypeScript 和纯 JS
- 内置的 JSON Schema 转换
- 广泛的生态系统

## [**安装**](https://zod.dev/?id=installation)

`npm install zod`

Zod 也可以在  [**jsr.io**](https://jsr.io/@zod/zod)  上使用  `@zod/zod`。

Zod 提供了一个 MCP 服务器，可以由代理使用来搜索 Zod 的文档。要为您的编辑器添加，按照以下步骤操作： [**这些说明**](https://share.inkeep.com/zod/mcp) 。Zod 还提供了一个  [**llms.txt**](https://zod.dev/llms.txt)  文件。

## [**要求**](https://zod.dev/?id=requirements)

Zod 经过了对 TypeScript v5.5 及更高版本的测试。较旧版本可能可以运行，但不被官方支持。

### [**`"strict"`**](https://zod.dev/?id=strict)

您必须在您的  `tsconfig.json`  中启用  `strict`  模式。这是所有 TypeScript 项目的最佳实践。

`// tsconfig.json{  // ...  "compilerOptions": {    // ...    "strict": true  }}`

## [**生态系统**](https://zod.dev/?id=ecosystem)

Zod 拥有一个蓬勃发展的生态系统，包括各种库、工具和集成。请参考[**生态系统页面**](https://zod.dev/ecosystem) ，了解支持 Zod 的完整库列表或基于 Zod 构建的工具。

- [**资源**](https://zod.dev/ecosystem?id=resources)
- [**API 库**](https://zod.dev/ecosystem?id=api-libraries)
- [**表单集成**](https://zod.dev/ecosystem?id=form-integrations)
- [**Zod to X**](https://zod.dev/ecosystem?id=zod-to-x)
- [**X to Zod**](https://zod.dev/ecosystem?id=x-to-zod)
- [**模拟库**](https://zod.dev/ecosystem?id=mocking-libraries)
- [**由 Zod 提供动力**](https://zod.dev/ecosystem?id=powered-by-zod)
