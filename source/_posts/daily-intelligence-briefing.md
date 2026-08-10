---
title: 自托管每日情报日报：从"看不过来"到"每天一条推送到飞书"
date: 2026-08-11 10:00:00
updated: 2026-08-11 10:00:00
categories:
  - 项目复盘
tags:
  - 自动化
  - RSS
  - arXiv
  - DeepSeek
  - 飞书
description: 一个自托管每日情报日报的完整复盘：需求定义、开源选型、一步步实现（抓取/论文/生成/推送/去重/部署），以及"怎么把别人的开源项目拿来当自己的工具"的方法论。
---

每天早上打开 RSS 阅读器，未读数几百上千；arXiv 上 NLP 方向每天几十篇新论文，根本刷不完；关注的 AI 基础设施公司（Modal、Together、Neon、PostHog……）什么时候发了新动态，只能靠偶尔刷 X 碰运气。信息不是不够，是**太多且分散**。

于是我做了个自托管的"每日情报日报"：每天定时把社会新闻、科技动态、开源社区、NLP/机器翻译论文、AI 基础设施公司动态聚合起来，用大模型整理成一份结构化中文日报，推送到飞书，全程数据自己持有。这篇文章完整记录它怎么一步步做出来的——包括我怎么挑开源项目、怎么把别人的轮子变成自己的工具。

## 一、需求定义：先写清楚"要什么"

动手前我列了七条需求：

1. **每天一份**：早晨 9 点前后生成。
2. **覆盖面**：国际社会新闻、科技动态、开源社区、NLP/机器翻译论文、AI 基础设施公司。
3. **中文**：原文大多是英文，要用大模型整理成中文摘要。
4. **可追溯**：每条都有来源链接，能点回去看原文。
5. **私有**：配置、状态、历史报告都自己持有。
6. **低成本**：免费数据源 + 便宜的 API（DeepSeek）。
7. **可配置**：研究方向、检索词、公司名单都能改，不用改代码。

想清楚"不要什么"同样重要：不做网页、不做多用户、不做全文翻译、不做实时推送——它就是一个"每天早上帮我扫一遍信息"的定时任务。

## 二、调研与选型：哪些轮子可以拿来用

这一节是重点：**一个个人项目最划算的路径，是站在成熟开源项目肩上**。我花了两天调研，结论如下。

### 2.1 为什么不直接用一个现成产品

候选有 Google News 邮件订阅、Feedly、Inoreader、以及 Dify/FastGPT/RSSHub 这类开源平台。排除理由：

- 邮件订阅：无法用 LLM 做中文结构化整理，也没法按我的检索词精确切分。
- Feedly/Inoreader：数据在别人服务器上，免费版限制多，不支持自定义"公司动态"这类结构化源。
- Dify/FastGPT/RSSHub：功能强但**重**——要跑数据库、向量库、前端，个人定时任务用不上 80% 的功能。

结论：写一个几百行的 Python 脚本，把所有开源能力"粘"起来，是最轻的路径。

### 2.2 最终选用的开源项目（附链接）

| 开源项目 | 用途 | 链接 |
| --- | --- | --- |
| feedparser | RSS/Atom 解析，处理各种不规范的 Feed | https://github.com/kurtmckee/feedparser |
| httpx | 异步 HTTP 客户端，并发抓取多个源 | https://github.com/encode/httpx |
| openai-python | OpenAI 兼容接口，用来调 DeepSeek | https://github.com/openai/openai-python |
| python-dotenv | 从 .env 读配置，密钥不入库 | https://github.com/theskumar/python-dotenv |
| arXiv API | 官方论文检索接口 | https://info.arxiv.org/help/api/index.html |
| Google News RSS | 按检索词聚合新闻 | https://news.google.com/rss |
| 飞书自定义机器人 | Webhook 推送消息卡片 | https://open.feishu.cn/document/client-docs/bot-v3/add-custom-bot |

选型原则只有一条：**核心能力用成熟库，不自己造轮子**。RSS 解析是最典型的例子——Feed 格式五花八门，feedparser 处理了十多年的边界情况，我写一版"能用"的解析器要一周，而它一行 `feedparser.parse(url)` 就够。

## 三、一步步实现

### 第 1 步：把数据源列成清单

数据源分成四组，全部是公开的 RSS/API：

```python
SOCIAL_RSS_FEEDS = (
    "https://www.theguardian.com/world/rss",
    "https://rss.nytimes.com/services/xml/rss/nyt/World.xml",
    "https://www.aljazeera.com/xml/rss/all.xml",
)

TECH_RSS_FEEDS = (
    "https://techcrunch.com/category/artificial-intelligence/feed/",
    "https://feeds.arstechnica.com/arstechnica/technology-lab",
    "https://www.technologyreview.com/feed/",
)

OPENSOURCE_RSS_FEEDS = (
    "https://github.blog/feed/",
    "https://www.cncf.io/feed/",
    "https://about.gitlab.com/atom.xml",
)

def bbc_world_url() -> str:
    return "https://feeds.bbci.co.uk/news/world/rss.xml"
```

另外用 Google News RSS 按检索词做"主题聚合"：

```python
def google_news_url(query: str) -> str:
    params = urlencode({"q": f"{query} when:2d", "hl": "zh-CN", "gl": "CN", "ceid": "CN:zh-Hans"})
    return f"https://news.google.com/rss/search?{params}"
```

论文用 arXiv 官方 API，按提交时间倒序取 20 条：

```python
def arxiv_url(query: str) -> str:
    params = urlencode({
        "search_query": query,
        "start": 0, "max_results": 20,
        "sortBy": "submittedDate", "sortOrder": "descending",
    })
    return f"https://export.arxiv.org/api/query?{params}"
```

### 第 2 步：抓取层——异步并发 + 时间窗口

十几个源如果串行抓，要几十秒；用 httpx 异步并发，几秒搞定。抓取的核心函数（脱敏）：

```python
async def fetch_feed(client, url, limit, start_at):
    """抓一个 RSS/Atom 源，只保留时间窗口内的条目。"""
    try:
        response = await client.get(url)
        feed = feedparser.parse(response.content)
        items = []
        for entry in feed.entries:
            published = parse_entry_time(entry)  # 统一各种时间格式
            if published is None or published < start_at:
                continue
            items.append({
                "title": html.unescape(entry.get("title", "")),
                "summary": truncate(strip_html(entry.get("summary", "")), 400),
                "url": entry.get("link", ""),
                "published": published.isoformat(),
            })
            if len(items) >= limit:
                break
        return items
    except Exception as exc:
        failed_sources.add(url)
        return []


async def fetch_feed_pool(client, urls, limit, start_at):
    """多个 RSS 源并发抓取后按时间合并。"""
    results = await asyncio.gather(*(fetch_feed(client, u, limit, start_at) for u in urls))
    merged = sorted((item for batch in results for item in batch),
                    key=lambda x: x["published"], reverse=True)
    return merged[:limit]
```

`collect()` 里用 `httpx.AsyncClient` + `asyncio.gather` 一次发起所有抓取任务：

```python
async def collect(config, state):
    start_at = news_window_start()  # 前一日 09:00（Asia/Shanghai）
    async with httpx.AsyncClient(timeout=httpx.Timeout(30, connect=12)) as client:
        tasks = {
            "social": fetch_feed(client, google_news_url(config["social_query"]), limit, start_at),
            "social_bbc": fetch_feed(client, bbc_world_url(), limit, start_at),
            "social_rss": fetch_feed_pool(client, SOCIAL_RSS_FEEDS, limit, start_at),
            "tech": fetch_feed(client, google_news_url(config["tech_query"]), limit, start_at),
            "opensource": fetch_feed_pool(client, OPENSOURCE_RSS_FEEDS, limit, start_at),
            "research": fetch_feed(client, arxiv_url(config["research_query"]), limit, start_at),
            # ... 还有 NLP 专项、苏大计算机学院动态、公司官网
        }
        sections = await asyncio.gather(**tasks)  # 简化示意
    return sections
```

时间窗口为什么要"前一日 09:00"而不是"昨天 0 点"？因为论文和新闻的"一天"通常以当地工作日起算，9 点前后刷新，这个窗口能稳定覆盖 24 小时。

### 第 3 步：公司动态——名单 + 回退简介

AI 基础设施公司大多没有 RSS，用官网抓取不可靠。我的方案是：维护一个公司名单（名字 + 官网），每天轮换一家"深度关注"，配一个手写的回退简介，抓不到官网内容时也能给出有意义的一行介绍：

```python
COMPANY_FALLBACKS = {
    "Modal": "面向开发者的无服务器云平台，重点提供按需 GPU 和 Python 运行环境。",
    "Neon": "无服务器 PostgreSQL 平台，支持按需扩缩容与分支数据库。",
    "PostHog": "开源产品分析套件，包含事件分析、会话录制与实验功能。",
    # ...
}
```

诚实地说，公司动态是整份日报里"信息含量"最弱的一块，因为小公司很少发公开新闻。它的价值更多是**提醒**：你关注的公司在做什么方向。

### 第 4 步：生成层——DeepSeek 整理成中文

所有原始条目先用 `compact()` 压缩成"标题 + 摘要 + 链接"的列表，再交给 DeepSeek 生成结构化日报：

```python
def compact(items):
    if not items:
        return "（本次检索未获得可用条目）"
    return "\n".join(
        f"- 标题：{x['title']}\n  摘要：{x['summary']}\n  链接：{x['url']}"
        for x in items
    )
```

生成 prompt 的核心要求（脱敏）：

```text
你是每日情报编辑。基于以下原始素材，输出一份中文日报，包含：
- 社会与科技新闻（5 条以内，按重要度排序）
- NLP / 机器翻译研究动态（论文标题 + 一句话结论 + 链接）
- 开源社区动态
- 今日新技术
- "小而强"团队/公司推荐
要求：只基于给定素材，不编造；每条附来源链接；总量控制在飞书卡片长度内。
```

大模型在这里的角色是"编辑"而不是"信息源"——它只整理素材，不补充知识，从根上避免编造。

### 第 5 步：推送层——飞书卡片 + 邮件

飞书自定义机器人 Webhook 推 interactive 卡片（宽屏模式，正文用 markdown）：

```python
def send_feishu(markdown, report_date):
    webhook = os.environ["FEISHU_WEBHOOK_URL"]
    payload = {
        "msg_type": "interactive",
        "card": {
            "schema": "2.0",
            "config": {"wide_screen_mode": True},
            "header": {"title": {"tag": "plain_text", "content": f"每日情报日报｜{report_date}"}, "template": "blue"},
            "body": {"elements": [{"tag": "markdown", "content": markdown}]},
        },
    }
    response = httpx.post(webhook, json=payload, timeout=30)
    response.raise_for_status()
```

同时保留 SMTP 邮件通道（`smtplib` 标准库），飞书不可用时还有备份。

### 第 6 步：状态与去重

日报最烦人的问题是"昨天看过的论文今天又出现"。用一个 JSON 状态文件记录已消费的论文 ID：

```python
def load_state():
    if not STATE_PATH.exists():
        return {"seen_research_ids": [], "last_company": ""}
    return json.loads(STATE_PATH.read_text(encoding="utf-8"))
```

只在报告**成功发送后**才推进状态，避免"源挂了导致状态被误消费"：

```python
if "research" not in failed_sources:
    state["seen_research_ids"] = research_ids
state["last_company"] = company
STATE_PATH.write_text(json.dumps(state, ensure_ascii=False, indent=2), encoding="utf-8")
```

### 第 7 步：部署

Docker 封装，目录挂载配置与状态，dry-run 模式先本地验证：

```yaml
services:
  daily-intelligence-briefing:
    build: .
    volumes:
      - ./data:/app/data   # 配置、状态、历史报告
    env_file: .env
    restart: "no"
```

由 cron 或 systemd timer 每天早上触发一次：`REPORT_DRY_RUN=1 docker compose run --rm daily-intelligence-briefing` 先看效果，确认后去掉 dry-run 正式跑。

## 四、踩过的坑

1. **RSS 时间格式五花八门**：有的带时区有的不带，有的用 GMT 有的用 +08:00。统一用 `dateutil` 风格容错解析（项目里自己写了一个带多种格式回退的解析函数）。
2. **Google News RSS 的时间戳不可靠**：上游给的是"当：2d"的宽窗口，必须在抓取后按本地时间精确过滤，否则会混进 48 小时外的旧新闻。
3. **arXiv 每天重复**：同一篇论文会连续几天出现在最新列表，必须用 ID 去重。
4. **单个源挂了会拖垮整份日报**：任何抓取异常都只记入 `failed_sources`，其他源照常生成；如果所有联网源都失败，日报会明确标注"未能验证实时信息"而不是编造。
5. **飞书卡片长度限制**：markdown 太长会被截断，生成 prompt 里明确要求控制篇幅，发送前再兜底截断。
6. **时区**：所有时间逻辑基于 `Asia/Shanghai`，避免服务器 UTC 导致"早上 9 点"变成"下午 5 点"。

## 五、怎么把别人的开源项目变成自己的工具

最后聊方法论。这次项目里 80% 的能力来自开源项目，我只写了"粘合逻辑"。复盘下来有四条经验：

1. **先搜索"有没有成熟库"，再决定自己写**。RSS 解析用 feedparser、异步 HTTP 用 httpx、LLM 调用用 openai-python——每一行都是别人踩过坑的结果。个人项目的时间应该花在"编排"而不是"造轮子"。
2. **读 README 和官方示例，别读源码**。feedparser 的用法三分钟就能学会；读源码是排查问题时的选项，不是入门路径。
3. **把开源项目当"黑盒组件"，用契约（函数签名/返回结构）而不是实现细节**。这样升级依赖、替换实现都很容易——比如我换过两次数据源 URL，代码一行没改。
4. **能贡献就贡献**。用久了发现问题，给上游提 issue 或 PR 是最好的学习方式，也让你的"使用"变成"参与"。

## 六、效果与复盘

现在每天早上一份日报准时出现在飞书里，历史报告归档在本地 `data/summaries/`，随时可查。它没有替代 RSS 阅读器，但它解决了我真正的痛点：**把"我需要主动刷"变成"它每天主动给我"**，而且每一条都能点回原文。

这个项目也验证了一个判断：**个人自动化工具的复杂度应该和它的用途匹配**。一个每天跑一次的脚本，不需要数据库、不需要前端、不需要队列——一个 Python 文件加 Docker 就够了。

项目代码（开源）：https://github.com/DreamZhongJu/daily-intelligence-briefing

如果你也在做类似的定时情报工具，欢迎交流。
