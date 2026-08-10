---
title: 让 Agent 长出"外挂"并自证能力：MCP 客户端与评测闭环
date: 2026-08-10 14:00:00
updated: 2026-08-10 19:00:00
categories:
  - 技术实践
tags:
  - MCP
  - 评测
  - 可观测性
  - CI
  - Python
description: 用 MCP 让工具即插即用，用可观测性与评测闭环让 Agent 的能力可衡量、可回归。附完整脱敏代码与 CI 配置。
---

Agent 的能力上限，很大程度上取决于工具数量和质量。但每加一个工具就写一遍"连接外部 API + 解析返回 + 注册进框架"的样板代码，很快会变成维护噩梦。这篇文章讲两件事：怎么用 MCP 让工具"即插即用"，以及怎么用可观测性和评测闭环让 Agent 的能力"可衡量、可回归"。

## 一、MCP 是什么

MCP（Model Context Protocol）是一个开放协议，把"外部能力"统一成"工具列表"暴露给 LLM 应用：

- **MCP Server**：能力的提供方。一个 GitHub server 暴露"查 issue、看仓库"的工具；一个文件系统 server 暴露"读写文件"的工具；一个搜索 server 暴露"语义搜索"的工具。
- **MCP Client**：LLM 应用这一侧。连接 server、发现工具、按需调用。

协议的三个核心操作：`initialize`（握手）、`tools/list`（发现工具）、`tools/call`（调用工具）。工具自带名称、描述和输入 JSON Schema，所以客户端可以做到完全动态。

## 二、动态工具注册

我们的 Agent 启动时读取 `mcp_servers.json`，为每个启用的 server 做三件事：

1. **发现**：连接 server，拉取工具列表。
2. **转换**：把 JSON Schema 输入转成 Pydantic 参数模型，包装成 LangChain 工具。
3. **注册**：并入 Agent 的可调用集合，模型下一轮推理就能看到并调用。

### 2.1 配置

```json
{
  "servers": [
    {
      "name": "exa",
      "transport": "streamable_http",
      "url": "https://mcp.example.com/mcp",
      "enabled": true
    },
    {
      "name": "filesystem",
      "transport": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "C:/data"],
      "enabled": true
    }
  ]
}
```

支持两种传输：

- `stdio`：本地子进程（`npx` 启动的 server、Python server 等）。
- `streamable_http`：远程 HTTP(S) 服务。

### 2.2 连接与发现

核心是一个异步上下文管理器，按传输类型选择连接方式：

```python
@asynccontextmanager
async def _open_session(server):
    transport = server.get("transport", "stdio")
    if transport == "streamable_http":
        url = server["url"]
        streams = streamable_http_client(url)
    elif transport == "stdio":
        params = StdioServerParameters(
            command=server["command"],
            args=server.get("args", []) or [],
            env=server.get("env") or None,
        )
        streams = stdio_client(params)
    else:
        raise ValueError(f"unsupported transport: {transport}")

    async with streams as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            yield session
```

发现工具并转成 LangChain 工具：

```python
async def _discover_async(server):
    async with _open_session(server) as session:
        result = await session.list_tools()
        return [
            {"name": t.name, "description": t.description or "",
             "input_schema": t.input_schema or {}}
            for t in result.tools
        ]


def make_tool(server, info):
    """把一个 MCP 工具包装成 LangChain 工具。"""
    tool_name = info["name"]
    args_model = _json_schema_to_pydantic(tool_name, info.get("input_schema", {}))

    def invoke_func(_server=server, _tool=tool_name, **kwargs):
        arguments = {k: v for k, v in kwargs.items() if v is not None}
        return _call_sync(_server, _tool, arguments)

    return StructuredTool.from_function(
        name=tool_name,
        description=info.get("description", "") or f"MCP tool: {tool_name}",
        func=invoke_func,
        args_schema=args_model,
    )
```

### 2.3 JSON Schema -> Pydantic

工具的输入 schema 是 JSON Schema，LangChain 需要 Pydantic 模型。写了一个轻量转换器，覆盖常用类型：

```python
_TYPE_MAP = {"string": str, "integer": int, "number": float,
             "boolean": bool, "array": list, "object": dict}

def _json_schema_to_pydantic(name, schema):
    fields = {}
    properties = schema.get("properties", {}) or {}
    required = set(schema.get("required", []) or [])
    for prop_name, prop in properties.items():
        if not isinstance(prop, dict):
            continue
        field_type = _TYPE_MAP.get(prop.get("type", "string"), str)
        if prop_name in required:
            fields[prop_name] = (field_type, Field(description=str(prop.get("description", ""))))
        else:
            fields[prop_name] = (field_type, Field(default=None, description=str(prop.get("description", ""))))
    if not fields:
        fields["_empty"] = (str, Field(default=""))
    return create_model(f"{name}_args", **fields)
```

### 2.4 调用

```python
async def _call_async(server, tool_name, arguments):
    async with _open_session(server) as session:
        result = await session.call_tool(tool_name, arguments)
        parts = [getattr(item, "text", None) or str(item) for item in result.content]
        payload = "\n".join(p for p in parts if p)
        if getattr(result, "isError", False):
            raise RuntimeError(payload or f"MCP tool {tool_name} returned an error")
        return payload or "(空结果)"


def _call_sync(server, tool_name, arguments):
    return asyncio.run(_call_async(server, tool_name, arguments))
```

一个务实的取舍：目前每次工具调用都新建连接（`asyncio.run` 包一层）。个人项目可以接受；如果要做成常驻服务，应该维护连接池，避免子进程反复启停。

## 三、容错与配置校验

工具变多之后，配置错误会成为最常见的问题。我们在加载时逐条校验，无效 server 跳过并给出明确原因，而不是静默失败：

```python
def validate_server(server):
    errors = []
    if not str(server.get("name", "")).strip():
        errors.append("缺少 name")
    transport = server.get("transport", "stdio")
    if transport not in {"stdio", "streamable_http"}:
        errors.append(f"不支持的 transport: {transport}")
    if transport == "stdio" and not str(server.get("command", "")).strip():
        errors.append("stdio server 缺少 command")
    if transport == "streamable_http" and not str(server.get("url", "")).strip():
        errors.append("streamable_http server 缺少 url")
    return errors
```

任何 server 连接失败、工具转换失败，都只影响自己，不影响内置工具。

## 四、可观测性：让每次请求可回溯

工具越多，问题定位越难。配套做了请求级可观测性：每次请求记录问题（脱敏）、工具调用链、token、耗时、状态和错误类型，存 SQLite：

```sql
CREATE TABLE request_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    request_id TEXT NOT NULL,
    owner_hash TEXT NOT NULL,          -- 用户哈希，不存明文 ID
    question TEXT NOT NULL,            -- 脱敏、截断
    context_len INTEGER NOT NULL,
    tool_sequence TEXT NOT NULL,       -- JSON 数组，如 ["web_search","read_webpage"]
    answer TEXT NOT NULL,              -- 脱敏、截断
    prompt_tokens INTEGER NOT NULL,
    completion_tokens INTEGER NOT NULL,
    total_tokens INTEGER NOT NULL,
    latency_ms INTEGER NOT NULL,
    status TEXT NOT NULL,              -- ok | error
    error_type TEXT NOT NULL DEFAULT '',
    created_at TEXT NOT NULL
);
```

埋点位置在 Agent 的入口函数：成功和异常都记录，观测层自己失败也不会拖垮主流程（fail-open）：

```python
def answer(graph, question, context, owner_id, chat_id=""):
    ...
    started = time.perf_counter()
    try:
        result = graph.invoke(...)
        tool_sequence = extract_tool_sequence(result)
        answer_text = extract_final_answer(result)
        obs.log_request(
            request_id=request_id, owner_id=owner_id, chat_id=chat_id,
            question=question, context_len=len(payload),
            tool_sequence=tool_sequence, answer=answer_text,
            prompt_tokens=_tool_context.prompt_tokens,
            completion_tokens=_tool_context.completion_tokens,
            latency_ms=int((time.perf_counter() - started) * 1000),
            status="ok",
        )
        return answer_text
    except Exception as exc:
        obs.log_request(..., status="error", error_type=type(exc).__name__)
        raise
```

Web 面板（Flask，内网访问）提供四个页面：

- **仪表盘**：请求量、成功率、今日失败、平均耗时、Token 消耗、估算成本、近 7 天趋势、工具调用排行（含失败数）、最近失败列表。
- **请求日志**：按状态/工具筛选、分页、单条详情（工具链 + 完整问答）。
- **记忆管理**：按用户查看/删除核心与存档记忆。
- **运行状态**：模型、工具数、数据库路径等配置概览。

成本估算按 token 计算（价格可配置）：

```python
estimated_cost_usd = (
    prompt_tokens / 1_000_000 * INPUT_PRICE_PER_M
    + completion_tokens / 1_000_000 * OUTPUT_PRICE_PER_M
)
```

这些数据让"Agent 今天表现如何"从感觉变成了数字。

## 五、评测闭环：离线评测 + 真实请求回放 + CI

可观测性解决"发生了什么"，评测解决"做得好不好"。我们搭了三层：

### 5.1 离线评测

基于项目文档语料构建知识问答集（含负例——问"资料里没有答案"的问题，验证不编造），加工具路由用例，DeepSeek 作为裁判模型打分：

```python
def judge_answer(client, model, question, context, answer):
    prompt = (
        "请评估一次知识问答。\n"
        f"问题：{question}\n参考资料：{context[:2000]}\n模型回答：{answer}\n\n"
        '输出 JSON：{"faithfulness": 0到1, "relevancy": 0到1}'
    )
    out = chat(client, model, [{"role": "system", "content": "只输出 JSON。"},
                               {"role": "user", "content": prompt}], json_mode=True)
    return json.loads(out)
```

### 5.2 真实请求回放

把请求日志里的真实问答转成评测样本，再打分（相关性、完整性），把生产使用变成回归信号：

```python
def main():
    cases = load_replay_cases()  # 从 request_logs 读 status=ok 的记录
    for case in cases:
        score = judge(client, MODEL, case["question"], case["answer"])
        ...
    # 输出 replay_report.md / replay_result.json
```

### 5.3 CI

GitHub Actions 在无凭据环境跑全部单测（这要求模块在无密钥时可导入，详见架构篇），配置 `DEEPSEEK_API_KEY` secret 后追加离线评测 job：

```yaml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - name: Run tests
        env:
          DEEPSEEK_API_KEY: ""
          LARK_APP_ID: ""
          LARK_APP_SECRET: ""
          TOKEN_ENCRYPTION_KEY: ""
        run: python -m unittest discover -s tests -v

  offline-eval:
    runs-on: ubuntu-latest
    needs: test
    if: ${{ secrets.DEEPSEEK_API_KEY != '' }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - run: pip install -r requirements.txt
      - name: Offline evaluation
        env:
          DEEPSEEK_API_KEY: ${{ secrets.DEEPSEEK_API_KEY }}
        run: python evaluation/run_eval.py
```

### 5.4 当前指标

自建评测集结果：

| 维度 | 指标 | 结果 |
| --- | --- | --- |
| 知识问答 | 平均忠实度（faithfulness） | 1.0 |
| 知识问答 | 上下文命中率 hit@3 | 1.0 |
| 知识问答 | 负例不编造率 | 1.0 |
| 工具路由 | Top-1 准确率 | 1.0 |
| 真实请求回放 | 平均相关性 / 完整性 | 1.0 / 1.0 |

需要诚实说明：自建评测集有同源偏差（语料就是项目自己的文档），满分不代表通用能力，它更像"回归测试"而不是"能力基准"。真正的能力评估需要外部数据集交叉验证，这是后续要做的事。

## 六、小结

这一套做下来，Agent 的工程闭环基本完整：MCP 让能力可扩展，可观测性让行为可回溯，评测让质量可回归。推荐按这个顺序投入：

1. **先可观测性**：不知道哪里坏了之前，加再多功能都是盲人摸象。
2. **再评测**：哪怕是最朴素的 LLM-as-judge，也比没有强。
3. **最后才是更多工具**：工具越多，越需要前两者兜底。

代码全部脱敏后放在 GitHub，项目本身还有一篇架构篇和一篇记忆治理篇，欢迎交流。
