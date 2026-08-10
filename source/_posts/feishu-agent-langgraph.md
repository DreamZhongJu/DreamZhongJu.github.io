---
title: 自托管飞书研究助手：用 LangGraph 搭一个多工具 Agent
date: 2026-08-10 10:00:00
updated: 2026-08-10 18:00:00
categories:
  - 项目复盘
tags:
  - Agent
  - LangGraph
  - 飞书
  - Python
description: 从分层架构、LangGraph 工具调用循环、ToolResult 契约到失败降级，完整复盘一个自托管飞书研究助手的 Agent 工程实践（附脱敏代码）。
---

过去几个月我做了一个自托管的飞书研究助手：你在飞书里发一句"最近机器翻译有什么值得关注的进展？"，它会自动检索网页和论文、综合可信来源、给出带来源的回答；你发一个 PDF 说"整理一下"，它会解析、总结，并在你明确同意后才归档。这篇文章完整复盘它的 Agent 工程部分——为什么用 LangGraph、工具层怎么设计、事件链路怎么处理、踩了哪些坑，以及关键的脱敏代码。

## 一、为什么做这个项目

市面上有很多"AI 助手"，但大多数是 SaaS，数据经过别人服务器。我的需求比较具体：

1. **私有**：模型密钥、飞书授权、记忆、日志全部由我自己持有。
2. **长在飞书里**：我的日常沟通和研究资料都在飞书（文档、知识库、日程），助手直接出现在对话里，比打开一个网页顺手得多。
3. **可解释**：助手做了什么（搜了什么、读了什么、结论来自哪）要能被看到，而不是黑盒。
4. **能写**：不只是问答，还要能在明确指令下创建云文档、归档知识库。

于是有了四个设计目标：把"对话、检索、阅读、整理、归档"连成一条可解释的工作流；由 LLM 自主决定工具而不是关键词路由；私有数据可控；失败可见、可恢复。

## 二、总体架构

代码按依赖方向从上到下分六层，低层模块不能导入高层：

```text
飞书事件
  -> Channel Adapter（验签、解析、去重、回复）
  -> Application Service（组装上下文、权限、任务）
  -> LangGraph Agent（模型决策、工具调用循环、结果整合）
  -> Tool Layer（网络、论文、文档/知识库、日程、文件、记忆、MCP）
  -> Infrastructure（DeepSeek、飞书 OpenAPI、SQLite、HTTP）
```

各层职责：

| 模块 | 职责 | 不负责什么 |
| --- | --- | --- |
| `channels` | 飞书事件验签、消息解析、幂等去重、发送回复 | Agent 决策、业务检索 |
| `application` | 组装会话上下文、权限、附件上下文 | 直接调底层 HTTP、自行决定工具 |
| `agent` | LangGraph 状态、LLM 决策、工具循环、汇总答复 | 飞书协议细节、数据库 SQL |
| `tools` | 外部能力封装为可调用工具 | 对话人格、最终回答 |
| `memory` | 核心/存档两层记忆治理 | 代替知识库作为事实来源 |
| `observability` | 请求日志、统计、面板数据 | 用户意图判断 |
| `infrastructure` | 模型、HTTP、SQLite、配置、日志 | 业务逻辑 |

依赖方向只允许从上到下，比如网页搜索工具不知道飞书消息格式，飞书适配器也不能拼接模型提示词。这是整个项目能长期维护的根基。

## 三、LangGraph 工作流

Agent 核心是一个很简单的图：

```text
START -> agent -> tools -> agent -> ... -> END
```

`agent` 节点把用户问题、会话摘要、相关长期记忆和工具定义交给模型，模型以原生 tool calling 决定调用哪些工具；`tools` 节点（`ToolNode`）执行并回填结果，再回到 `agent` 继续推理，直到模型不再请求工具。

构建图的代码（脱敏简化）：

```python
def build_graph():
    # 内置工具 + 动态加载的 MCP 工具（详见另一篇文章）
    active_tools = list(NATIVE_TOOLS) + mcp_client.load_mcp_tools()
    active_openai_tools = [convert_to_openai_tool(t) for t in active_tools]

    graph = StateGraph(MessagesState)
    graph.add_node("agent", native_agent_node)
    graph.add_node("tools", ToolNode(active_tools, handle_tool_errors=True))
    graph.add_edge(START, "agent")
    graph.add_conditional_edges("agent", tools_condition, {"tools": "tools", "__end__": END})
    graph.add_edge("tools", "agent")
    return graph.compile(checkpointer=sqlite_checkpointer)
```

Agent 节点每次调用模型，并把 token 用量累计到请求级统计（供可观测性使用）：

```python
def native_agent_node(state):
    if llm is None:  # 无凭据时保持可导入，CI/测试友好
        return {"messages": [AIMessage(content="模型未配置：缺少 DEEPSEEK_API_KEY。")]}

    history = list(state.get("messages", []))
    tool_turns = sum(1 for m in history if isinstance(m, ToolMessage))
    completion = llm.chat.completions.create(
        model=MODEL,
        messages=[{"role": "system", "content": SYSTEM_PROMPT}]
        + [_as_openai_message(m) for m in history],
        # 最多 3 轮工具调用后强制结束，防止循环
        tools=ACTIVE_OPENAI_TOOLS if tool_turns < 3 else None,
        tool_choice="auto" if tool_turns < 3 else "none",
        temperature=0.2,
    )
    response = completion.choices[0].message
    usage = getattr(completion, "usage", None)
    if usage:  # 记录 token，供仪表盘统计成本
        _tool_context.prompt_tokens += usage.prompt_tokens or 0
        _tool_context.completion_tokens += usage.completion_tokens or 0

    calls = []
    for call in response.tool_calls or []:
        try:
            args = json.loads(call.function.arguments or "{}")
        except json.JSONDecodeError:
            args = {}
        calls.append({"name": call.function.name, "args": args, "id": call.id, "type": "tool_call"})

    content = response.content or ""
    if not calls:
        calls = _dsml_tool_calls(content)  # 兼容模型的文本工具调用
    return {"messages": [AIMessage(content=content, tool_calls=calls)]}
```

## 四、工具层设计

### 4.1 ToolResult 契约

所有工具统一返回结构化结果，即使失败也要有明确的 `status`、`error_code` 和 `retryable`，Agent 据此决定重试、换工具还是向用户说明：

```python
@dataclass
class ToolResult:
    tool_name: str
    status: str          # ok | error
    data: str            # 清洗、截断后的内容
    sources: list[str]   # 来源 URL / 文档标识
    error_code: str = ""
    retryable: bool = False
```

### 4.2 工具清单

当前注册 20 个原生工具：

| 分组 | 工具 | 说明 |
| --- | --- | --- |
| 搜索阅读 | `web_search` / `read_webpage` / `semantic_web_search` | 多来源搜索与正文读取 |
| 论文 | `paper_lookup` / `huggingface_papers` | arXiv、OpenAlex、Semantic Scholar、HF Papers |
| 飞书域 | `read_feishu_document` / `knowledge_search` / `today_schedule` | 文档、知识库、日程读取 |
| 写操作 | `save_cloud_document` / `archive_to_knowledge_base` / `knowledge_save` | 需显式指令 |
| 归档 | `preview_cloud_archive` | 批量归档先预览确认 |
| 平台 | `github_research` / `x_search` / `reddit_search` / `bilibili_search` / `youtube_video_details` | GitHub 与社交媒体 |
| 记忆 | `memory_search` | 用户范围长期记忆 |
| 其他 | `daily_report` / `agent_reach_health` | 日报与健康检查 |

工具用 `langchain_core.tools` 的装饰器注册，例如：

```python
@tool("web_search")
def native_web_search(query: str) -> str:
    """Search the public web for current information."""
    results = search_engine(query)  # 内部实现，脱敏
    return format_results(results)  # 标题/摘要/URL/时间
```

### 4.3 写操作的安全设计

创建文档、归档知识库、创建日程属于"高风险写工具"，遵循两个原则：

1. **只有明确指令才执行**：模型系统提示里写死约束，工具本身不猜测用户意图。
2. **批量操作先预览确认**：批量归档先返回分类预览和确认码，用户回复确认码后才真正写入。

## 五、事件链路与幂等

飞书 WebSocket 可能重推事件，消息处理前先写 `handled_messages` 表做幂等去重：

```python
def claim_message(message_id: str) -> bool:
    with sqlite3.connect(DB_PATH) as con:
        try:
            con.execute("INSERT INTO handled_messages(message_id, handled_at) VALUES (?, ?)",
                        (message_id, now()))
            return True
        except sqlite3.IntegrityError:
            return False  # 重复事件，直接丢弃
```

事件处理是异步的（`threading.Thread` 启动后台任务），这样飞书不会因为 LLM 请求慢而超时重试；写操作失败也会尝试回复用户而不是静默吞掉：

```python
def process_event(data):
    ...
    try:
        result = route_and_answer(...)   # 各种指令分支
        reply(message_id, result)
    except Exception as exc:
        LOG.exception("request failed")
        reply(message_id, f"处理失败：{exc}")
```

## 六、兼容模型的"文本工具调用"

这是踩坑最多的地方。DeepSeek 某些版本在压力下会把工具调用写成 DSML 文本而不是结构化 JSON：

```xml
<|DSML|> <|tool_calls|> <|invoke name="web_search">
  <|parameter name="query" string="true">NLP news</|parameter>
</|invoke>
```

我们加了一个解析层，把这种文本转成真正的工具调用：

```python
def _dsml_tool_calls(content: str) -> list[dict]:
    if "DSML" not in content:
        return []
    names = {t.name for t in ACTIVE_TOOLS}
    calls = []
    for match in re.finditer(
        r'invoke\s+name="([A-Za-z0-9_]+)"(.*?)(?=</[^>]*invoke>|\Z)',
        content, re.DOTALL,
    ):
        name, body = match.group(1), match.group(2)
        if name not in names:
            continue
        args = {}
        for param in re.finditer(r'parameter\s+name="([A-Za-z0-9_]+)"[^>]*>(.*?)</[^>]*parameter>', body, re.DOTALL):
            value = re.sub(r"<[^>]+>", "", param.group(2)).strip()
            if value:
                args[param.group(1)] = value
        calls.append({"name": name, "args": args, "id": f"dsml_{secrets.token_hex(6)}", "type": "tool_call"})
    return calls
```

如果模型输出的是文本但解析不出任何合法调用，就明确告诉用户"这项操作暂时没有可用的工具配置"，**绝不把内部调用内容发到聊天里**。

## 七、可靠性与降级策略

| 场景 | 策略 |
| --- | --- |
| 单个搜索源失败 | 记录失败，尝试备用来源，其他来源正常时继续 |
| 所有联网来源失败 | 明确说"本次未能验证实时信息"，已有知识必须标注非实时性 |
| 模型超时/失败 | 有限重试一次，仍失败返回简短提示 |
| 飞书回复失败 | 保留待发送结果，允许安全重试 |
| 工具连续调用 | 最大轮数 + 总超时 + 重复调用检测 |
| 写工具失败 | 不自动无限重试，返回目标、原因和可重试步骤 |

## 八、效果与局限

这个项目让"研究助手"从概念变成了每天在用的工具。工程上的收获是：分层让每个模块可独立测试；失败可见让线上问题可定位；工具可扩展让能力持续增长。

局限同样清楚：单机自托管没有多租户和横向扩展；检索质量依赖外部接口；Agent 的深度（多步规划、反思）还比较浅。

## 九、给同样在做 Agent 的人

三条最实在的建议：

1. **先把"失败可见"做进去，再谈功能丰富**。一个会诚实说"我没查到"的助手，比一个会编造的助手可靠得多。
2. **不要让关键词路由决定工具**。规则看起来简单，但会随着工具增多变成维护噩梦；LLM 决策 + 工具描述写清楚，长期更省事。
3. **无凭据环境要能跑测试**。模块顶层别直接创建客户端，否则 CI 和新人 onboarding 都会卡在环境配置上。

后续我还会写两篇：一篇讲记忆分层治理（mem0/Letta 思路落地），一篇讲 MCP 动态工具接入与评测闭环，欢迎继续关注。
