---
title: LLM API 中转站探秘：一次"模型用不了"的完整排障与原理复盘
date: 2026-08-23 10:00:00
updated: 2026-08-23 10:00:00
categories:
  - 技术实践
tags:
  - LLM
  - API 网关
  - OpenAI 协议
  - 排障方法论
  - 中转站
description: 我使用的 LLM API 中转站出现了"某个供应商的模型全部用不了"的问题。本文完整记录分层排障过程（配置层→凭证层→协议层→上游层），并顺带讲透中转站原理、API 调用原理与 OpenAI 系协议（Chat Completions / Responses / Messages）的差异。
---

# LLM API 中转站探秘：一次"模型用不了"的完整排障与原理复盘

## TL;DR

我配置的模型中转站（一个聚合多家 LLM 供应商的统一网关）里，某个供应商的模型突然"全部用不了"。经过分层排障发现：**不是模型坏了，而是凭证权限、协议选择、模型 ID、上游故障四类问题叠加在一起**。其中最有价值的洞察是：

1. **同一个网关，不同 API Key 看到的是不同的模型列表** —— Key 是套餐级权限隔离的，不是简单认证；
2. **同一模型 ID，换一个 Key 就 404 → 200** —— 对照实验可以一秒钟定位是"模型问题"还是"Key 问题"；
3. **流式请求（`stream: true`）与普通请求在网关上的表现可能完全不同** —— 排障时不能只看非流式；
4. **`502/503 upstream_error` 是网关上游故障**，与你的配置无关，重试/换时段即可。

这篇文章既是排障记录，也借机把**中转站原理、API 调用原理、API 协议**讲清楚。

> 脱敏说明：文中中转站域名、API Key、供应商内部代号均已替换为占位符；模型名（如 gpt-5.5、claude-opus-4-8）是公开命名，予以保留以便对照。

---

## 一、什么是 LLM API 中转站

直连方式下，每个模型供应商（OpenAI、Anthropic、DeepSeek……）各有一套 API 域名、认证方式和协议细节，客户端要分别对接。**中转站（Relay / Gateway / 聚合 API）** 就是在你与各家模型之间插入一个统一入口：

```
┌──────────┐   HTTPS    ┌──────────────┐   转发   ┌─────────────┐
│  客户端   │ ─────────▶ │   中转站网关   │ ───────▶ │  真实上游     │
│ (Agent/  │  Bearer    │  /v1/...     │          │ (OpenAI 等)  │
│  SDK)    │  Key A     │  路由/鉴权    │          └─────────────┘
└──────────┘            └──────────────┘
```

中转站通常承担这些职责：

- **统一入口**：所有模型共用一个域名（如 `https://relay.example.com/v1`），客户端只需改 `base_url`；
- **多模型路由**：请求体里的 `model` 字段决定转发给哪家上游，一个 Key 通吃多家模型；
- **协议适配**：有的上游原生只支持某种协议，网关负责把请求/响应翻译成 OpenAI 兼容格式；
- **凭证与套餐管理**：为不同用户签发不同 Key，限制可访问的模型子集（套餐）；
- **计量计费与限流**：按 token 计费、配额控制、速率限制；
- **缓存与优化**：prompt 缓存、负载均衡、失败重试。

这次排障里，**"Key 决定你能看到哪些模型"** 这一点直接决定了后续所有判断。

---

## 二、事故现场：一个供应商的模型全部用不了

我的 DSH（DeepSeek Harness，插件化 Agent 框架）在 `settings.yaml` 里配置了多个供应商路由，每个路由一个 Key、一组模型。某一天，供应商 `ren2` 下的所有模型（gpt-5.5、gpt-5.6-luna……）在 GUI 里选择后都报错。

错误形态有两种：

1. **历史日志**：`OpenAI API error (502): {"message":"Upstream service temporarily unavailable"}` 和 `503 Service temporarily unavailable`；
2. **后来**：直接变成 `HTTP 404`（空响应体）。

注意 502/503 和 404 是完全不同的故障语义（后面第五节细讲），这说明问题**在恶化/变化**，而不是单一根因。

---

## 三、排障方法论：分层排查

LLM 调用链是一条长链，每一层都可能出错。我把排查分成四层，逐层排除：

```
配置层 ──▶ 凭证层 ──▶ 协议层 ──▶ 上游层
（settings  （Key 是否   （请求格式/  （真实模型
  是否写对）  有权限）     协议对不对）  服务是否健康）
```

### 第 1 层：配置层

检查 `settings.yaml`：供应商路由、`base_url`、`api`（协议）、模型列表是否自洽。

```yaml
llm-pi-ai:
  providers:
    ren2:
      apiKeyEnv: REN2_API_KEY
      api: openai-responses        # ← 协议
      baseURL: https://relay.example.com/v1
      models:
        - id: gpt-5.5
```

### 第 2 层：凭证层（关键发现）

**直接调 `/v1/models` 看这个 Key 能"看到"什么**：

```powershell
curl -H "Authorization: Bearer sk-xxxx" https://relay.example.com/v1/models
```

发现：**旧 Key 返回 13 个 GPT 模型；换新 Key 后只返回 8 个非 GPT 模型**（deepseek-v4-flash-0731、glm-5.2、gpt-oss-120b、kimi-k2.6……）。

**结论：`/v1/models` 返回的是该 Key 的套餐视图，不是网关的全量模型表。** 我的 `settings.yaml` 里 `ren2` 仍写着旧 Key 的 13 个 GPT 模型，新 Key 根本没有这些模型的权限 → 全部 404。

这解释了为什么"换了 Key 还是用不了"：不是 Key 坏了，而是**新 Key 属于另一个套餐**。

### 第 3 层：协议层（对照实验）

同一个模型 `gpt-5.6-luna`，用不同 Key、同一协议分别调用：

| 组合 | 结果 |
|---|---|
| `ren2` Key + `gpt-5.6-luna` + responses | ❌ 404 |
| `gpt-cli` Key + `gpt-5.6-luna` + responses | ✅ 200（5.4s） |

**一秒钟定位：模型本身没问题，是 Key 权限问题。** 对照实验（控制变量）是这套排障里最锋利的工具。

### 第 4 层：上游层

剩下的故障（502/503、超时、时好时坏）指向网关上游：

- 同一模型 10 分钟内从 200 变 502 再变 200；
- 非流式全部正常（2-5s），但**流式+缓存组合**（`stream: true` + `prompt_cache_key`）经常 40s+ 超时；
- 错误体是 `upstream_error`，说明网关已经把请求转发给上游、上游拒绝了，而不是网关自己的问题。

---

## 四、API 调用原理：一个请求的完整旅程

以 OpenAI 系 SDK 为例，一次模型调用在客户端侧大致是：

### 1. 构造请求

SDK 把 `messages`、`model`、`max_tokens`、`stream` 等参数序列化成 JSON，POST 到 `{base_url}/chat/completions`（或 `/responses`）：

```json
{
  "model": "gpt-5.6-luna",
  "input": [{"role": "user", "content": "hi"}],
  "stream": true,
  "max_output_tokens": 1024,
  "prompt_cache_key": "session-xxx",
  "store": false
}
```

### 2. 认证

`Authorization: Bearer <api_key>`。Key 不只做认证，**在中转站里它同时是"权限凭证"**——决定你能调哪些模型、什么速率、什么套餐。这是中转站与直连最大的心智差异。

### 3. 流式传输（SSE）

Agent 场景几乎总是流式（`stream: true`），因为工具调用循环需要边生成边处理。响应是 **SSE（Server-Sent Events）**：一段段 `data:` 事件，最后 `data: [DONE]`：

```text
event: response.created
data: {"type":"response.created","response":{"id":"resp_...","status":"in_progress",...}}

data: {"type":"response.output_text.delta","delta":"Hello"}

data: [DONE]
```

客户端 SDK 负责把 SSE 流解析回消息对象。**流式排障的坑**：连接挂起 40s 不返回、中途 socket 断开，在非流式下都看不到。

### 4. 重试与超时

我的 Agent 框架对可重试错误（`SERVER`、`TIMEOUT`、`TRANSPORT`、`RATE_LIMIT`、`EMPTY_RESPONSE`）自动指数退避重试（最多 2 次）。所以你在日志里看到的往往不是一次报错，而是"报错 → 重试 → 再报错 → 放弃"的完整序列。

---

## 五、API 协议：OpenAI 生态的三兄弟

中转站最常见的协议适配是这三种（都可能以 `/v1/` 为前缀暴露）：

### 1. Chat Completions（`/v1/chat/completions`）

最经典。请求体是 `messages` 数组，响应是 `choices[].message`：

```json
{"model": "...", "messages": [{"role": "user", "content": "hi"}], "stream": true}
```

### 2. Responses（`/v1/responses`）

OpenAI 新一代协议。请求体用 `input`（可以是字符串或消息数组），响应是 `response` 对象，支持 `reasoning`、`tools` 等一等公民字段：

```json
{"model": "...", "input": [{"role": "user", "content": "hi"}], "stream": true, "prompt_cache_key": "..."}
```

### 3. Messages（`/v1/messages`）

Anthropic 原生协议（`anthropic-messages`），`system` 是顶层字段、`max_tokens` 必填，语义与前两者不同。

### 关键差异点（排障时最容易踩）

| 维度 | Chat Completions | Responses | Messages |
|---|---|---|---|
| 请求字段 | `messages` | `input` | `messages` + 顶层 `system` |
| 输出上限 | `max_tokens` | `max_output_tokens` | `max_tokens`(必填) |
| 缓存 | `prompt_cache_control` | `prompt_cache_key` | `cache_control` |
| 流式事件 | `choices[].delta` | `response.output_text.delta` | `content_block_delta` |

**同一个模型可能同时支持多个协议，但在中转站上不同协议的表现可能不同**（比如有的网关只对 responses 流式做了稳定转发，有的只对 completions 做了缓存）。我的案例里：同一批 GPT 模型，走 `openai-completions` 稳定，走 `openai-responses` 流式就慢/超时——**换协议有时本身就是一种"修复"**。

### 错误码语义（决定排查方向）

| 状态码 | 含义 | 排查方向 |
|---|---|---|
| 400 | 请求参数/业务校验失败 | 检查请求格式、模型要求 |
| 401 | 认证失败 | 检查 Key 是否有效 |
| 403 | 无权限 | 套餐不含该模型/资源 |
| **404** | **模型不存在 或 该 Key 无权访问** | **先查 `/v1/models` 确认 Key 的套餐视图** |
| 429 | 限流/配额耗尽 | 等待或降速 |
| **502/503** | **上游服务不可用** | **网关上游故障，与配置无关** |

---

## 六、其他两个供应商的教训

顺带排查了另外两个供应商，各暴露一类典型问题：

### 1. 模型 ID 过时（404 的另一种成因）

供应商 `claude-ren` 的模型列表里写的是 `claude-haiku-4-5`、`claude-opus-4-5`、`claude-sonnet-4-5`（无日期后缀），**全部 404**；但网关实际可用的是带日期后缀的 ID：

| 配置里（404） | 网关实际可用（200） |
|---|---|
| `claude-haiku-4-5` | `claude-haiku-4-5-20251001` |
| `claude-opus-4-5` | `claude-opus-4-5-20251101` |
| `claude-sonnet-4-5` | `claude-sonnet-4-5-20250929` |

**教训：模型的"别名 ID"会变，网关只认精确 ID。** 配置里的模型列表应该用 `/v1/models` 返回的精确 ID 定期校准。

### 2. 上游时好时坏（502/503 的另一种成因）

供应商 `gpt-cli` 的 9 个模型 7 个稳定 200，但 `gpt-5.6` 和 `gpt-4o-audio-preview` 反复 503。重测多次、换时段后仍是上游问题 —— **这类故障只能等，或者换模型/换协议绕过**。

---

## 七、经验总结：LLM 中转站排障 Checklist

遇到"模型用不了"，按这个顺序查，半小时内基本能定位：

1. **查配置**：`base_url` / 协议 / 模型 ID 是否自洽；
2. **查 Key 的套餐视图**：`curl /v1/models -H "Authorization: Bearer <key>"` —— 看这个 Key 到底能看到哪些模型；
3. **对照实验**：同一模型换 Key、同一 Key 换协议，各测一次，用控制变量切分"模型问题 / Key 问题 / 协议问题"；
4. **分清错误码**：404 先怀疑 Key 权限与模型 ID；502/503 是上游故障，与配置无关；429 是配额；
5. **别只看非流式**：流式请求单独测，注意 SSE 挂起/断流/超时；
6. **校准模型列表**：定期用 `/v1/models` 同步配置里的模型 ID，避免"别名过期"。

---

## 八、中转站的取舍

最后聊聊"要不要用中转站"。我的体感：

**优点**
- 一个 Key、一个域名、一套 OpenAI 兼容协议，接入成本极低；
- 模型切换不改代码，改配置即可；
- 多供应商互备，一家挂了换一家。

**代价**
- **多了一层不确定性**：网关本身、网关到上游的链路，都会引入新的故障模式（本次排查的大部分时间都花在这里）；
- **协议是"兼容"不是"原生"**：高级特性（reasoning、缓存、流式细节）可能被阉割或表现不同；
- **套餐/权限不透明**：Key 能看什么模型、速率多少，只能靠试。

> 一句话：**中转站适合"多模型灵活切换、不想维护多家 SDK"的场景；但排障时请记住——你看到的错误，可能来自网关，也可能来自网关背后的网关。**
