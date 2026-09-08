---
title: LLM 额度用完了，功能还能继续上线吗？——用确定性兜底撑住 Kairós Wiki 上线
date: 2026-09-09 00:10:00
updated: 2026-09-09 00:10:00
categories:
  - 项目复盘
tags:
  - Kairós
  - 工程实践
  - Docker
  - 架构设计
description: DeepSeek 账户余额归零，知识图谱抽取全线 402 的当口，还是把新功能（wiki 导出）接上了生产环境。记一次"LLM 不可用时功能还能不能继续演进"的真实决策，以及在一个不做代码热挂载、靠 docker cp + commit 热补丁的部署环境里怎么验证一次改动。
---

## TL;DR

给 Kairós 接一个新功能（群聊 wiki 导出）的当天，正好撞上 LLM 服务商账户欠费，知识图谱抽取（依赖 LLM）全线 402。按理说这时候该先修支付问题再谈新功能，但我们把两件事拆开处理了：**新功能本身不依赖 LLM，所以照常上线，只是先用"零成本兜底"跑起来，账户充值后再切换成 LLM 增强版**。这篇记这个决策为什么合理，以及在一个代码不挂载、只能靠 `docker cp` + `docker commit` 热补丁的生产环境里，怎么验证一次改动真的生效了。

## 背景：一次真实的"依赖不可用"

Kairós 部署在自己的服务器上，一个 Docker 容器跑着 Flask API + LangGraph agent，通过 Koishi 采集 QQ 群消息，POST 到 `/api/knowledge/ingest`，用 DeepSeek 做实体/关系抽取，写进 SQLite + Neo4j。检查线上状态时发现容器日志里在刷：

```
httpx: HTTP Request: POST https://api.deepseek.com/chat/completions "HTTP/1.1 402 Payment Required"
kairos.knowledge.extract: kg extraction call failed: APIStatusError
```

账户余额用完了。摄入接口本身还在正常返回 200——因为 `kg_extract.extract()` 内部把异常吞掉、返回空的 entities/relations，不让一次抽取失败拖垮整个 ingest 响应——但新消息实质上已经**不再产生新的图谱关系**，只是存了一份原始文档。这是个明确的降级：图谱在"停止生长"，但没有崩。

这时候提的新需求是给知识图谱加一层 wiki 导出。直觉上会觉得"LLM 都用不了了，还怎么做新的知识加工功能"，但拆开看会发现假设是错的：wiki 导出这个需求本身可以不依赖 LLM。

## 拆分：哪些能力天生不需要 LLM

把"从聊天记录里提炼知识"这个大任务拆开看，能发现层次：

| 层级 | 能力 | 是否需要 LLM |
| --- | --- | --- |
| 过滤 | 判断一条消息是否"有信息量" | 否，关键词 + 正则规则即可 |
| 归档 | 把选中的消息按主题分类、去重合并、记录来源 | 否，纯确定性逻辑 |
| 摘要/结构化 | 把一堆原始消息压缩成一段通顺的摘要，或抽取成实体关系三元组 | 是 |

知识图谱抽取（实体、关系）天生落在第三层——三元组抽取本质上是语义理解任务，规则很难做，这也是为什么它必须依赖 LLM，欠费就直接停摆。但 wiki 导出的最小可用版本，只需要前两层：筛出高价值消息，按结构归档成 Markdown，带上来源和时间戳。这个版本对读者已经有价值（比刷聊天记录快得多），而且完全不touch LLM API。

于是设计上把 `WikiCompiler` 的"摘要生成"做成一个可替换的 `compiler` 参数，默认给一个不调用任何外部服务的实现（原样摘录 + 分类归档），以后账户恢复了，换一个调 LLM 的 compiler 函数就能升级成"总结版"，接口不用改。这不是"等以后有空了再做的技术债"，是刻意的分层设计：**把任务拆到能不依赖 LLM 的最小颗粒度，先把那部分立即上线**。

## 部署环境本身也是约束：没有代码热挂载

这台服务器上 Kairós 的部署方式是 `docker-compose build`，代码在 `Dockerfile` 里用 `COPY kairos ./kairos` 打进镜像，只有 `data/`、`reports/` 这些运行时目录是 bind mount。也就是说改代码 → 重新 `docker compose up --build` 才是"标准流程"，但这台机器上一次完整构建要装 apt 包、npm 包、走代理，很慢。

查这台服务器的操作历史发现了已经在用的一套更快的模式：

```bash
docker cp kairos/knowledge/wiki.py kairos:/app/kairos/knowledge/wiki.py
docker cp kairos/knowledge/ingest.py kairos:/app/kairos/knowledge/ingest.py
docker restart kairos
docker commit kairos feishu-assistant-kairos:latest
```

直接把改动的文件拷进运行中的容器、重启进程加载新代码，验证没问题后再 `docker commit` 把当前容器状态固化成新镜像——这样下次 `docker restart`（不是重建）不会丢失改动，等下次真正需要重新 build 时（比如改了 `requirements.txt`）再走完整流程。这不是什么最佳实践，是这台个人服务器在"够用"和"折腾"之间的真实取舍，但至少要知道自己在做什么：这种方式改的是镜像层而不是 Dockerfile，`docker-compose.yml` 和 git 仓库里的源码本身是脱节的，长期靠人记住"镜像里有哪些手动补丁"是会腐化的，只适合这种小规模个人项目、且有本地 git 仓库当基准的场景。

## 验证一次改动：不能只看"没报错"

热补丁流程里最容易偷懒的一步是"重启没报错就当成功了"。这次实际走的验证链路：

1. **容器内语法检查**：`docker cp` 完先跑一次 `docker exec kairos python -m py_compile ...`，避免语法错误留到运行时才炸（宿主机 Python 是 3.10，容器内是 3.12，两边对新语法的支持不一样，用容器自己的解释器编译才准）。
2. **重启后看日志有没有 Traceback**，`/health` 返回 200。
3. **真正调一次新接口，看返回值里该有的字段有没有出现**——不是 hit 一下 200 完事，而是构造一条会触发过滤器判定为"高价值"的消息，POST 到 `/api/knowledge/ingest`，确认响应体里 `wiki.status == "ok"`，再用 `GET /api/wiki/pages` 确认页面真的生成了，`GET /api/wiki/page?path=...` 能读到正文。
4. **顺手验证一个安全边界**：`GET /api/wiki/page?path=../../../etc/passwd` 要拿到 404 而不是文件内容，确认路径穿越防护在真实网络请求下也生效（单元测试里测过，但线上环境的 Flask 路由参数解码行为值得单独确认一次）。

第 3、4 步比第 1、2 步更容易被跳过，但恰恰是它们在验证"这次改动真的按预期工作"，而不是"这次改动没有让进程崩溃"——两者是完全不同的确信程度。

## 小结

"依赖服务不可用"经常被当成一个二元开关：能用/不能用，不能用就等它恢复。但拆开看任务边界，往往能找到一部分工作是可以立即推进的。这次的经验可以概括成一句话：**先问"这个功能里哪部分真的需要那个不可用的依赖"，而不是把整个功能一起挂起等依赖恢复**。

代码：[DreamZhongJu/kairos-intel](https://github.com/DreamZhongJu/kairos-intel)
