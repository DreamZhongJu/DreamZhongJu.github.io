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

### 番外：生产事故复盘

**《一次 DeepSeek thinking 模式 400 的事故复盘：reasoning_content 回传与自愈重试》**

[https://dreamzhongju.github.io/2026/08/11/deepseek-reasoning-content-400/](https://dreamzhongju.github.io/2026/08/11/deepseek-reasoning-content-400/)

凯伊在生产环境遇到的多轮对话 400 事故：根因是 DeepSeek thinking 模式的 `reasoning_content` 必须原样回传，隐藏坑是旧会话线程的存量数据，最终用"保存 + 回传 + 自愈重试"三步解决。

## Kairós 深潜系列（知识图谱支线）

### 总览

**《Kairós：一个自托管多渠道个人情报助手的架构与工程实践》**

[https://dreamzhongju.github.io/2026/08/18/kairos-architecture/](https://dreamzhongju.github.io/2026/08/18/kairos-architecture/)

25 个工具、知识图谱、技能系统、MCP 服务的整体架构复盘。

### 深潜 1：多 Provider 并发灌库

**《三线并发灌库实战：nous/zen/openrouter 故障转移、按请求钉扎与 74 窗/分钟》**

[https://dreamzhongju.github.io/2026/08/26/kairos-multi-provider-sharded-ingest/](https://dreamzhongju.github.io/2026/08/26/kairos-multi-provider-sharded-ingest/)

三级故障转移链、空响应保护、X-Kairos-Provider 钉扎、flock 分片断点，以及限流面前的并发配比实测。

### 深潜 2：实体身份对齐

**《群聊知识图谱的身份对齐：昵称、QQ 锚点与人工共指的三轨制》**

[https://dreamzhongju.github.io/2026/08/26/kairos-identity-alignment/](https://dreamzhongju.github.io/2026/08/26/kairos-identity-alignment/)

canonical 锚点模型、nickmap 反查、coref_overrides 人工共指表，以及 merge_entity 的"名字覆盖"坑与九马甲归宗案例。

### 深潜 3：超边挖掘改造

**《当 13.7 万条"推理链"只有 2% 是真知识：超边挖掘的锚点过滤改造》**

[https://dreamzhongju.github.io/2026/08/26/kairos-hyperedge-anchor-mining/](https://dreamzhongju.github.io/2026/08/26/kairos-hyperedge-anchor-mining/)

谓词白名单、锚点类型约束、证据计数重构，Neo4j 内存分批删除，以及模块缓存幽灵进程事故与人肉事实纠错通道。

### 深潜 4：JSONL 结构化重导与事件星实战

**《给知识图谱喂全量聊天记录：JSONL 结构化重导与事件星实战》**

[https://dreamzhongju.github.io/2026/08/26/kairos-jsonl-reimport-event-stars/](https://dreamzhongju.github.io/2026/08/26/kairos-jsonl-reimport-event-stars/)

45 万条消息带 QQ 号与毫秒时间戳全量重灌：发言人钉扎、@/回复确定性边、慢滴响应的墙钟防御、看门狗自愈，以及"核酸检测 @2022-09-18"这种从数据里自己长出来的时代切片。

### 番外：开源与申请复盘

**《一次 OSPP 申请的事故复盘：材料都做好了，为什么还是被拒》**

[https://dreamzhongju.github.io/2026/08/26/ospp-application-retrospective/](https://dreamzhongju.github.io/2026/08/26/ospp-application-retrospective/)

申请 OSPP 财报可追溯课题的完整复盘：抽取技术闭环都做完了、上游 PR 也合入了，却因为"交付的时机与形式"落选。五条归因 + 同课题另一条失败路线的旁证，附可执行的下次清单。

## 相关项目

- 飞书研究助手：[github.com/DreamZhongJu/feishu-research-assistant](https://github.com/DreamZhongJu/feishu-research-assistant)
- 每日情报日报：[github.com/DreamZhongJu/daily-intelligence-briefing](https://github.com/DreamZhongJu/daily-intelligence-briefing)
- Kairós 个人情报平台：[github.com/DreamZhongJu/kairos-intel](https://github.com/DreamZhongJu/kairos-intel)

## 系列状态

持续更新中。接下来的主题：RAG 与知识图谱增强、大模型后训练、开源贡献实践。
