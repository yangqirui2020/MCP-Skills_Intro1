## 从聊天模型到智能执行体：MCP 与 Agent Skills 的实践笔记
在以往使用大语言模型（LLM）的过程中，往往会遇到这些问题：

* **无法接入外界工具：** 模型往往给人聊天机器人的印象，能力被固定在了网页端的聊天界面中无法直接访问外部数据、工具或系统。这并非模型能力不足，而是缺乏一种标准化、可控的方式让模型接入真实世界的能力


* **能力难以复用：** 在实际应用中，模型的上下文高度依赖手工拼接的 prompt，能力接入方式零散，难以复用和扩展，稳定性和可控性不足

模型上下文协议 MCP（Model Context Protocol）解决了大模型如何规范地接入外部能力的问题，而 Agent Skills 则进一步将这些能力模块化、可组合，使模型能够在明确边界下执行任务，实现大模型从聊天机器人到智能执行体的转变

## 1 模型上下文协议：拓展大模型的能力边界
### 1.1  MCP 与外部工具的交互
![MCP 核心组件](C:/Users/yang/Desktop/MCP介绍/MCP核心组件.png)

MCP（Model Context Protocol）是一套让 AI 模型与外部工具、数据源安全协作的协议。MCP 具备以下特点：


* 统一接口：不同厂商的 MCP 服务器都暴露类似的“工具”和“资源”，客户端（如 Cursor）用同一套方式调用；
* 真实数据：模型通过 MCP 调用的工具（如 search_papers、download_paper、read_paper）会真正访问 arXiv 等数据源，拿到的是真实存在的论文元数据和正文；
* 可追溯：用户可以看到模型调用了哪些工具、拿到了哪些论文 ID 和内容，便于核对和复现。

### 1.2 MCP 实战：使用 arxiv-mcp-serve 检索与下载文献
以前让 AI 大模型推荐相关论文时，经常遇到：
* 虚构文献：模型会编造不存在的论文标题、作者、期刊或会议；
* 张冠李戴：把真实论文的结论、方法安到错误的论文上；
* 无法核实：用户没有直接访问原文的途径，难以验证 AI 给出的引用是否真实。
![MCP 关系图](C:/Users/yang/Desktop/MCP介绍/MCP关系图.png)


读取论文场景下，MCP 的价值是：把文献的检索与阅读交给专门的服务（arxiv-mcp-server），模型只基于这些真实结果进行总结、对比、分析，从而显著减少虚构文献和错误引用。

**MCP：arxiv-mcp-server**
这是一个名为 **ArXiv MCP Server** 的项目。它的核心作用是充当 AI 助手与 arXiv 学术论文库之间的桥梁，通过模型上下文协议（Model Context Protocol, MCP）让 AI 能够以编程方式搜索、获取和分析学术论文。

主要功能包括：

   **论文搜索 (Paper Search)**
* 允许用户通过关键词查询 arXiv 论文。
* 支持高级过滤功能，包括按日期范围（`date_from`）和学科类别（如 `cs.AI`, `cs.LG`）筛选结果。


**论文下载与访问 (Paper Download & Access)**
* 提供工具 (`download_paper`) 根据 arXiv ID 下载特定论文。
* 下载的论文会被保存在本地存储中，以便快速访问。

**论文阅读与转换 (Read Paper)**
* 提供工具 (`read_paper`) 让 AI 读取已下载论文的全文内容。
* 将 PDF 格式的论文转换为 Markdown 格式，以便大语言模型更容易理解和处理。

**论文管理 (Paper Listing)**
* 提供工具 (`list_papers`) 查看当前已下载并存储在本地的所有论文列表。


**研究辅助提示词 (Research Prompts)**
* 内置了专门的研究提示词（`deep-paper-analysis`），这不仅仅是一个工具，而是一个完整的工作流。
* 指导 AI 系统地分析论文，涵盖执行摘要、研究背景、方法论分析、结果评估、未来研究方向等多个维度。

### 1.3 在 Cursor 中安装与使用arxiv-mcp-server
* 确保本机已安装 uv https://github.com/astral-sh/uv，
* 在终端执行：uv tool install arxiv-mcp-server
* 安装成功以后：
![arxiv-mcp-server ](C:/Users/yang/Desktop/MCP介绍/arxiv_mcp.png)
* 使用效果（在ArXiV上下载5篇语义分割相关的论文）
![下载 Arxiv 论文示例](C:/Users/yang/Desktop/MCP介绍/下载arxiv论文.png)
* 下载以后的汇总：
![完成下载 Arxiv 论文示例](C:/Users/yang/Desktop/MCP介绍/完成下载arxiv论文.png)
* 本地打开下载的论文
![本地打开下载的论文](C:/Users/yang/Desktop/MCP介绍/本地打开下载的论文.png)
## 2 利用智能体技能完善提示词
---
> **Agent Skills 是把“大模型能做的事”拆成可复用、可组合、可约束的能力模块，让模型从“会说”变成“会干活”**

* MCP 解决的是 **“模型如何安全、规范地接入外部能力”**
* Agent Skills 解决的是**模型“在什么条件下，用什么能力，按什么步骤做事”**
---

### 2.1 为什么需要 Agent Skills

在没有 Skills 的情况下，AI Agent 往往是这样的：

* 所有能力 **塞在一个超长 prompt 里**
* 行为逻辑靠“模型自己悟”
* 不同任务之间 **prompt 无法复用**
* 同一个模型：

  * 有时会查资料
  * 有时会胡编
  * 有时会越权调用工具

典型问题包括：

*  **不稳定**：同一个 prompt，多次执行结果差异很大
*  **不可控**：模型什么时候用工具、用哪个工具，说不清
*  **不可组合**：A 任务的 prompt 很难直接用于 B 任务
*  **不可维护**：prompt 越写越长，没人敢改

本质原因：
**能力、流程、策略”全部混在自然语言里**

---

### 2.2 Agent Skills 是什么（核心定义）

从工程角度，一个 **Skill** 通常包含 4 个要素：

 **能力边界（What can I do）**

   * 这个 Skill 负责做什么
   * 不负责做什么

 **触发条件（When to use）**

   * 在什么任务 / 状态下启用
   * 是否需要前置条件

 **执行方式（How to do）**

   * 内部 prompt 结构
   * 是否调用 MCP tools
   * 是否分步骤执行

 **输入 / 输出规范（I/O Contract）**

   * 输入需要什么信息
   * 输出必须满足什么格式

可以把 **Skill** 理解成：

> **给 Agent 用的、带说明书的功能模块**


```
模型
 ├── MCP（工具与数据接入层）
 └── Skills（能力与决策层）
```

---

### 2.3 Agent Skills 概念结构

一个 Skill 在概念上通常包含：

```text
Skill 名称
├── 使用场景说明
├── 输入参数定义
├── 内部 prompt / 规则
├── 可调用的工具（通常经 MCP）
└── 输出格式约束
```
**重点不是“写 prompt”，而是“写规则 + prompt 的组合”**

### 2.4 使用Agent Skills完善提示词

![promptlookuo](C:/Users/yang/Desktop/MCP介绍/skills_promptlookup.png)
**项目概述**

**名称：** prompt-lookup
**平台：** SkillsMP（Agent Skills Marketplace）
**目的：** 提供针对提示词（prompts）的搜索、获取和增强功能，让 AI 代理在对话中更快、更准确地找到和改进合适的提示词。([SkillsMP][1])


---

**功能与用途**

**搜索提示词模板**

当用户需要某类提示（比如 “写代码 review 提示模板”）时：

✔ 根据关键词在 prompts 库中检索匹配的提示
✔ 按类别、标签、关键词过滤结果
✔ 返回标题、描述、作者、标签等信息
功能由 `search_prompts` 工具实现。([SkillsMP][1])


**获取具体提示词**

如果你知道具体需要某个提示：

✔ 可以根据提示 ID 直接获取
✔ 如果提示带有变量（如 `${variable}`），Skill 会提示你填值
✔ 填好变量后返回完整可用的提示文本
功能由 `get_prompt` 工具实现。([SkillsMP][1])


**提示词增强（Improve Prompt）**

对于已有提示：

✔ 提升其质量或准确度
✔ 针对不同输出类型（文本、结构化 JSON/YAML、图像等）优化
✔ 返回改进后的版本并附带解释
功能由 `improve_prompt` 工具提供。([SkillsMP][1])


**何时应触发此 Skill**

Skill 会在以下对话场景激活：

* 用户询问提示词模板或搜索提示词
* 想查找已存在的提示词
* 用户想让现有提示词更好用
* 提到 prompts.chat 或提示库
  这样的上下文通过 Skill 的触发规则匹配，Claude 自动启用。

---

### 2.5 如何使用prompt-lookup

该 Skill 是按照 **SkillsMP / Agent Skills 规范**定义的，使用方法大致如下：

   * 在 AI 代理环境中 **安装 prompt-lookup Skill**
   例如通过命令行工具：

   ```bash
   npx skills add f/prompts.chat
   ```

（取决于 SkillsMP CLI/安装方式）
*  在对话中，当你问到提示词相关问题时，**Claude 会自动识别并启用这个 Skill**。无需显式手动调用。

  * 使用 AI 提示语或 Skill 调用命令（根据不同 AI 代理平台可能不同）：

   * 发送类似 “查找 Python 数据处理提示模板” 的问题

   系统便会调用 `prompt-lookup` 的相关工具返回结果。

---

[1]: https://skillsmp.com/zh/skills/f-prompts-chat-plugins-claude-prompts-chat-skills-prompt-lookup-skill-md "prompt-lookup - Agent Skill by f | SkillsMP"
[2]: https://skillsmp.com/?utm_source=chatgpt.com "SkillsMP: Agent Skills Marketplace - Claude, Codex ..."



## 3 对比MCP与Agent Skills
| 维度       | MCP       | Agent Skills |
| -------- | --------- | ------------ |
| 解决问题     | 能力接入      | 能力组织         |
| 关注点      | 接口、协议、安全  | 行为、流程、策略     |
| 是否和模型强相关 | 相对弱       | 很强           |
| 是否偏基础设施  | 是         | 否（偏上层）       |
| 类比       | USB / API | 应用程序 / 工作流   |

## 4 总结
* **MCP：** 让大模型规范接入外部工具与数据的协议，统一接口、真实调用、可追溯。
* **ArXiv MCP Server**：  用 MCP 连接 arXiv，支持搜索、下载、阅读论文（转 Markdown），减少虚构文献。
* **Agent Skills：** 把模型能力拆成可复用模块（边界、触发、执行、I/O），从“会说”到“会干活”。
* **分工：** MCP 管“怎么接”（接口与数据），Skills 管“何时用、怎么用”（流程与策略）。
*  MCP 像 USB/API（基础设施），Skills 像应用/工作流（上层），一起支撑智能执行体。