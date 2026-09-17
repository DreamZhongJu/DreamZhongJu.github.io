---
title: 《大语言模型实用指南》学习笔记
date: 2026-09-17 15:20:00
layout: page
comments: false
---

这个系列整理《大语言模型实用指南》的阅读与学习笔记。每篇文章对应一个章节，重点保留概念之间的关系、容易混淆的边界，以及与实际使用相关的推理过程。

## 阅读目录

### 第三章：从 Forward Pass 到 Attention

[《大语言模型实用指南》第三章：从 Forward Pass 到 Attention](/2026/09/17/transformer-chapter3-study-notes/)

从文本如何进入模型开始，梳理 Tokenizer、Embedding、位置编码、因果自注意力、QKV、FFN、残差连接、LM Head 与 KV Cache，并说明 MHA、MQA、GQA 的取舍。

### 第四章：Text Classification 与现代 LLM 任务范式

[《大语言模型实用指南》第四章：Text Classification 与现代 LLM 任务范式](/2026/09/17/transformer-chapter4-text-classification/)

对比 BERT 微调、Embedding 加分类器和生成式 LLM 三条文本分类路线，并说明 T5 的 text-to-text 范式，以及指令微调与偏好对齐在 ChatGPT 中各自解决的问题。

### 第五章：文本聚类、主题建模与 BERTopic

[《大语言模型实用指南》第五章：文本聚类、主题建模与 BERTopic](/2026/09/17/chapter5-text-clustering-topic-modeling-bertopic/)

解释 Embedding、UMAP、HDBSCAN、c-TF-IDF 和 LLM 在主题建模流水线中的职责，并厘清 BERTopic 与 RAG、GraphRAG 的联系和边界。

### 第六章：Prompt Engineering、推理控制与输出约束

[《大语言模型实用指南》第六章：Prompt Engineering、推理控制与输出约束](/2026/09/17/chapter6-prompt-engineering-and-output-control/)

从 Prompt 如何改变上下文开始，梳理解码参数、任务链、In-context Learning、CoT、自洽性，以及面向 RAG 和 Agent 的结构化输出与验证机制。

### 第七章：LLM Memory、状态管理与 Agent 基础

[《大语言模型实用指南》第七章：LLM Memory、状态管理与 Agent 基础](/2026/09/17/chapter7-llm-memory-state-and-agent-foundations/)

从 LLM 的无状态本质出发，比较短期会话记忆策略，并梳理长期记忆的提取、检索、冲突处理、遗忘与权限边界，以及它与 RAG 的区别。

### 第八章：RAG、语义搜索与外部知识增强

[《大语言模型实用指南》第八章：RAG、语义搜索与外部知识增强](/2026/09/17/chapter8-rag-semantic-search-and-external-knowledge/)

解释 Chunk、Dense Retrieval、Hybrid Search、Reranker、查询改写与多跳检索的关系，并给出面向检索、上下文和生成的 RAG 评测框架。

### 第九章：多模态 LLM 与视觉语言模型

[《大语言模型实用指南》第九章：多模态 LLM 与视觉语言模型](/2026/09/17/chapter9-multimodal-llm-vision-language-models/)

从 ViT 的 Patch 表示、CLIP 的图文对齐到 BLIP-2 的视觉语言桥接，理解图像如何转换成可供 LLM 使用的视觉 token，以及相关训练和工程取舍。

### 第十章：构建文本 Embedding 模型

[《大语言模型实用指南》第十章：构建文本 Embedding 模型](/2026/09/17/chapter10-creating-text-embedding-models/)

梳理 Bi-Encoder、Pooling、对比学习、Triplet Loss、批内负例和困难负样本，并说明如何以检索指标评估和迭代向量模型。

后续章节会持续补充到本页。
