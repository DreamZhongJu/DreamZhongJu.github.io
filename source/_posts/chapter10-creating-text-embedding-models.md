---
title: 《大语言模型实用指南》第十章：构建文本 Embedding 模型
date: 2026-09-17 17:40:00
updated: 2026-09-17 17:40:00
categories:
  - 大语言模型实用指南
  - 第十章 文本 Embedding 模型
tags:
  - Embedding
  - Contrastive Learning
  - Sentence-BERT
  - Dense Retrieval
  - RAG
  - 学习笔记
description: 解释文本 Embedding 的任务依赖性，梳理 Bi-Encoder、Pooling、对比学习、Triplet Loss、批内负例与困难负样本，并说明如何用检索指标评估和迭代向量模型。
---

Embedding 不是“把文本变成一串数字”这么简单。它是在学习一个度量空间：什么文本应该彼此靠近、什么文本应该拉远，完全由训练数据、正负样本定义和损失函数决定。语义搜索、RAG、聚类、匹配和 Agent Memory 的检索质量，往往先受这个空间的质量约束。

## 一、什么是任务有用的 Embedding

编码器将文本 (x) 映射成向量 (e(x))。在语义检索中，希望语义上能回答同一需求的文本更接近：

```text
“神经机器翻译方法”      → 向量 A
“基于神经网络的机器翻译”  → 向量 B
“今天股票上涨”           → 向量 C

similarity(A, B) 高，similarity(A, C) 低
```

但“相似”本身没有唯一含义。问答检索更关心“文档是否回答查询”；复述检测更关心“两个句子是否表达同一事实”；情感任务可能更关心正负极性。一个模型在通用语义相似度上好，不代表它天然适合代码搜索、法律条款或机器翻译文献检索。

## 二、为什么直接用普通 BERT 的 CLS 向量常常不够

BERT 的掩码语言建模目标是根据上下文预测被遮住的 token。它会学到丰富的语言表示，但没有直接被要求让“可互相检索的两句话”在向量空间中接近。

此外，BERT 的 token hidden states 是上下文相关的；直接取 `[CLS]` 作为句向量，是否能表示整句语义取决于下游训练目标。对于大规模检索，需要专门优化“两个独立编码的文本能否通过向量相似度比较”的能力。

## 三、Cross-Encoder 与 Bi-Encoder：精度和效率的分工

Cross-Encoder 把查询和候选文档放进同一个 Transformer，让所有 token 直接互相注意：

```text
[query] + [document] → Transformer → relevance score
```

它能建模细粒度的词对齐和否定关系，因此排序精度通常高；但每来一个查询，都需要与每篇候选文档重新前向计算，无法预先缓存百万文档的结果。

Bi-Encoder 分别编码查询和文档，再计算向量相似度：

```text
query → Encoder → q embedding ─┐
                               ├→ cosine / dot product → score
document → Encoder → d embedding ─┘
```

文档向量可以离线建立索引，在线只编码查询并做近邻搜索，适合大规模召回。实际 RAG 常将两者组合：Bi-Encoder 先取高召回候选，Cross-Encoder Reranker 再精排少量候选。

## 四、共享参数与 Pooling

Sentence-BERT（SBERT）常用同一个 encoder 分别处理两个文本，这被称为共享权重的 Siamese/Bi-Encoder。共享不是唯一可选设计，但能使两边在同一表示体系中学习，参数更少，也利于对称的句子相似度任务。

Transformer 输出的是每个 token 的 hidden state，句子级检索却需要一个固定长度向量，因此要 Pooling：

| 方法 | 做法 | 特点 |
| --- | --- | --- |
| Mean Pooling | 对非 padding token 的 hidden states 求平均 | 稳定常用，需正确处理 attention mask |
| CLS Pooling | 取 `[CLS]` 或第一个 token | 依赖预训练/微调是否专门优化该位置 |
| Max Pooling | 每维取 token 表示的最大值 | 突出局部显著特征，较少作为默认选择 |
| Last-token Pooling | 取最后一个有效 token | 常见于某些 decoder embedding 模型 |

Pooling 不是实现细节。训练时使用哪种 Pooling，推理时必须一致；否则向量空间的分布会发生变化，索引和查询不再可比。

## 五、对比学习：直接塑造距离关系

对比学习使用正样本对和负样本对，让 anchor 更接近 positive、远离 negative。常用相似度是归一化向量的点积，即余弦相似度：

<div class="attention-formula" role="math">cosine(a, b) = (a · b) / (||a|| · ||b||)</div>

例如查询“Prompt 优化方法”的正样本可以是讨论 Prompt Engineering 的段落；仅主题相近、但不能回答该问题的 Instruction Tuning 文档可成为困难负样本。训练目标不是学会“都和 LLM 有关”，而是分清对当前查询来说什么才是正确证据。

### Triplet Loss

Triplet 由 anchor (a)、正样本 (p) 与负样本 (n) 组成，要求正样本比负样本至少更近一个 margin：

<div class="attention-formula" role="math">max(0, distance(a, p) - distance(a, n) + margin)</div>

它直观地表达了相对排序目标，但训练效果很依赖负样本挖掘：随机负样本太容易时，损失很快变为零，模型学不到细粒度边界。

### Multiple Negatives Ranking Loss

在一批匹配对 ((q_i, d_i)) 中，对每个 (q_i)，对应的 (d_i) 是正例，其他 (d_j) 可作为批内负例：

```text
batch: (Q1, D1), (Q2, D2), (Q3, D3)
Q1 的正例：D1
Q1 的批内负例：D2、D3
```

这类训练高效利用 batch，batch 越大，每个查询可见的负例越多。不过它有一个前提：批内其他文档确实不是当前查询的正例。若数据中存在多个都能回答 Q1 的文档，却被错误当作负例，会制造 false negative，反而伤害表示空间。

## 六、困难负样本决定模型能否学会边界

“Transformer 论文”与“天气预报”作为负例差异太大；模型几乎不需学习就能区分。真正有价值的困难负样本通常主题接近、措辞相似，却不满足查询意图或关键约束：

```text
Query：如何降低 RAG 的检索延迟？
Positive：讨论向量索引、缓存与候选规模的文章
Hard negative：讨论如何降低 LLM 解码延迟的文章
```

困难负样本可来自 BM25 或旧 embedding 模型的高排位误召回、同一主题下的其他段落、人工标注或合成数据。它们应经过抽查：把真正相关文档当成负例，是 embedding 训练中很常见且隐蔽的质量问题。

## 七、数据比模型名更重要

构建 embedding 模型时，训练对的质量应优先于“换一个热门 base model”。一条训练数据最好明确：查询是什么、哪个片段是正确答案、它为什么相关、负例是否确实不满足需求，以及数据来自什么领域。

```text
基础 Encoder
  → 领域内 query-document / sentence-pair 数据
  → 过滤错误配对，加入高价值 hard negatives
  → 对比学习微调
  → 在真实查询集上检索评测
  → 用失败案例继续挖掘和修正数据
```

通用模型可作为很好的起点；在术语、语言风格、文档结构明显不同的领域，继续训练或微调通常更有效。无论使用何种模型，文档与查询的预处理、指令前缀、最大长度和归一化方式必须在建库、查询和离线评测中保持一致。

## 八、Embedding 评测看的是排序，不是生成

Embedding 不生成文字，评测重点是正确文档在候选列表中的位置：

| 指标 | 回答的问题 |
| --- | --- |
| Recall@K | 正确结果是否至少有一个出现在前 K 名？ |
| MRR | 第一个正确结果排得有多靠前？ |
| nDCG@K | 前 K 名整体排序是否与多级相关性一致？ |
| Precision@K | 前 K 名中有多少真正相关？ |

离线指标需要和真实任务对应。法律、医疗或企业知识问答中，“关键条款是否被召回”可能比平均相似度更重要；多文档综合问题则不能只标一个唯一正例。还应按语言、领域、问题长度、时间敏感性和失败类型切片分析。

## 九、与 RAG 的关系：Embedding 是召回层，不是完整答案

第八章的 RAG 流程中，Embedding 模型负责将查询与 chunk 放到可搜索的表示空间，主要优化召回；Reranker 负责细粒度排序；LLM 负责基于证据组织回答与引用。

```text
Embedding 训练好 → 更可能召回正确 chunk
Reranker 做好    → 更可能将正确 chunk 排到上下文前部
Prompt 与验证做好 → 更可能基于证据回答并暴露不确定性
```

因此，不应只通过“最终回答好不好”猜测 embedding 是否出问题。应记录并检查每一步：正确 chunk 是否入库、是否被召回、是否被 rerank 保留、是否进入 Prompt、是否被回答正确引用。

## 十、本章总结

```text
好的 Embedding 空间
  = 任务定义清楚
  + 高质量正样本
  + 足够有区分度的负样本
  + 与部署一致的编码和 Pooling
  + 基于真实检索问题的持续评测
```

向量模型的核心能力不是“把句子压缩为一个向量”，而是让距离和排序对下游任务有意义。只有把数据、损失、负例和评测闭环一起设计，Embedding 才能成为可靠的检索基础设施。
