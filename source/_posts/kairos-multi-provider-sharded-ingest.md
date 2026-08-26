---
title: 三线并发灌库实战：nous/zen/openrouter 故障转移、按请求钉扎与 74 窗/分钟
date: 2026-08-26 10:00:00
updated: 2026-08-26 10:00:00
categories:
  - 项目复盘
tags:
  - Kairós
  - 知识图谱
  - LLM
  - Python
  - 工程实践
description: 把一个群 10693 个聊天窗口灌进知识图谱：三级 Provider 故障转移链、空响应保护、X-Kairos-Provider 按请求钉扎、分片断点续传，以及在 429/503 面前摸出的并发配比。
---

## TL;DR

Kairós（我的自托管个人情报助手）最近吃下了一个 8300 人的大群：**10,693 个聊天窗口**，每个窗口都要过一遍 LLM 抽取实体和关系。这篇文章记录这次"灌库工程"的完整方案：

1. **三级故障转移链**（nous → zen → openrouter），带空响应保护——推理模型把 token 烧在思考上返回 `content=None` 时不能算成功；
2. **按请求钉扎 Provider**：`X-Kairos-Provider` 头一路穿透到抽取函数，让三个导入进程各用一条线；
3. **分片断点**：`--shards=N --shard=M` 按 `i % N == M` 切窗口，flock 合并共享 checkpoint；
4. 实测并发配比：nous/zen 各 10 workers 直接被限流打崩，**openrouter 单线 26.7 窗/分，三进程 × 6 workers = 74 窗/分钟零报错**。

## 一、为什么需要故障转移

单 Provider 灌库有两个致命问题：**配额**和**稳定性**。10,693 个窗口 × 平均 2~3 次 LLM 调用（主抽取 + gleanings 补漏）≈ 3 万次请求，任何一家免费/低价 API 都可能中途限流或抖动。

所以 Kairós 的 `FailoverClient` 维护一条解析出来的调用链：

```text
nous(主) ──失败──> zen(备1) ──失败──> openrouter(备2, 走代理)
```

`create()` 顺着链试，谁先给出可用结果就用谁。

## 二、空响应保护：推理模型的隐形坑

第一个版本的 failover 有个隐蔽 bug：上游返回 HTTP 200、`finish_reason=length`，但 `content=None`。带思考模式的模型在 `max_tokens` 很小时会把预算全部烧在推理上，最终答案一个字都没吐出来。

这种响应如果算"成功"，failover 就永远不会触发。修复是加了一个统一的可用性判定：

```python
@staticmethod
def _usable(resp) -> bool:
    """A response counts only if it carries non-empty content."""
    try:
        msg = resp.choices[0].message
        return bool((msg.content or "").strip())
    except Exception:
        return False
```

然后 `create()` 里每条链成员的返回都过一遍 `_usable()`：空内容视为软失败，继续走下一家；全链皆空才返回最后一个响应兜底。这个十几行的函数，后来在真实灌库里救了我们几千次请求。

## 三、按请求钉扎：让三条线互不干扰

光有 failover 还不够快——它是串行降级，吞吐上限就是单家 API。真想要并发，得让不同进程**钉死在不同的 Provider 上**，互不抢额度、互不触发彼此的限流。

方案是一个 HTTP 头贯穿到底：

```python
# api.py
provider = request.headers.get("X-Kairos-Provider", "")
ingest_chat_window(channel_id, window, provider=provider)

# ingest.py
extracted = kg_extract.extract(window_text, provider=provider)

# extract.py —— 顶部解析一次，线程池里全部复用
pinned_client, pinned_model = client_for_provider(provider)
use_client = pinned_client or llm
```

`client_for_provider(name)` 在解析链里按 provider/model 名匹配，返回预构建好的 `(client, model)`。匹配不到就回落默认链——所以这个机制对老调用完全无侵入。

## 四、分片与断点：三个进程吃同一份队列

服务端导入脚本支持两个参数：

```bash
python import_chat_txt.py --channel=830070676 --provider=openrouter --shards=3 --shard=0
python import_chat_txt.py --channel=830070676 --provider=openrouter --shards=3 --shard=1
python import_chat_txt.py --channel=830070676 --provider=openrouter --shards=3 --shard=2
```

过滤逻辑一行：`todo = [w for i, w in enumerate(windows) if i % N == M]`。

checkpoint 是共享文件，用 `fcntl.flock` 做"读-合并-写"：

```python
with open(ckpt_path, "r+") as f:
    fcntl.flock(f, fcntl.LOCK_EX)
    data = json.load(f) if f.seek(0) == 0 else {}
    done = {**data.get("done", {}), **local_done}
    f.seek(0); f.truncate()
    json.dump({"done": done}, f)
    fcntl.flock(f, fcntl.LOCK_UN)
```

任何一个进程崩了，重启后自动跳过已完成的窗口；三个分片的进度在同一个文件里汇合，不会互相覆盖。

## 五、限流实测：数据说话

最初我天真地用了 10+10 双线并发（nous×10 + zen×10），5 分钟内的成绩单：

```text
nous: 429 Too Many Requests ×45
zen : 503 Service Unavailable ×53
吞吐: ~9 窗/分钟，且持续劣化
```

切换策略后（openrouter 承接主力，三进程各 6 workers）：

```text
openrouter ×18 并发: 零错误
总吞吐: 74 窗/分钟
10,693 窗 ≈ 2.5 小时灌完
```

结论很朴素：**限流面前，加 worker 数量不如加 provider 数量**。同一家 API 的限流阈值是按账号算的，workers 堆到一定程度只会让 429 排队更长。

## 六、运维纪律：三条血泪教训

1. **`docker restart` 之前先杀导入进程**。有一次我在导入中途重启了两次容器做别的事，醒来发现 27 分钟"正常完成"、实际 err=6221——LLM 调用全打在容器重启的空档上。
2. **后台启动后的 PipeTimeout 是良性的**。SSH 通道跑 `nohup ... & disown` 会因为 stdout 无人认领而超时报错，但进程活得好好的——验证方式永远是 `ps aux | grep`，不是看 SSH 返回值。
3. **改代码要么整包同步，要么别同步**。单文件 `docker cp` 七个文件总有漏网之鱼（这次漏了 `coref.py`），最后改成 tar 整包 + 解压覆盖，一劳永逸。

## 结语

现在这套"failover 链 + 钉扎 + 分片断点"已经沉淀成通用能力：换一个群，导出 txt、起 N 个进程，剩下的交给时间。下一篇讲这批数据进来之后更麻烦的事——**同一个群友的五六个马甲怎么在图谱里合并成一个人**。

GitHub 仓库：[DreamZhongJu/kairos-intel](https://github.com/DreamZhongJu/kairos-intel)，觉得有用欢迎 Star。
