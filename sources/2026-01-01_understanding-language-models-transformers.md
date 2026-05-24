# 05 - Understanding Language Models: Transformers

> 课程来源：DeepLearning.AI / LLM-book.com

---

## 1. Transformer 的起源："Attention is All You Need"

**核心思想**：完全基于注意力机制（Attention），抛弃循环神经网络（RNN）。

- 2017 年论文 "Attention is All You Need" 提出了 Transformer 架构
- **关键优势**：可以**并行训练**，大幅提升计算速度（RNN 必须顺序处理，无法并行）

![Transformer 编码器-解码器结构](screenshots/03_encoder_decoder_blocks.jpg)

**Transformer 由堆叠的 Encoder（编码器）和 Decoder（解码器）块组成**，每个块内部都包含注意力机制。通过堆叠多层，模型的表达能力逐层增强。

---

## 2. Encoder（编码器）详解

![编码器内部结构](screenshots/04_encoder_detail.jpg)

编码器的处理流程：

1. **输入转换为 Embeddings**：与 Word2Vec 不同，这里从**随机初始化**的值开始
2. **Self-Attention（自注意力）**：对输入序列自身进行注意力计算，更新 embeddings
3. **Feed-forward Neural Network（前馈神经网络）**：进一步处理
4. **输出**：生成**上下文化的词嵌入**（Contextualized Word Embeddings）

### Self-Attention 是什么？

- 自注意力是注意力机制的一种变体
- 不是处理两个不同的序列，而是**将输入序列与自身进行比较**
- 目的：让每个词都能"看到"句子中的其他词，获取上下文信息

---

## 3. Decoder（解码器）详解

![解码器内部结构](screenshots/08_masked_self_attention.jpg)

解码器的处理流程：

1. **接收已生成的词**（Previously generated words）
2. **Masked Self-Attention（掩码自注意力）**：处理已生成的序列
3. **Encoder Attention（编码器注意力）**：结合编码器的输出，同时处理"已有信息"和"已生成信息"
4. **Feed-forward Neural Network**
5. **输出下一个词**

### Masked Self-Attention 为什么要"掩码"？

- 去除注意力矩阵上三角的值（即未来位置）
- **确保每个 token 只能关注它之前的 token，不能"偷看"未来的词**
- 防止在生成过程中泄露信息

---

## 4. 两大主流架构

### 4.1 表示模型（Representation Models）—— BERT

![BERT 架构](screenshots/09_bert_architecture.jpg)

**BERT** = Bidirectional Encoder Representations from Transformers（2018）

- **Encoder-only 架构**：只用编码器，不用解码器
- 目标：生成**上下文化的词嵌入**，用于理解语言
- 特点：
  - 输入包含 **[CLS] token**（分类标记），用作整个输入的表示
  - 可用于 fine-tuning（微调）到具体任务（如分类）

**训练方式：Masked Language Modeling（MLM，掩码语言模型）**

1. 随机遮住输入中的一些词
2. 让模型预测被遮住的词
3. 通过这种方式学会理解语言

**训练分两步：**
- **Pre-training（预训练）**：在大量数据上做 MLM
- **Fine-tuning（微调）**：在具体下游任务上调整

---

### 4.2 生成模型（Generative Models）—— GPT

![GPT 解码器架构](screenshots/12_gpt_decoder_only.jpg)

**GPT** = Generative Pre-Trained Transformer

- **Decoder-only 架构**：只用解码器，不用编码器
- 使用 Masked Self-Attention + Feed-forward Neural Network
- 逐个生成下一个词

---

## 5. Context Length（上下文长度）

![上下文长度示意](screenshots/13_context_length.jpg)

- **上下文长度** = 当前正在处理的 token 总数（输入 + 已生成的 token）
- 模型有**最大上下文长度**限制（如 GPT-1 的限制是 512 tokens）
- 生成的 token 也会占用上下文空间，逐渐消耗可用长度

---

## 6. 模型规模的增长

![参数规模](screenshots/14_model_scale.jpg)

| 模型 | 参数量 |
|------|--------|
| GPT-1 | ~117 Million（1.17亿） |
| GPT-2 | ~1.5 Billion（15亿） |
| GPT-3 | ~175 Billion（1750亿） |

**参数越多，模型能力越强** —— 这就是为什么要做"大"模型。

---

## 7. 生成式 AI 的爆发

![生成式 AI 时代](screenshots/15_generative_ai_year.jpg)

- **2023 年**：被称为"生成式 AI 之年"
- 始于 ChatGPT（GPT-3.5）的发布
- 随后大量**闭源模型**（Proprietary Models）跟进
- 很快**开源模型**（Open-Source Models）也跟上，权重公开可用，部分可商用

---

## 核心概念总结

| 概念 | 解释 |
|------|------|
| Transformer | 完全基于注意力机制的架构，支持并行训练 |
| Self-Attention | 序列与自身比较，获取上下文信息 |
| Masked Self-Attention | 遮住未来位置，防止信息泄露 |
| Encoder | 理解/表示语言，生成上下文化嵌入 |
| Decoder | 生成文本，逐词预测下一个 token |
| BERT | Encoder-only，用于理解任务 |
| GPT | Decoder-only，用于生成任务 |
| Context Length | 模型单次能处理的最大 token 数 |
| Pre-training + Fine-tuning | 先大规模预训练，再针对具体任务微调 |

---

## 关键时间点索引（方便回看视频）

| 时间 | 内容 |
|------|------|
| 0:00 | 引入：从 Attention 到 Transformer |
| 0:43 | Transformer 整体结构（Encoder + Decoder 堆叠） |
| 1:08 | Encoder 内部细节 |
| 1:53 | Self-Attention 解释 |
| 2:12 | Decoder 处理流程 |
| 2:46 | Masked Self-Attention 解释 |
| 3:06 | 原始 Transformer 的局限（主要用于翻译） |
| 3:19 | BERT 架构介绍 |
| 4:05 | Masked Language Modeling 训练方法 |
| 4:30 | Pre-training + Fine-tuning 两步训练 |
| 4:46 | GPT / Decoder-only 生成模型 |
| 5:47 | Context Length 概念 |
| 6:34 | 模型参数规模增长 |
| 7:01 | 2023 生成式 AI 爆发 |
