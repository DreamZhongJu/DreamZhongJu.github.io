---
title: 一次 DeepSeek thinking 模式 400 的事故复盘：reasoning_content 回传与自愈重试
date: 2026-08-11 10:00:00
updated: 2026-08-11 12:00:00
categories:
  - 项目复盘
tags:
  - Agent
  - DeepSeek
  - 事故复盘
  - LangGraph
  - Python
description: 我的飞书助手凯伊在生产环境连续报错 "reasoning_content must be passed back to the API"。本文完整复盘根因、修复过程与"旧会话线程"这个隐藏坑，并给出带自愈重试的脱敏代码。
---

## TL;DR

凯伊（我的自托管飞书研究助手）在某天开始在多轮对话里随机报错：

```text
处理失败：Error code: 400 - {'error': {'message': 'The `reasoning_content` in the thinking mode must be passed back to the API.', 'type': 'invalid_request_error', ...}}
```

根因不是模型"抽风"，而是我用的 `deepseek-v4-flash` 是一个带思考（thinking）模式的模型：**它每次回答都会额外返回一段 `reasoning_content`（模型内部思考过程），而 DeepSeek 要求同一段对话的后续请求必须把上一轮的 `reasoning_content` 原样回传，否则直接 400 拒绝。**

修复分三步：把 `reasoning_content` 从响应里存进会话消息、在下一轮请求里原样回传、再给整条链路加一层"遇到这个特定 400 就自动清掉旧会话重试一次"的自愈兜底。本文记录整个过程和踩到的隐藏坑。

## 一、事故现场

凯伊是一个跑在自己服务器上的 LangGraph Agent：飞书里发消息 → webhook 接收 → Agent 决定是否调工具（联网搜索、读论文、查知识库）→ 带着工具结果再问一次模型 → 把最终回答发回飞书。

某天开始，用户问第二句、或者某次工具调用之后，飞书里就回一句：

```text
处理失败：Error code: 400 - {'error': {'message': 'The `reasoning_content` in the thinking mode must be passed back to the API.', 'type': 'invalid_request_error', 'param': None, 'code': 'invalid_request_error'}}
```

注意几个特征：

1. 报错格式是 OpenAI SDK 的 `BadRequestError`（`Error code: 400 - {...}`）；
2. 错误来自 DeepSeek 服务端，不是本地代码抛的异常；
3. 触发场景都是"同一段对话的第二轮及以上"，尤其是**工具调用循环**里。

第一轮提问是好的，第二轮就开始炸。这几乎把原因写在脸上了：**模型要求把上一轮返回的 `reasoning_content` 传回去，而我（当时的代码）没有传。**

## 二、理解 `reasoning_content`

带思考模式的模型（如 DeepSeek 的 reasoner 系列、或默认开启 thinking 的模型）在返回内容时，响应结构长这样（脱敏简化）：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": "这是给用户看的最终回答……",
      "reasoning_content": "这是模型内部逐步推理的过程……"
    }
  }]
}
```

`reasoning_content` 是思考过程，`content` 是最终输出。两者分开存放。

DeepSeek 的接口有一个硬性约定：**在同一个多轮会话里，如果上一轮 assistant 消息带过 `reasoning_content`，那么下一轮请求里的 assistant 消息必须把它原样带上**，否则接口直接返回 400，就是事故里看到的那条消息。

这个设计的原因不难猜：服务端需要把完整的上下文（包括思考过程）喂给模型，才能保证多轮一致性；如果客户端把思考过程丢了，服务端只能拒绝，而不是默默继续。

## 三、为什么"多轮"才炸：LangGraph 工具循环

凯伊的核心是 LangGraph 的 agent 循环：

```text
用户消息
  → agent 节点（调用模型，可能返回 tool_calls + reasoning_content）
  → tools 节点（执行工具，返回 ToolMessage）
  → agent 节点（再次调用模型，需要带上之前所有消息，包括 reasoning_content）
  → …… 直到模型不再要求调工具
  → 返回最终回答
```

关键在第三次箭头：**第二次调用模型时，请求里包含了上一轮 assistant 的完整消息**。如果这条消息里没有 `reasoning_content`，DeepSeek 就报 400。

第一轮提问只发 system + user，自然不炸；一旦进入工具循环或第二次提问，历史里就出现了带 `reasoning_content` 的 assistant 消息，问题立刻暴露。

## 四、根因：两个叠加的坑

### 坑 1：代码没有保存和回传 `reasoning_content`

最初的 `_as_openai_message` 只把 `content` 和 `tool_calls` 转成 OpenAI 格式，`reasoning_content` 被丢弃了：

```python
def _as_openai_message(message: BaseMessage) -> dict[str, Any]:
    if isinstance(message, AIMessage):
        payload = {"role": "assistant", "content": str(message.content or "")}
        if message.tool_calls:
            payload["tool_calls"] = [...]
        return payload
    ...
```

同时，`native_agent_node` 拿到模型响应后，也没有把 `reasoning_content` 存进 LangGraph 的 `AIMessage`：

```python
return {"messages": [AIMessage(content=content, tool_calls=calls)]}
```

于是"响应里有 → 消息里没有 → 下一轮请求里也没有 → 400"。

### 坑 2（隐藏坑）：旧会话线程的存量数据

修好坑 1 之后，我以为没事了，结果**旧会话还是报 400**。

原因：LangGraph 用 SQLite checkpointer 持久化每个用户的会话线程（`assistant:<owner_id>`）。线程里存的是历史消息，而**旧线程里的 assistant 消息是在修复之前写入的，天然没有 `reasoning_content`**。代码虽然修好了，但旧数据没有。

所以哪怕部署了修复，只要用户继续的是修复前就存在的旧线程，下一次请求依然 400。这就是"为什么我改了代码还是报错"。

## 五、修复方案

### 第一步：保存 `reasoning_content`

模型响应里取出来，塞进 LangGraph 消息的 `additional_kwargs`：

```python
additional_kwargs: dict[str, Any] = {}
reasoning = getattr(response, "reasoning_content", None)
if reasoning:
    additional_kwargs["reasoning_content"] = reasoning

return {"messages": [AIMessage(content=content, tool_calls=calls, additional_kwargs=additional_kwargs)]}
```

### 第二步：回传 `reasoning_content`

把消息转成 OpenAI 请求格式时，从 `additional_kwargs` 读回来，原样放进 assistant 消息：

```python
def _as_openai_message(message: BaseMessage) -> dict[str, Any]:
    if isinstance(message, AIMessage):
        payload: dict[str, Any] = {"role": "assistant", "content": str(message.content or "")}
        reasoning = message.additional_kwargs.get("reasoning_content")
        if reasoning:
            payload["reasoning_content"] = reasoning
        if message.tool_calls:
            payload["tool_calls"] = [...]
        return payload
    ...
```

这两步解决了"新会话"的问题。但坑 2（旧线程）还需要第三步。

### 第三步：自愈重试

与其让用户手动清数据库，不如让代码自己处理：捕获这个特定的 `BadRequestError` → 用 LangGraph 自带的 `delete_thread` 清掉该用户的旧线程 → 用干净的新线程重试一次。**只有**错误消息同时包含 `reasoning_content` 和 `must be passed back` 才触发，其他 400（限流、参数错误等）一律原样抛出，避免误伤：

```python
from openai import BadRequestError, OpenAI

def answer(graph, question, context, owner_id) -> str:
    ...
    thread_id = f"assistant:{owner_id}"
    try:
        try:
            result = graph.invoke(
                {"messages": [HumanMessage(content=payload)]},
                {"configurable": {"thread_id": thread_id}},
            )
        except BadRequestError as exc:
            if "reasoning_content" not in str(exc) or "must be passed back" not in str(exc):
                raise
            # 旧线程缺 reasoning_content，清掉后用干净线程重试一次
            cleared = False
            try:
                if memory_store._memory_checkpointer is not None:
                    memory_store._memory_checkpointer.delete_thread(thread_id)
                    cleared = True
            except Exception:
                pass
            if not cleared:
                thread_id = f"{thread_id}:fresh"  # 清不掉就用全新线程兜底
            result = graph.invoke(
                {"messages": [HumanMessage(content=payload)]},
                {"configurable": {"thread_id": thread_id}},
            )
    finally:
        ...
```

注意两个细节：

1. `delete_thread` 是 `SqliteSaver` 提供的公开方法（内部带锁、自动提交），比手写 SQL 删表安全；
2. 重试用的 payload 本来就包含"最近聊天上下文 + 长期记忆"，所以即使线程被清空，模型仍然知道用户刚才在聊什么，不会失忆到无法回答。

## 六、验证

### 单元测试：触发与不误触发

```python
# 用例 1：特定 400 → 清线程 → 重试成功
graph = FakeGraph()  # 第一次调用抛 reasoning_content 400，第二次返回正常回答
out = answer(graph, "继续", "旧上下文", "user-1")
assert graph.calls == 2
assert checkpointer.deleted == ["assistant:user-1"]
assert "已恢复" in out

# 用例 2：其他 400 → 不清理、不重试
graph = FakeGraph2()  # 抛 "rate limit"
with pytest.raises(BadRequestError):
    answer(graph, "hi", "", "user-2")
assert graph.calls == 1
assert checkpointer.deleted == ["assistant:user-1"]  # 没有新增删除
```

两个用例都通过：只对这一个错误自愈，其余错误行为不变。

### 线上部署验证

把修复同步到服务器、备份原文件、重建容器：

```bash
cp assistant/agent/runtime.py assistant/agent/runtime.py.bak-20260811
docker compose up -d --build feishu-assistant
```

重启后看日志，确认飞书长连接恢复：

```text
INFO feishu-assistant: starting Feishu long connection
INFO Lark: connected to wss://msg-frontier.feishu.cn/ws/v2?...
```

再进容器确认修复代码确实在运行镜像里：

```bash
docker exec feishu-assistant grep -n "SELF_HEAL" /app/assistant/agent/runtime.py
```

## 七、经验教训

1. **对接带思考模式的模型，第一件事就是把额外返回字段当一等公民存起来。** `reasoning_content`、`tool_calls`、`usage` 这类字段，只要模型会返回，就要有明确的存取路径，不能只盯着 `content`。
2. **持久化状态和代码版本是耦合的。** 只要会话/状态会落库，改动消息结构时就要问一句："数据库里已经存在的旧数据怎么办？" 这是这次事故里最隐蔽的坑。
3. **生产故障要修"自愈"，而不只是修"新请求"。** 只修新代码，存量用户会继续踩坑；给故障路径加自动恢复，用户甚至感知不到发生了什么。
4. **异常要带可观测信息。** 这次排查快，靠的是飞书回给用户的完整错误原文 + 服务端日志。生产 Agent 的错误信息不要截断到看不出原因。
5. **兜底重试要"窄触发"。** 自愈逻辑只匹配特定错误，绝不 catch-all 重试，否则会把限流、参数错误也吞掉，反而掩盖真实问题。

## 相关代码

完整项目：[github.com/DreamZhongJu/feishu-research-assistant](https://github.com/DreamZhongJu/feishu-research-assistant)

本系列其他文章：[从零搭一个自托管 Agent（系列）](https://dreamzhongju.github.io/agent-series/)
