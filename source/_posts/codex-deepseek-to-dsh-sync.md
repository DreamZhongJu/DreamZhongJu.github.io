---
title: 把 Codex 接 DeepSeek 的历史会话，无损迁移到 DeepSeek Harness
date: 2026-08-14 12:00:00
updated: 2026-08-14 12:00:00
categories:
  - 项目复盘
tags:
  - DeepSeek
  - Codex
  - DeepSeek Harness
  - 插件
  - Agent
description: 复盘一个把「接入 DeepSeek 的 Codex 本地会话」导入到 DeepSeek Harness 的插件：从两半插件架构、事件源会话模型，到 live/resume、cwd、标题这几个坑，以及合规边界的判断。
---

## 为什么做这个

前阵子我把 DeepSeek 通过 `model_providers` 接进了 OpenAI Codex CLI，用起来不错，历史会话也落在本机 `~/.codex/sessions/**/rollout-*.jsonl` 里。后来我打算切到 DeepSeek Harness（dsh），却发现一个很现实的问题：**Codex 里的历史对话带不走**。

Codex 的会话日志是给 Codex 自己重放用的，直接读出来是一堆 `session_meta`、`response_item` 事件，不是普通聊天记录。我想把这些历史变成 dsh 里的会话，能在 dsh 里继续看、继续聊。于是有了 [dsh-codex-sync](https://github.com/DreamZhongJu/dsh-codex-sync)。

## 目标

只做一件事，但要做对：

1. 列出本机 Codex 里的 DeepSeek 会话；
2. 单个 / 批量导入成 dsh 会话；
3. 保留标题、保留工作目录，导入后能直接接着干活。

## 架构：dsh 的插件是「两半」

dsh 是「一切都是插件」的 Cordis 框架。浏览器里读不到本机 `~/.codex`，所以这个插件必须拆成两半：

- **Host 端**：运行在 Node 里，读 Codex 日志，通过 Typert RPC 暴露 `codexSync/list` 和 `codexSync/import`。
- **Client 端**：React 组件，注册进设置页的 slot，调 Host 的 RPC 画界面。

这也是 dsh 里「能力缝」的标准三件套：Service Definition / Provider / Consumer。我的 Host 服务用 `TypertRemoteService` + `@Remote('import')` 声明方法，构建时 Typert 自动生成 wire codec 和远程接口，客户端通过 `ctx.remote.codexSync.import(...)` 调用。

## 事件源会话：导入不是「写文件」

dsh 的会话是**事件源**的 append-only log。要导入历史，不能直接塞一段文本，而是要重建事件序列：

```text
turn/start → step/start → user/message → assistant/message → step/end → turn/end
```

每条 `user/message` 用 `createUserMessage`，每条 `assistant/message` 用 `createAssistantMessage`，消息内容就是 Codex `response_item` 里 `input_text` / `output_text` 的文本块。

## 踩过的三个坑

**1. live vs resume**

我一开始用 `ctx.sessions.create()`，导入的会话一直挂在内存 live store 里。结果在侧边栏点它时，dsh 想从持久化「恢复」这个会话，报 `cannot prepare session while it is live`。正确姿势是 `prepare → enter → announce → append → flush → detach`，写进磁盘后立刻从 live store 摘除，才能被 resume。

**2. 工作目录**

Codex 的 `session_meta` 里每个会话都记了 `cwd`。把它写进 dsh 会话的 `meta.cwd`，导入后 bash / 文件工具才在原来的项目目录执行，而不是 dsh 启动目录。

**3. 标题**

dsh 的标题是一条 log-only 的 `session/title` 事件。要沿用 Codex 的 `thread_name`，得追加 `{ source: { kind: 'user' }, messageSeqs: [] }` 把它「钉」住，否则 dsh 的自动标题会覆盖。

## 合规边界

这个插件只读**你自己电脑上**的 Codex 本地日志，不联网、不抓服务端、不上传任何内容。依据 OpenAI 使用条款（输入 / 输出所有权归用户）和 DeepSeek 开放平台协议（输入 / 输出归用户，可用于个人使用），本地读取 / 导入自己的数据不构成违规。真正会踩线的是：用非官方接口抓 ChatGPT / Codex 云端数据、绕过鉴权限流、或拿输出训练与 OpenAI 竞争的模型。这些我都没做。

## 现状与后续

目前还在开发中：`dsh plugin add` 一键安装、导入去重、目录选择器都在计划里。仓库见 [DreamZhongJu/dsh-codex-sync](https://github.com/DreamZhongJu/dsh-codex-sync)，欢迎 issue / PR。