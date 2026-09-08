---
title: Kairós Wiki 导出：一个不需要 LLM 也能跑的知识沉淀层
date: 2026-09-08 23:30:00
updated: 2026-09-08 23:30:00
categories:
  - 项目复盘
tags:
  - Kairós
  - 知识图谱
  - 工程实践
  - Python
  - Flask
description: 给 Kairós 加一个"可重建的 wiki 投影"：从群聊窗口里筛出高价值消息，增量归档成带 provenance 的 Markdown 页面，checkpoint 防重复。全程不调 LLM，抽取失败也不受影响——顺带记一个"没有消息 id 时怎么做增量 checkpoint"的坑。
---

## TL;DR

给 Kairós 的知识图谱加了一层"wiki 投影"：群聊消息经过一个零成本的高价值过滤器后，被增量编译成带 frontmatter（`source_refs`/`change_log`/`confidence`）的 Markdown 页面，用 SQLite 做 checkpoint 防止重复处理。整个链路**默认完全不调 LLM**——用一个确定性的内置 compiler 把消息原样摘录归档，为以后接 LLM 摘要留好接口。过程中踩了一个坑：上游消息没有稳定 id，按列表下标生成的 id 会在不同请求之间撞车，导致 checkpoint 把新消息永久跳过。

代码：[kairos/knowledge/wiki.py](https://github.com/DreamZhongJu/kairos-intel/blob/main/kairos/knowledge/wiki.py)

## 为什么要做这个

Kairós 的知识图谱（Neo4j + SQLite 双写）解决的是"结构化检索"：实体、关系、多跳查询。但群友经常要的其实是"这件事当时到底是怎么定的"——一段完整的、可读的、带来源的记录，而不是拆散成三元组的 `(小明, 决定, SQLite)`。这正是 wiki 类工具（Notion / Obsidian / 各种 WikiMind 项目）擅长的：**页面是一等公民，图谱关系是附加信息**。

所以 wiki 层不重新发明存储，而是做成图谱之外的一个"可重建投影"——同样的消息窗口，图谱抽取一份结构化关系，wiki 编译一份可读页面，两条流水线互不依赖，一个挂了不影响另一个。这个设计决定在这次接入时被意外验证了：接入当天 DeepSeek 账户余额刚好用完，图谱抽取全线 402，但 wiki 导出完全没受影响，因为它压根不走 LLM。

## 设计：确定性优先，LLM 是可选增强

`WikiCompiler` 的核心签名很朴素：

```python
Compiler = Callable[[list[dict[str, Any]], str, str], dict[str, Any]]

class WikiCompiler:
    def __init__(self, root, checkpoint_db=None, compiler: Compiler | None = None, ...):
        self.compiler = compiler or _default_compile
```

`compiler` 是一个可插拔的策略：接收消息列表 + domain + page_type，返回 `{title, content, summary, confidence, tags}`。默认实现 `_default_compile` 不调用任何外部服务，只是把消息原样摘录、拼成 Markdown：

```python
def _default_compile(messages, domain, page_type):
    title = str(messages[0].get("title") or f"{domain} 增量知识")
    lines = [f"# {title}", "", "## 来源消息", ""]
    for message in messages:
        stamp = str(message.get("time") or "")
        sender = str(message.get("nickname") or message.get("user_id") or "未知")
        lines.append(f"- [{stamp}] {sender}: {_text(message)}")
    return {"title": title, "content": "\n".join(lines), "confidence": "high", ...}
```

它不发明事实，只是"摘录 + 归档"——但已经比原始聊天记录有价值得多：过滤掉了闲聊，按主题分类归档，还带增量合并。以后账户充值了，只要把 `compiler` 换成一个调 LLM 做摘要/结构化的函数，`WikiCompiler` 本身一行都不用改。这是典型的"能力分层"：确定性规则打底，LLM 是可插拔的增强，而不是硬依赖。

## 高价值过滤：先用规则挡掉 90% 的噪声

群聊里大部分消息是"收到""哈哈""早安"，不值得进 wiki。在调用 compiler 之前先用一个零成本的正则+关键词过滤器筛一遍：

```python
_HIGH_VALUE_TERMS = ("决定", "决策", "结论", "计划", "里程碑", "截止", "问题", "bug", "修复", "方案", "风险", ...)
_LOW_VALUE_RE = re.compile(r"^(?:嗯+|哦+|好+|哈哈+|收到|ok|好的|谢谢|晚安|早安|\.+|!+|\?+)$", re.IGNORECASE)

def is_high_value(message):
    text = _text(message)
    if not text or _LOW_VALUE_RE.fullmatch(text) or len(text) < 12:
        return False
    if any(term in text.lower() for term in _HIGH_VALUE_TERMS):
        return True
    return len(text) >= 80 or bool(re.search(r"https?://|\b\d{4}[-/]\d{1,2}", text))
```

实测一批 35 条真实群聊导出，只有 1 条命中——大部分群聊确实是纯闲聊，这个过滤器基本符合预期：宁可漏掉一些边界情况，也不要把噪声灌进 wiki（宁可"假阴性"多一点，也不要"假阳性"污染页面质量，这个取舍在信息密度低的群聊场景下是对的）。

## 增量合并与 checkpoint：page identity 是标题

一个 wiki 页面不是一次性生成就完事的，同一个话题会随着聊天持续被补充。`WikiCompiler.compile_batch(scope, messages, domain, page_type)` 的行为：

1. 用 SQLite 记录每个 `scope`（比如 `qq:群号`）已处理过的消息 id 集合；
2. 新一批消息里，已处理过的 id 会被跳过（除非 `force=True`）；
3. 全部跳过则返回 `status: "noop"`，不产生任何写盘；
4. 新页面写入时，如果同名（同 title）页面已存在，会把 `source_refs` 去重合并、`change_log` 追加一条、正文拼接在旧内容后面——是真正的"增量归档"而不是覆盖。

```yaml
---
title: "kairos-wiki 增量知识"
source_refs: ["m002", "m004", "m010"]
change_log:
  - "2026-09-08T14:53:29 ingest 2 messages"
  - "2026-09-08T14:53:29 ingest 1 messages"
---
```

这里有个隐含约束：**页面身份是靠 title 字符串识别的**，不是靠某个显式 id。调试时我踩过一次——同一个 scope 里，如果后续消息批次传了一个和默认标题对不上的自定义 `title`，会被当成一个全新页面而不是合并进已有页面。不算 bug，但用的时候要注意：同一 scope 内不传自定义 title（让它按 domain 推导默认标题），或者自己保证 title 严格一致。

## 真正的坑：没有消息 id 时，checkpoint 怎么办

`WikiCompiler` 依赖每条消息有一个稳定的 id 来做 checkpoint 去重。但接给 Koishi 采集端的消息窗口长这样：

```json
{"user_id": "111", "nickname": "小明", "time": "2026-09-08T10:00:00", "text": "..."}
```

没有 `message_id`，也没有 `id`。wiki 模块的 fallback 是按列表下标生成：`f"batch-item-{index}"`。这在单元测试里（同一个 Python 对象、同一个列表）没问题，但接到真实 HTTP 接口后就炸了：**每次 POST 请求，`messages` 列表的下标都从 0 开始**。第一批消息处理完，checkpoint 里记了 `batch-item-0`、`batch-item-1`；下一批完全不同的消息，只要长度一样，下标还是从 0 开始——checkpoint 一看"这些 id 都处理过了"，直接把新消息全部当成已处理，永久跳过，wiki 再也不更新。

修法是不依赖调用方传 id，而是从消息内容自己派生一个稳定 id：

```python
def _wiki_message_id(channel_id: str, item: dict) -> str:
    existing = str(item.get("message_id") or item.get("id") or "").strip()
    if existing:
        return existing
    raw = f"{channel_id}|{item.get('user_id', '')}|{item.get('time', '')}|{_msg_text(item)}"
    return hashlib.sha1(raw.encode("utf-8")).hexdigest()[:16]
```

`channel_id + user_id + time + text` 的组合在实践中足够唯一（同一个人同一秒发两条一模一样的消息的概率可以忽略），而且是内容确定性的——同一条消息不管过多久重新计算，id 都一样，这正是 checkpoint 去重需要的性质。这类"调用方没给稳定标识符"的场景在接第三方数据源时很常见，思路都是一样的：**用内容的哈希代替调用方本该提供但没提供的 id**，而不是退化成位置索引。

## 接口

```
GET /api/wiki/pages          # 列出所有已编译页面：path/title/type/confidence/updated
GET /api/wiki/page?path=...  # 读取一个页面的原始 Markdown
```

`read_page` 复用了和 `_write_page` 一致的根目录逃逸检查（`target.relative_to(root_path)` 失败就返回 404），`../../../etc/passwd` 这种路径穿越尝试直接被挡在文件系统操作之前。

## 小结

这次接入的核心不是"用 LLM 生成漂亮的 wiki 摘要"——那个后面账户充值了再做。核心是先把**骨架**搭对：过滤、增量、checkpoint、provenance、可插拔的 compiler 接口。骨架对了，LLM 什么时候接进来、用什么模型，都只是换一个函数参数的事。
