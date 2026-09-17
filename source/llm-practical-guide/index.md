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

后续章节会持续补充到本页。
