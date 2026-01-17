+++
title = "Antigravity Skills 安装与使用指南"
date = 2026-01-17T16:00:00+08:00
draft = false
categories = ["技术", "Antigravity"]
tags = ["Antigravity", "Skills", "AI Agent"]
+++

# Antigravity Skills 安装与使用指南

这份指南详细说明了如何在您的环境中安装和使用 Google Antigravity Skills。

## 1. 安装过程

Antigravity Skills 是通过将技能定义文件放置在特定的全局目录中来安装的。

### 安装位置

全局技能目录位于：

- **Windows**: `%USERPROFILE%\.gemini\antigravity\skills`
- **Mac/Linux**: `~/.gemini/antigravity/skills`

### 安装步骤回顾

我们执行了以下操作来安装官方示例技能库：

1.  **创建目录**：确保全局技能目录存在。
2.  **克隆仓库**：从 GitHub 克隆了 `rominirani/antigravity-skills` 仓库。
    ```powershell
    git clone https://github.com/rominirani/antigravity-skills "%USERPROFILE%\.gemini\antigravity\skills"
    ```
3.  **调整结构**：将 `skills_tutorial` 目录下的技能移动到了根目录，以便 Agent 能直接识别。

现在，您的技能目录结构如下：

- `adk-tool-scaffold/`
- `database-schema-validator/`
- `git-commit-formatter/`
- `json-to-pydantic/`
- `license-header-adder/`

---

## 2. 如何使用 Skills

Antigravity Skills 旨在增强 AI Agent 的能力。一旦安装，您可以通过自然语言指令直接调用这些技能。Agent 会根据技能定义（`SKILL.md`）自动识别何时使用它们。

以下是已安装技能的具体用法示例：

### 1. Git Commit Formatter (git-commit-formatter)

- **功能**：拦截提交请求，并根据 Conventional Commits 规范格式化提交信息。
- **如何使用**：
  在聊天中告诉 Agent 您想提交代码，例如：
  > "帮我提交这些更改，信息是'修复了登录页面的 bug'"
  > "Commit these changes with message 'added new feature'"
- **效果**：Agent 会自动将您的非结构化消息转换为标准格式（如 `fix: resolve login page bug`）。

### 2. License Header Adder (license-header-adder)

- **功能**：为源代码文件添加标准的 Apache 2.0 许可头。
- **如何使用**：
  > "给当前打开的文件添加 License 头"
  > "Add license headers to all python files in the src directory"
- **效果**：Agent 会读取 `resources/HEADER_TEMPLATE.txt` 并在文件顶部插入许可文本。

### 3. JSON to Pydantic (json-to-pydantic)

- **功能**：将 JSON 数据转换为 Python Pydantic 模型代码。
- **如何使用**：
  提供一段 JSON 数据并要求转换：
  > "把这个 JSON 转换成 Pydantic 模型：{...}"
  > "Create a Pydantic model for the user response API"
- **效果**：Agent 会参考技能中的 "Golden Example"（少样本学习），生成准确的 Python 类定义。

### 4. Database Schema Validator (database-schema-validator)

- **功能**：验证 SQL 架构文件的安全性和命名规范。
- **如何使用**：
  > "检查一下 schema.sql 文件是否合规"
  > "Validate the database schema"
- **效果**：Agent 会运行 Python 脚本 `validate_schema.py` 来执行确定性的检查，确保 100% 准确。

### 5. ADK Tool Scaffold (adk-tool-scaffold)

- **功能**：脚手架工具，用于创建新的 Antigravity ADK 工具。
- **如何使用**：
  > "帮我创建一个新的天气查询工具"
  > "Scaffold a new tool named StockTicker"
- **效果**：Agent 会编排整个流程：生成文件、应用模板、并根据参考示例指导实现。

## 3. 注意事项

- **生效时间**：安装新技能后，通常需要**重启 Agent 会话**（或重新加载窗口）才能生效。
- **自定义技能**：您可以参考 `%USERPROFILE%\.gemini\antigravity\skills` 下的现有文件夹结构，创建自己的 `SKILL.md` 来扩展 Agent 的能力。

## 4. 常见问题与高级指南

### 4.1 是否有 GUI 操作界面？

目前 Google Antigravity **官方没有内置专门的图形化（GUI）技能商店或管理器**。
它的设计理念是 **"Agentic Command Line"（代理式命令行）**，即通过**文件系统**来管理技能。

- **您的“GUI”就是资源管理器**：您通过在文件夹中添加/删除文件来管理技能，就像管理代码一样。
- 社区可能存在第三方工具，但官方推荐的方式是直接操作 `.agent/skills` 目录。

### 4.2 如何安装第三方 Skills？

安装第三方技能的通用流程是：

1.  **获取技能包**：通常是一个包含 `SKILL.md` 和相关脚本的文件夹（可以从 GitHub 等开源社区下载）。
2.  **放入目录**：
    - **全局生效**：复制到 `%USERPROFILE%\.gemini\antigravity\skills\`
    - **仅当前项目生效**：复制到您项目根目录下的 `.agent/skills/`
3.  **重启生效**：通常需要重启 Agent 会话。

### 4.3 如何保证 Skills 符合 HHH 法则？

由于 Antigravity Skills 是开放的文本和代码，**安全性主要掌握在您手中**。平台本身不预先审查所有第三方技能，您需要采取以下措施来确保符合 **HHH (Helpful, Honest, Harmless)** 法则：

#### Honest (诚实) - 透明性审查

- **必读 `SKILL.md`**：这是技能的“大脑”。打开它，阅读 Prompt 定义。看它是否试图欺骗、隐瞒信息或通过 Prompt Injection 绕过限制。
- **检查 Prompt**：确保它没有包含类似 "Ignore previous instructions"（忽略之前的指令）等恶意引导。

#### Harmless (无害) - 代码审计

- **检查脚本**：如果技能包含 Python/Shell 脚本（如 `scripts/` 目录下），**务必在运行前阅读代码**。
- **权限控制**：看脚本是否请求了不必要的网络访问、文件删除或系统修改权限。
- **沙箱运行**：Antigravity 通常在受限环境中运行，但您仍应避免运行来源不明的混淆代码。

#### Helpful (有益) - 实用性评估

- 测试技能是否真的解决了问题，还是只是在空转。您可以先在一个测试项目中试用，确认效果后再全局安装。

**总结**：Antigravity 的强大在于其开放性，但这也意味着您需要像对待开源代码库一样对待 Skills——**先审查，再信任**。

## 5. 专家建议：如何更高效地使用 Skills

作为一名 Skills 开发专家，我为您总结了以下几点技巧，帮助您从“会用”进阶到“精通”：

### 5.1 提高触发成功率 (Triggering)

Agent 是根据 `SKILL.md` 中的描述来决定是否调用技能的。为了提高触发率：

- **使用关键词**：打开 `SKILL.md`，查看 `description` 字段。在您的 Prompt 中包含这些关键词。
  - _例如_：如果技能描述中有 "scaffold new tool"，那么用 "scaffold a tool" 比 "create a new project" 更容易触发。
- **明确意图**：不要含糊其辞。直接说 "使用 [技能名] 来做..." 是最保险的。

### 5.2 调试与排错 (Debugging)

如果技能没有按预期工作：

1.  **检查 SKILL.md**：确保文件格式正确，YAML 头没有语法错误。
2.  **查看 Agent 思考过程**：在 Antigravity 的输出中，留意 Agent 是否提到了 "Thinking about using skill..." 或 "Reading skill definition..."。如果没有，可能是描述不匹配。
3.  **脚本测试**：如果技能包含脚本（如 `scripts/validate.py`），先在终端手动运行它，确保脚本本身没有 bug。

### 5.3 定制化您的 Skills (Customization)

官方技能只是起点，您可以随意修改它们以适应您的工作流：

- **修改模板**：觉得 License 头里的名字不对？直接编辑 `license-header-adder/resources/HEADER_TEMPLATE.txt`，把 "Copyright [Year]" 改成您的名字。
- **调整规范**：觉得 Git 提交信息格式太严格？修改 `git-commit-formatter/SKILL.md` 中的 Prompt 指令，让它允许更宽松的格式。
- **添加示例**：在 `json-to-pydantic/examples/` 中添加更多您公司的特有数据格式，让 Agent 学习您的编码风格（Few-Shot Learning）。

记住，**Skills 本质上就是文件**。像管理代码一样管理它们，您就能发挥 Antigravity 的最大威力。

## 6. 进阶概念：Skills vs MCP

在 AI 开发领域，您可能听说过 **MCP (Model Context Protocol)**。虽然它们都旨在扩展 Agent 的能力，但它们的定位和作用截然不同。

### 6.1 核心区别

| 特性         | Antigravity Skills                       | Model Context Protocol (MCP)                |
| :----------- | :--------------------------------------- | :------------------------------------------ |
| **定位**     | **"大脑" (Brains)** - 方法论与流程       | **"双手" (Hands)** - 连接与工具             |
| **本质**     | **文件包** (Markdown 指令 + 脚本)        | **开放标准/协议** (Client-Server 架构)      |
| **作用**     | 教 Agent **"如何做"** (How to do)        | 告诉 Agent **"有什么"** (What is available) |
| **典型场景** | 代码审查规范、特定测试流程、文档生成模板 | 连接数据库、访问 GitHub 仓库、操作 Slack    |
| **生命周期** | **按需加载** (Ephemeral) - 用完即走      | **持久连接** (Persistent) - 随时待命        |

### 6.2 协同工作

它们不是竞争关系，而是**互补关系**。

- **MCP** 负责修路搭桥，把外部工具（如 PostgreSQL 数据库、Linear 任务板）连接到 Agent 面前。
- **Skills** 负责指挥交通，告诉 Agent 在特定场景下该如何使用这些工具，以及遵循什么样的业务规范。

**举例说明**：
假设您要开发一个 "自动修复 Bug" 的功能：

1.  **MCP Server** 提供了一个 `git_diff` 工具和一个 `run_tests` 工具。
2.  **Skill** 定义了修复流程：
    - _第一步_：调用 `git_diff` 查看更改。
    - _第二步_：分析错误日志。
    - _第三步_：生成修复代码。
    - _第四步_：调用 `run_tests` 验证。
    - _第五步_：如果失败，重复上述步骤（最多重试 3 次）。

**总结**：MCP 给了 Agent **能力**，而 Skills 给了 Agent **智慧**。

## 7. 推荐 Skills

想要让您的 Agent 更强大？这里有三个我强烈推荐的“必装”技能（或概念），能瞬间提升您的开发效率。

### 7.1 Superpower (超能力)

- **简介**：这不仅仅是一个技能，而是一套让 Agent 突破限制的工具集。这里我们推荐 `obra/superpowers`，它包含了一整套完整的开发工作流（TDD、调试、规划等）。
- **为什么推荐**：它让 Agent 从“聊天机器人”变成了“系统管理员”，支持 TDD、系统化调试等高级工作流。
- **安装步骤**：
  1.  **克隆仓库**：
      ```powershell
      git clone https://github.com/obra/superpowers temp_superpowers
      ```
  2.  **复制技能**：将 `temp_superpowers/skills` 目录下的所有文件夹复制到您的全局技能目录：
      - Windows: `%USERPROFILE%\.gemini\antigravity\skills`
      - Mac/Linux: `~/.gemini/antigravity/skills`
  3.  **清理**：删除临时克隆的 `temp_superpowers` 文件夹。
      > _注：安装完成后，您将获得 `brainstorming`、`writing-plans`、`systematic-debugging` 等十几个强大的工程化技能。_

### 7.2 Planning with Files (基于文件的规划)

- **简介**：这是 Antigravity 的核心理念，也是一个极其强大的官方技能。它教 Agent 如何通过创建 `TODO.md`、`PLAN.md` 等文件来规划复杂的任务。
- **为什么推荐**：对于复杂任务（如重构整个模块），Agent 容易“走神”。这个技能强制 Agent 先写计划、再执行，大大提高了长任务的成功率。
- **安装地址**：通常作为 Antigravity 的**内置核心技能**提供，无需额外安装。您只需在 Prompt 中说：“创建一个 plan 文件来规划这次任务”，即可激活它。
  > **深入解析：GitHub 上的 vs Antigravity 原生**
  > 您可能会在 GitHub 上看到类似 `OthmanAdi/planning-with-files` 地址：`https://github.com/OthmanAdi/planning-with-files`的仓库。
  >
  > - **相同点**：核心理念一致，都源自 "Manus-style" 的持久化 Markdown 规划（即用 `TODO.md` 管理进度）。
  > - **不同点**：
  >   - **GitHub 版**：通常是社区为其他 Agent（如 Claude Code）实现的“平替版”技能包，或者是一个包含预设规则的 workspace 模板。
  >   - **Antigravity 原生版**：是 IDE 深度集成的功能。它不仅生成文件，还能联动 IDE 的 UI（如在侧边栏显示任务进度），并自动维护上下文（Artifacts），体验更丝滑。

### 7.3 NotebookLM (智能笔记本)

- **简介**：虽然 NotebookLM 是 Google 的独立产品，但将其作为 Skill 集成到 Antigravity 中是目前的流行玩法。通过这个 Skill，您可以让 Agent 直接读取您在 NotebookLM 中整理的论文、文档或会议记录。
- **为什么推荐**：它相当于给 Agent 外挂了一个“无限知识库”。您可以把几百页的技术文档喂给 NotebookLM，然后通过这个 Skill 让 Agent 随时查阅，实现超高精度的 RAG（检索增强生成）。
- **安装步骤**：
  1.  **克隆仓库**：
      ```powershell
      git clone https://github.com/PleasePrompto/notebooklm-skill temp_notebooklm
      ```
  2.  **复制技能**：将 `temp_notebooklm` 文件夹复制到您的全局技能目录：
      - Windows: `%USERPROFILE%\.gemini\antigravity\skills\notebooklm-skill`
      - Mac/Linux: `~/.gemini/antigravity/skills/notebooklm-skill`
  3.  **认证 (关键步骤)**：
      该技能需要访问您的 Google 账号。请在终端中运行以下命令进行一次性认证：
      ```powershell
      cd %USERPROFILE%\.gemini\antigravity\skills\notebooklm-skill
      python scripts/run.py auth_manager.py setup
      ```
      _(浏览器会自动弹出，请登录您的 Google 账号，登录成功后关闭浏览器即可)_
  4.  **添加笔记本**：
      认证完成后，您需要注册您想查询的笔记本（这一步可以直接在 Antigravity chat UI 中完成，把notebookLM 的 URL 复制粘贴进去即可）：
      ```powershell
      python scripts/run.py notebook_manager.py add --url "https://notebooklm.google.com/notebook/..." --name "My Docs" --description "Project documentation" --topics "docs"
      ```
  5.  **清理**：删除临时克隆的 `temp_notebooklm` 文件夹。

---

**结语**：Skills 的世界浩如烟海，这三个只是冰山一角。去探索，去创造，去定义属于您自己的 AI 助手吧！
