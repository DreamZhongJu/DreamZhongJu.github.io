---
title: 给 Agent 装上分层记忆：mem0/Letta 式记忆治理实践
date: 2026-08-10 12:00:00
updated: 2026-08-10 18:30:00
categories:
  - 技术实践
tags:
  - 记忆
  - LLM
  - mem0
  - Agent
  - SQLite
description: 从"对话记录"到"治理过的长期事实"：核心/存档两层记忆、add/update/delete 操作、预算与遗忘的完整落地笔记（附脱敏代码）。
---

Agent 的一个老问题是"没有记忆"。我做的飞书研究助手早期接了 Claude-Mem 做对话级记忆，但很快发现它不够：Claude-Mem 存的是原始对话观察（observations），越多越杂，没有分层也没有治理。后来参考 mem0 和 Letta（MemGPT）的思路，把记忆重构成两层治理。这篇文章完整记录落地过程，包括表结构、操作语义、预算遗忘和隐私处理。

## 一、先想清楚：Agent 记忆到底有几种

动手之前，我把"记忆"拆成了三类，职责完全不同：

| 类型 | 内容 | 生命周期 | 典型实现 |
| --- | --- | --- | --- |
| 会话记忆 | 当前对话的上下文 | 短（一次对话） | LangGraph checkpointer / 消息摘要 |
| 对话观察 | 历史对话的原始记录 | 中（可积累可检索） | Claude-Mem（外部服务，可选） |
| 长期事实 | 提炼后的稳定信息（偏好、方向、决定） | 长（治理、可遗忘） | 本地 SQLite + LLM 提取 |

关键决策是：**只有"长期事实"这一层做严格治理**。Claude-Mem 继续当"原始记忆仓库"，本地表存"提炼过的知识"。两边各司其职：

- Claude-Mem：按会话/项目存 observations，语义搜索召回，回答时作为"可能不完整的参考"。
- 本地事实表：模型每次从对话里提炼稳定事实，走 add/update/delete 操作写入，回答时分层加载。

读取时拼接顺序是：**核心记忆 -> 相关存档记忆 -> 对话观察**。

## 二、表结构：两层 + 访问统计

一张 `long_term_memories` 表同时承载两层，用 `is_core` 区分；另外记录访问统计，供遗忘策略使用：

```sql
CREATE TABLE IF NOT EXISTS long_term_memories (
    owner_id          TEXT NOT NULL,        -- 用户范围（脱敏后哈希）
    memory_id         TEXT PRIMARY KEY,     -- UUID
    category          TEXT NOT NULL,        -- 偏好|研究|项目|习惯|决定|身份
    content           TEXT NOT NULL,
    source_message_id TEXT NOT NULL,
    is_core           INTEGER NOT NULL DEFAULT 0,   -- 1=核心记忆
    access_count      INTEGER NOT NULL DEFAULT 0,   -- 被召回次数
    last_accessed_at  TEXT,                 -- 最近一次被召回
    created_at        TEXT NOT NULL,
    updated_at        TEXT NOT NULL
);
CREATE INDEX idx_mem_owner ON long_term_memories(owner_id, updated_at DESC);
CREATE INDEX idx_mem_core  ON long_term_memories(owner_id, is_core, updated_at DESC);
```

SQLite 已存在的库升级时用 `ALTER TABLE ADD COLUMN` 增量加列，老数据不受影响：

```python
def _add_column(con, table, column, ddl):
    existing = {row[1] for row in con.execute(f"PRAGMA table_info({table})")}
    if column not in existing:
        con.execute(f"ALTER TABLE {table} ADD COLUMN {column} {ddl}")
```

## 三、两层记忆：核心 vs 存档

参考 Letta 的 core memory / archival memory 分层：

**核心记忆**：用户长期身份与核心偏好（姓名、研究方向、重要偏好）。每次请求无条件加载，数量严格控制（每用户 8 条），超出自动修剪最旧的。

**存档记忆**：项目、习惯、决定等可检索事实。按相关性召回，每用户 120 条，按最近访问时间清理。

加载逻辑（脱敏）：

```python
def core_memories(owner_id: str) -> str:
    """核心层：总是加载，量小且稳定。"""
    rows = db_query(
        "SELECT category, content FROM long_term_memories "
        "WHERE owner_id = ? AND is_core = 1 ORDER BY updated_at DESC LIMIT ?",
        (owner_id, CORE_LIMIT),
    )
    return "\n".join(f"- {cat}: {content}" for cat, content in rows) or "（暂无核心记忆）"


def relevant_memories(owner_id: str, question: str) -> str:
    """存档层：按相关性召回，并更新访问统计。"""
    rows = db_query(
        "SELECT memory_id, category, content, updated_at FROM long_term_memories "
        "WHERE owner_id = ? AND is_core = 0 ORDER BY updated_at DESC LIMIT 120",
        (owner_id,),
    )
    query_terms = _memory_terms(question)  # 英文词 + 中文 bigram
    ranked = sorted(
        ((len(query_terms & _memory_terms(content)), updated_at, memory_id, category, content)
         for memory_id, category, content, updated_at in rows),
        key=lambda x: (x[0], x[1]), reverse=True,
    )
    selected = [r for r in ranked if r[0] > 0][:MEMORY_LIMIT] or ranked[:3]
    # 召回即更新访问统计
    db_executemany(
        "UPDATE long_term_memories SET last_accessed_at = ?, access_count = access_count + 1 "
        "WHERE owner_id = ? AND memory_id = ?",
        [(now(), owner_id, r[2]) for r in selected],
    )
    return "\n".join(f"- {cat}: {content}" for _, _, _, cat, content in selected)
```

这里有个值得说的细节：中文不做分词，直接用字符 bigram 做召回键，配合 SQLite 全量扫描（数据量小，个人项目完全够用）。没有引入向量库，服务器压力小。

## 四、操作治理：add / update / delete / noop

早期版本用"内容哈希"去重：同样的话不会重复插入，但"喜欢喝咖啡"和"喜欢喝美式咖啡"会变成两条。mem0 的做法是让模型输出显式操作，我照搬了这个思路。

提取节点把"用户已有记忆摘要 + 当前消息"交给模型，模型返回 JSON 操作列表：

```python
def memory_extract_node(state):
    question = state.get("question", "").strip()
    if not question or any(p in question for p in ("不要记住", "别记住", "忘记这条")):
        return {}
    if llm is None:  # 无凭据环境直接跳过
        return {}

    existing = _existing_memory_brief(state["owner_id"], limit=40)
    prompt = (
        "你是长期记忆管理员。根据用户当前消息与已有记忆，决定记忆操作。\n\n"
        f"用户已有记忆：\n{existing}\n\n"
        f"当前消息：{question[:4000]}\n\n"
        "规则：\n"
        "- add：新的长期事实/偏好/决定，且不与已有记忆重复。\n"
        "- update：当前消息补充或修正某条已有记忆，target_id 指向它，content 写合并后的完整内容。\n"
        "- delete：用户明确要求删除或推翻某条已有记忆，target_id 指向它。\n"
        "- noop：没有值得长期记住的内容（临时聊天、新闻、敏感信息）。\n"
        "category 取值：偏好|研究|项目|习惯|决定|身份。\n"
        'is_core=true 只用于用户长期身份与核心偏好。\n'
        '只输出 JSON：{"ops":[{"op":"add|update|delete|noop","target_id":"","category":"偏好","is_core":false,"content":"..."}]}'
    )
    raw = llm.chat.completions.create(model=MODEL, messages=[...], temperature=0).choices[0].message.content
    ops = _parse_ops(raw)
    apply_memory_ops(state["owner_id"], state.get("message_id", ""), ops)
    return {}
```

执行层（脱敏）：

```python
def apply_memory_ops(owner_id, message_id, ops):
    changed = 0
    for op in ops[:6]:  # 每条消息最多 6 个操作，防滥用
        action = op.get("op", "").strip().lower()
        content = re.sub(r"\s+", " ", str(op.get("content", ""))).strip()[:160]
        if content and re.search(r"(?:sk-|api[_ -]?key|密码|口令|token|授权码)", content, re.IGNORECASE):
            continue  # 敏感内容直接丢弃
        category = str(op.get("category", "偏好"))[:16]
        if category not in SAFE_CATEGORIES or (len(content) < 4 and action != "delete"):
            continue
        is_core = 1 if op.get("is_core") in (True, "true", 1, "1") else 0
        target_id = str(op.get("target_id", "")).strip()

        if action == "add" and content:
            insert_memory(owner_id, category, content, message_id, is_core)
            changed += 1
        elif action == "update" and target_id and content:
            changed += update_memory(owner_id, target_id, category, content, is_core, message_id)
        elif action == "delete" and target_id:
            changed += delete_memory(owner_id, target_id)
        elif action == "update" and content and not target_id:
            insert_memory(owner_id, category, content, message_id, is_core)
            changed += 1
    enforce_memory_budget(owner_id)
    return changed
```

效果：同一事实只会有一条记忆；内容变化时原地更新而不是新增；用户说"忘记 XX"时真正删除。

## 五、预算与遗忘

光有操作还不够，记忆必须有限度：

```python
def enforce_memory_budget(owner_id):
    pruned = 0
    # 核心记忆超 8 条：按更新时间（并列时按 rowid）修剪最旧的
    core_ids = db_query(
        "SELECT memory_id FROM long_term_memories WHERE owner_id=? AND is_core=1 "
        "ORDER BY updated_at DESC, rowid DESC LIMIT -1 OFFSET ?",
        (owner_id, CORE_LIMIT),
    )
    # 存档记忆超 120 条：按最近访问时间清理最久没用过的
    keep = db_query(
        "SELECT memory_id FROM long_term_memories WHERE owner_id=? AND is_core=0 "
        "ORDER BY COALESCE(last_accessed_at, created_at) DESC, rowid DESC LIMIT ?",
        (owner_id, ARCHIVE_LIMIT),
    )
    # 删除核心超额部分 / 存档不在保留列表的部分
    ...
    return pruned
```

一个真实的坑：同一批写入的记忆时间戳完全相同，单纯按 `updated_at` 排序会删错条目。加 `rowid` 作为并列排序的次级键后行为才稳定。

## 六、隐私与安全

记忆是敏感数据，落地时做了四件事：

1. **写入前过滤**密钥、密码、授权码等模式：

```python
_SECRET_PATTERNS = [
    re.compile(r"https?://[^\s]*(?:[?&](?:token|code|access_token|api_key)=)[^\s]*", re.I),
    re.compile(r"\bsk-[A-Za-z0-9_-]{12,}\b"),
    re.compile(r"(?i)\b(?:api[_ -]?key|password|authorization)\s*[:：=]\s*\S+"),
]
```

2. **按用户范围隔离**：owner 用哈希存储，不直接暴露飞书 ID。
3. **可删除**：提供单条删除、按用户清空（Web 面板可操作）。
4. **提取保守**：宁可不记，不记错；模型输出非法 JSON 时静默跳过，不写坏数据。

## 七、效果与局限

重构后最直观的变化：

- 记忆不再无限膨胀（有预算自动修剪）。
- 同一事实不会重复（操作语义合并）。
- 用户明确遗忘的信息真的会被删掉。
- 每次请求有访问统计，遗忘策略有据可依。

局限也要说清楚：

- 没有 embedding，冲突判断和相似召回依赖 LLM 判断，成本比向量检索高，规模大了会吃力。
- 提取质量取决于模型；"是否该记住"本身有主观性，目前过滤规则是保守的。
- 记忆与知识库的边界需要持续维护：记忆不是事实来源，不能拿它代替知识库回答。

## 八、给同样在做 Agent 记忆的人

三个建议：

1. **先定好"存什么、谁来存、存多久"，再动手写表结构**。记忆治理的本质不是存储技术，而是信息生命周期管理。
2. **把"操作"作为一等公民**。只做去重不做更新，记忆迟早会变成垃圾场。
3. **给遗忘留一条真正的路**。用户说"忘掉这个"时，删除要比提取更容易做到。

下一篇我会讲 MCP 动态工具接入和评测闭环——工具越来越多之后，怎么让能力可衡量、可回归。
