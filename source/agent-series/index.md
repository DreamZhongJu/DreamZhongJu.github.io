---
title: 从零搭一个自托管 Agent（系列）
date: 2026-08-11 12:00:00
layout: page
---

这个系列记录我如何从零搭建一个**自托管的飞书研究助手**：一个跑在自己服务器上的多工具 Agent，能联网搜索、读论文、查文档、归档知识库，并且可观测、可评测、可扩展。

## 为什么值得读

市面上讲 Agent 的文章大多是"用框架写个 demo"。这个系列不一样：它记录的是一个**真实在生产使用的项目**的完整工程决策——为什么用 LangGraph、记忆怎么治理、工具怎么即插即用、能力怎么评测。每一篇都附脱敏代码和真实踩坑。

## 阅读路径

### 第 1 篇：架构与工具层

**《自托管飞书研究助手：用 LangGraph 搭一个多工具 Agent》**

[https://dreamzhongju.github.io/2026/08/10/feishu-agent-langgraph/](https://dreamzhongju.github.io/2026/08/10/feishu-agent-langgraph/)

分层架构、LangGraph 工具调用循环、ToolResult 契约、写工具确认机制、飞书事件幂等，以及 DSML 文本工具调用的兼容处理。

### 第 2 篇：记忆治理

**《给 Agent 装上分层记忆：mem0/Letta 式记忆治理实践》**

[https://dreamzhongju.github.io/2026/08/10/agent-layered-memory/](https://dreamzhongju.github.io/2026/08/10/agent-layered-memory/)

核心/存档两层记忆、add/update/delete 操作语义、预算与遗忘、访问统计，以及"不装向量库"的轻量方案。

### 第 3 篇：能力扩展与评测

**《让 Agent 长出"外挂"并自证能力：MCP 客户端与评测闭环》**

[https://dreamzhongju.github.io/2026/08/10/agent-mcp-evaluation/](https://dreamzhongju.github.io/2026/08/10/agent-mcp-evaluation/)

MCP 动态工具接入、请求级可观测性、离线评测 + 真实请求回放 + CI 的三层评测闭环。

### 番外：自动化情报

**《自托管每日情报日报：从"看不过来"到"每天一条推送到飞书"》**

[https://dreamzhongju.github.io/2026/08/11/daily-intelligence-briefing/](https://dreamzhongju.github.io/2026/08/11/daily-intelligence-briefing/)

同一套自托管哲学的另一个应用：每天自动聚合 RSS/arXiv/新闻并用 LLM 整理成中文日报。

## 相关项目

- 飞书研究助手：[github.com/DreamZhongJu/feishu-research-assistant](https://github.com/DreamZhongJu/feishu-research-assistant)
- 每日情报日报：[github.com/DreamZhongJu/daily-intelligence-briefing](https://github.com/DreamZhongJu/daily-intelligence-briefing)

## 系列状态

持续更新中。接下来的主题：RAG 与知识图谱增强、大模型后训练、开源贡献实践。
