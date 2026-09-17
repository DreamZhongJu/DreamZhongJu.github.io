---
title: 《大语言模型实用指南》第三章：从 Forward Pass 到 Attention
date: 2026-09-17 14:45:21
updated: 2026-09-17 15:20:00
categories:
  - 大语言模型实用指南
  - 第三章 Transformer
tags:
  - Transformer
  - Attention
  - 大语言模型
  - 学习笔记
description: 从一次前向传播出发，系统理解 Tokenizer、Embedding、位置编码、因果自注意力、QKV、残差连接、FFN、LM Head、KV Cache，以及 MHA、MQA 和 GQA 的作用与取舍。
---

这一章讨论的不是“模型怎样凭空写出一句话”，而是模型在某一轮计算里如何把已有 token 变成**下一个 token 的概率分布**。把这条链路看清后，Attention、FFN、KV Cache 等概念会自然落到各自的位置上。

## 一、先建立总图：一次生成到底发生了什么

给定已出现的文本，模型会重复做下面这件事：

```text
上下文文本
  → Tokenizer（文本切分并编号）
  → token IDs
  → Token Embedding + Position Embedding
  → N 个 Transformer Block
      ├─ 因果自注意力：从允许看到的上下文取信息
      └─ FFN：逐位置做非线性特征变换
  → 最后一个位置的 hidden state
  → LM Head（映射到词表）
  → logits → softmax 概率分布
  → 解码策略选出一个 token
  → 把该 token 追加回上下文，进入下一轮
```

所以自回归语言模型的核心任务是：

> 在前文已经给定时，估计下一个 token 的条件概率 (P(x_t \mid x_{<t}))。

整段回答并不是一次性输出的。假设当前上下文是“今天天气”，模型先预测“很”的分布，再把“很”放入上下文预测“好”，如此循环。训练时可以同时计算一段文本中所有位置的下一个 token 损失；推理时，下一个 token 尚未确定，仍要逐个生成。

## 二、Tokenizer、Embedding 与位置：输入是怎样进入网络的

### 1. Token ID 不携带语义

Tokenizer 把文本切成词表中的子词单元，并映射成整数。例如“机器翻译”可能被切成一个或多个 token：

```text
“机器翻译” → [31872, 9461]
```

这些数字只是词表索引。`31872` 比 `9461` 大，并不意味着前者“语义更多”或“距离更远”。神经网络不能直接从这种编号顺序中获得语言规律。

### 2. Embedding 是可学习的查表

模型维护一个形状约为 (V \times d_{model}) 的嵌入矩阵：(V) 是词表大小，(d_{model}) 是隐藏维度。token ID 用来取出矩阵中的一行：

```text
token id  →  embedding table lookup  →  d_model 维向量
```

这一步是查表，不是 Transformer Block。训练会更新嵌入表中的参数，使经常在相似语境出现的 token 在表示空间中形成有用的结构。

### 3. 还必须告诉模型顺序

自注意力本身对输入排列不敏感：把 token 向量整体换序，注意力只会看到另一组向量，无法知道谁在前谁在后。因此输入给第一层的通常是：

<div class="attention-formula" role="math">h<sup>(0)</sup><sub>i</sub> = token_embedding(x<sub>i</sub>) + position_embedding(i)</div>

现代模型常用 RoPE（旋转位置编码）等方式，将相对位置信息注入 Query 与 Key；它和“给每个位置加一个绝对位置向量”形式不同，但目标相同：让注意力分数能感知相对距离和顺序。

## 三、一个 Transformer Block：信息交流，再局部加工

以常见的 Pre-Norm Decoder Block 为例，一层的计算结构可以概括为：

```text
x
 ├─ RMSNorm / LayerNorm → Causal Multi-Head Attention ─┐
 └─────────────────────────────────────────────────────┼→ x + attention_output = y
                                                        │
y
 ├─ RMSNorm / LayerNorm → FFN ──────────────────────────┐
 └──────────────────────────────────────────────────────┼→ y + ffn_output = 下一层输入
```

Attention 和 FFN 不是二选一，而是职责互补的两个子层：

| 子层 | 信息如何流动 | 它回答的问题 |
| --- | --- | --- |
| Attention | 一个位置从其他允许位置读取信息 | “此刻我该参考上下文中的谁？” |
| FFN | 每个位置独立使用同一组参数变换 | “把读到的信息组合后，提取出什么特征？” |

残差连接保留原始通道并改善深层网络的梯度传播；归一化稳定每层输入的数值尺度。它们是 Transformer 能堆叠几十层甚至上百层的关键工程组成，并非无关紧要的装饰。

## 四、Attention 在做什么：为每个位置动态查资料

设一层输入为 (X \in \mathbb{R}^{n \times d_{model}})，其中 (n) 是序列长度。模型通过三组可学习的投影矩阵生成：

<div class="attention-formula" role="math">Q = XW<sub>Q</sub>，K = XW<sub>K</sub>，V = XW<sub>V</sub></div>

可以用“检索”来理解它们：

- **Query（Q）**：当前位置正在提出什么需求。
- **Key（K）**：每个候选位置有哪些可被匹配的线索。
- **Value（V）**：一旦被选中，实际要传出的内容。

同一个 token 在不同层、不同头中会投影出不同的 Q/K/V；它们不是词典里固定的“问题、标签、正文”，而是随着上下文计算得到的向量。

### 1. 分数、缩放与 softmax

第 (i) 个 token 对第 (j) 个 token 的匹配分数是 (q_i \cdot k_j)。收集为矩阵后，缩放点积注意力为：

<div class="attention-formula" role="math">Attention(Q, K, V) = softmax((QK<sup>T</sup> / √d<sub>k</sub>) + M)V</div>

其中：

- 除以 (√d_k) 用来避免维度大时点积方差过大，softmax 过早变得过尖，从而让训练不稳定。
- (M) 是掩码矩阵。允许关注的位置加 0，不允许的位置加一个极小值（实现中近似 (-∞)）。
- softmax 沿每一行归一化，因此每个 Query 都得到一组权重和为 1 的分布。
- 最后用这些权重加权求和 Value，得到新的上下文相关表示。

它不是挑出“最相关的一个词”并复制，而是连续的加权汇聚。例如代词“它”的表示可以同时吸收主语、宾语和句法线索，只是权重不同。

### 2. 为什么语言模型必须使用因果掩码

训练文本“我 喜欢 猫”时，预测“喜欢”的位置不能偷看后面的“猫”；否则训练指标会很好，实际生成时却无法复现这种信息条件。Decoder-only LLM 使用下三角因果掩码：

```text
可见性（行是当前 Query，列是被关注的 Key）

          我   喜欢  猫
我        ✓    ×    ×
喜欢      ✓    ✓    ×
猫        ✓    ✓    ✓
```

这也解释了一个常见表面矛盾：训练期间虽然所有 token 的矩阵运算可以并行完成，但每个位置能看到的信息仍被严格限制在左侧；推理期间由于新的 token 必须先被采样出来，生成过程仍然是串行的。

## 五、多头注意力：在多个表示子空间中并行检索

单头注意力只有一套投影，可能把语法、指代、位置和语义关系都挤在同一组相关性里。Multi-Head Attention 将隐藏维度切分为多个头：

```text
每个 head：Q_h, K_h, V_h → Attention_h
所有 head 的输出拼接 → 线性投影 W_O → attention output
```

不同头没有被人为指定职责，但训练可能使某些头更擅长局部搭配、长距离依赖、分隔符或实体指代。多头的价值是提供多组可学习的“查询视角”，不能把某一个头机械地解释为固定语法规则。

## 六、FFN：不跨 token，但负责逐位置的非线性计算

Attention 把上下文信息带到当前位置后，FFN 对每个位置分别处理。经典形式是：

<div class="attention-formula" role="math">FFN(x) = W<sub>2</sub> σ(W<sub>1</sub>x + b<sub>1</sub>) + b<sub>2</sub></div>

通常先从 (d_{model}) 升到更宽的 (d_{ff})，经过 GELU、SiLU 等非线性，再投影回 (d_{model})。许多现代 LLM 使用 SwiGLU 一类门控 FFN：一条分支产生内容，另一条分支决定哪些维度通过。

因此，“Attention 负责 token 之间通信，FFN 负责 token 内部特征变换”是很有用的近似，但要记住：FFN 的输入已经含有 Attention 融合进来的上下文，它并不是只处理原始词义。

## 七、从 hidden state 到下一个 token：LM Head 与解码

经过最后一层后，每个位置都有 hidden state。要预测下一个 token，只取最后一个位置 (h_t)，用 LM Head 投影到词表大小：

<div class="attention-formula" role="math">logits = h<sub>t</sub>W<sub>vocab</sub><sup>T</sup> + b</div>

logits 是未归一化分数，不是概率。softmax 后才得到每个 token 的概率。实际生成还要选择解码策略：

| 策略 | 做法 | 适合场景 |
| --- | --- | --- |
| Greedy | 每次取概率最大 token | 结果稳定，但可能重复或保守 |
| Temperature | 调整 logits 的尖锐程度 | 控制随机性 |
| Top-k | 只在概率最高的 k 个 token 中采样 | 限制低概率噪声 |
| Top-p | 取累计概率达到 p 的候选集采样 | 候选规模随分布自适应 |

模型参数保存的是训练得到的统计规律；Attention、FFN 在一次前向传播中计算的是当前上下文的激活值。Attention 改变当前 hidden state，并不会在推理时修改模型知识或参数。

## 八、KV Cache：为什么生成会越来越占显存

生成第 (t) 个 token 时，新的 Query 需要和历史所有 Key、Value 交互。历史 token 的 K/V 在当前层不会因新 token 的加入而改变，所以可以缓存：

```text
第 t 步：计算新 token 的 q_t、k_t、v_t
缓存：K = [k_1, ..., k_t]，V = [v_1, ..., v_t]
注意力：q_t 只需与缓存中的 K、V 计算
```

不缓存 Query，是因为旧 token 的 Query 不再参与“为新 token 查询上下文”的计算；新一步需要的是新 token 自己的 Query。KV Cache 避免反复重算历史层，但缓存大小随层数、序列长度、KV 头数和 head 维度线性增长，因此长上下文推理常常受显存和带宽限制。

## 九、MHA、MQA 与 GQA：用更少 KV 换推理效率

| 结构 | Query 头 | Key/Value 头 | 特点 |
| --- | ---: | ---: | --- |
| MHA | 每个注意力头一套 Q | 每个头一套 KV | 表达灵活，KV Cache 最大 |
| MQA | 多个 Q 头 | 全部共享一套 KV | Cache 很小，但共享最强 |
| GQA | 多个 Q 头 | 若干组共享 KV | 在质量与缓存之间折中，现代 LLM 很常用 |

若有 32 个 Query 头，GQA 每 4 个 Query 头共享一组 KV，则只有 8 组 KV。它减少的是 Key/Value 缓存与读取成本，不是把 Query 也合并掉；保留多组 Query 让模型仍能以多种方式提出检索需求。

## 十、把整章串起来：四个容易混淆的边界

1. **Tokenizer 与 Embedding**：前者将文本映射为离散 ID，后者将 ID 查表成连续向量。
2. **Attention 与参数更新**：Attention 在前向传播中重组信息；只有训练中的反向传播和优化器会更新参数。
3. **FFN 与 LM Head**：FFN 继续加工 hidden state；LM Head 才把 hidden state 映射为词表 logits。
4. **训练并行与生成串行**：训练可借因果掩码并行算所有位置；生成必须等待上一步采样出的 token。

理解 Transformer 最好的视角是：每层先让每个位置按当前需求从历史上下文检索信息，再在该位置上做非线性加工；这一过程层层叠加，最后由 LM Head 把最后位置的表示翻译成下一个 token 的概率分布。
