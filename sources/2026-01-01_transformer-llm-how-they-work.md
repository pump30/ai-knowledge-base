# How Transformer LLMs Work - 完整课程笔记

> 课程来源：DeepLearning.AI / LLM-book.com  
> 讲师：Jay Alammar & Maarten Grootendorst  
> 参考书：*Hands on Large Language Models*

---

## 第1课 Introduction（课程介绍）

### 核心要点

- Transformer 架构由 2017 年论文 "Attention is All You Need" 提出，最初用于机器翻译（英→德）
- 原始 Transformer 包含两大部分：**Encoder（编码器）** 和 **Decoder（解码器）**
- 现代 LLM 的"魔法"来自两个方面：(1) Transformer 架构 (2) 海量训练数据

### 两大模型家族

| 类型 | 代表 | 用途 |
|------|------|------|
| Encoder 模型 | BERT、Embedding 模型 | 文本表示、RAG 应用 |
| Decoder 模型 | GPT、Claude、Llama | 文本生成、代码编写、问答 |

### 生成流程概览

1. 输入 token → Embedding 向量
2. 经过一堆 Transformer Block 处理
3. Language Modeling Head 输出下一个 token

---

## 第2课 Language as a Bag-of-Words（词袋模型）

### 核心问题：如何把文字变成计算机能处理的数字？

![词袋模型的向量表示](screenshots/bow_vector.jpg)

### Bag-of-Words 工作原理

1. **分词（Tokenization）**：把句子拆成单词
2. **建立词汇表（Vocabulary）**：收集所有出现过的唯一单词
3. **计数**：统计每个词在输入中出现的次数
4. **生成向量**：得到一个数字列表，即**向量表示**

> **生动比喻**：想象你有一个信封，把句子里的词都倒进去，然后数每个词出现了几次。你不关心词的顺序，只关心"有什么词"和"有多少个"——就像把句子装进一个袋子里。

### 例子

- 输入 "My cat is cute"
- 词汇表：[that, is, a, cute, dog, my, cat]
- 向量：[0, 1, 0, 1, 0, 1, 1]（对应每个词出现的次数）

### 局限性

- **完全忽略语义**：不理解词的含义
- **忽略词序**："狗咬人"和"人咬狗"得到相同的向量表示
- 向量是**稀疏的**（大部分是0）

---

## 第3课 Word Embeddings（词嵌入）

### 从 Bag-of-Words 到 Word2Vec

Bag-of-Words 的致命缺陷：它认为语言只是一堆词的集合，完全忽略了**语义**。

**Word2Vec**（2013）是第一个成功捕获词义的方法。

![Word2Vec 词嵌入空间](screenshots/word2vec_embeddings.jpg)

### Word2Vec 工作原理

1. 给每个词分配一个**随机初始化的向量**（比如5个值）
2. 从训练数据中取词对，预测它们是否在句子中相邻
3. 不断训练，学习词与词之间的关系
4. 最终：**经常出现在相似上下文中的词，嵌入向量更接近**

> **生动比喻**：想象你是一个新转学的学生。你不认识任何人，但你观察到：小明总和小红、小刚一起出现；小李总和小王、小赵一起出现。过了一段时间，你就知道小明和小红是同一类朋友圈的。Word2Vec 就是这样——通过观察词的"朋友圈"（上下文）来理解词义。

### 嵌入的性质

![嵌入空间中的相似性](screenshots/embedding_similarity.jpg)

- 每个维度可以理解为某种"属性"（如：动物性、复数性、新生性）
- "cats"：动物性高，复数高，新生低
- "puppy"：动物性高，新生高，复数低
- **含义相似的词在向量空间中距离更近**

### 嵌入的层级

| 级别 | 说明 |
|------|------|
| Token Embedding | 每个 token 一个向量 |
| Word Embedding | 平均一个词的所有 token 嵌入 |
| Sentence Embedding | 表示整个句子 |
| Document Embedding | 表示整个文档 |

### Word2Vec 的局限

- **静态嵌入**：同一个词无论在什么上下文中，嵌入都一样
- "bank"（银行）和 "bank"（河岸）会得到相同的向量！

---

## 第4课 Encoding and Decoding Context with Attention（注意力机制）

### 问题：如何让嵌入包含上下文信息？

**RNN（循环神经网络）**的尝试：
- 用 Encoder 将整个输入序列压缩成一个 context embedding
- 用 Decoder 基于这个 embedding 生成输出

**RNN 的问题**：单个 context embedding 无法充分表示长句子的完整信息。

### Attention（注意力机制）登场（2014）

![注意力机制示意](screenshots/attention_mechanism.jpg)

**核心思想**：生成时，不只看一个压缩的向量，而是让模型**关注输入序列中所有位置**，并根据相关性分配权重。

> **生动比喻**：想象你在翻译一本书。
> - **没有注意力的 RNN**：你先把整本书读完，然后凭记忆翻译。书越长，忘得越多。
> - **有注意力的模型**：你翻译每个句子时，可以翻回去看原文的任何位置，重点看和当前翻译最相关的段落。

### 注意力权重示例

翻译 "I love llamas" → "Ik hou van lama's"（荷兰语）：
- 翻译 "Ik" 时，注意力高度集中在 "I"
- 翻译 "lama's" 时，注意力集中在 "llamas"
- 不相关的词对（如 "I" 和 "llamas"）注意力权重很低

### 自回归生成（Autoregressive）

模型一次生成一个 token：
1. 输入 "I love llamas" → 生成 "Ik"
2. 输入 "I love llamas Ik" → 生成 "hou"
3. 输入 "I love llamas Ik hou" → 生成 "van"
4. 重复直到完成

**RNN 的致命缺陷**：顺序处理，**无法并行**→训练极慢。

---

## 第5课 Transformers（Transformer 架构）

### "Attention is All You Need"

![Transformer 编码器-解码器结构](screenshots/03_encoder_decoder_blocks.jpg)

**革命性创新**：完全抛弃 RNN，只用注意力机制。

- 支持**并行训练**，大幅提速
- Transformer = 堆叠的 Encoder 块 + 堆叠的 Decoder 块

### Encoder 内部

![编码器内部结构](screenshots/04_encoder_detail.jpg)

1. 输入 → 随机初始化 Embeddings
2. **Self-Attention**：输入和自身比较，更新嵌入
3. **Feed-forward Neural Network**：进一步处理
4. 输出：**上下文化词嵌入**

### Decoder 内部

![解码器完整结构](screenshots/08_masked_self_attention.jpg)

1. 已生成的词 → **Masked Self-Attention**
2. + Encoder 的输出 → **Encoder Attention**
3. → Feed-forward Network
4. → 生成下一个词

### Masked Self-Attention 为什么要"掩码"？

- 遮住注意力矩阵的上三角（未来位置）
- 每个 token 只能看到它**之前**的 token
- 防止"作弊"偷看答案

> **生动比喻**：考试时用挡板只让你看到已经写好的答案，不能偷看后面的题目。

### 两大后续架构

#### BERT（2018）— Encoder-only

![BERT 架构](screenshots/09_bert_architecture.jpg)

- 12 层 Transformer Encoder 堆叠
- 输入前加 [CLS] token 代表整句
- 训练方式：**Masked Language Modeling**（随机遮词，让模型猜）
- 用途：文本分类、语义搜索、NER 等理解任务

#### GPT — Decoder-only

![GPT 架构](screenshots/12_gpt_decoder_only.jpg)

- 只用 Decoder，不用 Encoder
- 训练方式：预测下一个词
- 用途：文本生成（ChatGPT、Claude 等）

### Context Length（上下文长度）

![上下文长度](screenshots/13_context_length.jpg)

- = 输入 prompt 的 token 数 + 已生成的 token 数
- 模型有最大限制（如 GPT-1: 512, 现代模型: 128K+）
- 生成的 token 也占用空间

### 模型规模

![参数量增长](screenshots/14_model_scale.jpg)

| 模型 | 参数量 | 年份 |
|------|--------|------|
| GPT-1 | 117M | 2018 |
| GPT-2 | 1.5B | 2019 |
| GPT-3 | 175B | 2020 |

### 生成式 AI 爆发

![生成式 AI 时代](screenshots/15_generative_ai_year.jpg)

2023 年被称为"生成式 AI 之年"，始于 ChatGPT，随后闭源和开源模型竞相涌现。

---

## 第6课 Tokenizers（分词器）

### 为什么需要 Tokenizer？

LLM 不能直接处理文字，需要先把文字切成**token**（可以是整词或词片段），再转成数字 ID。

### 分词层级

| 层级 | 示例 "bards" | 特点 |
|------|-------------|------|
| Word | bards | 简单但词汇表巨大 |
| Subword | b + ards | 灵活，最常用 |
| Character | b + a + r + d + s | 词汇表小但序列长 |
| Byte | 每个字符的 UTF-8 字节 | 最底层 |

> **生动比喻**：Tokenizer 就像一把裁缝的剪刀。有的剪刀把布按整块裁（word-level），有的按花纹裁（subword-level），有的一条线一条线裁（character-level）。大多数现代 LLM 用的是"按花纹裁"的方式——既不会让词汇表爆炸，又能处理没见过的词。

### 特殊 Token

- **[CLS]**：BERT 用，代表整句的分类表示
- **[SEP]**：句子结束标记
- **[UNK]**：未知 token（无法表示时的兜底）

### 词汇表大小的权衡

| 模型 | 词汇表大小 | 效果 |
|------|-----------|------|
| BERT | ~30,000 | 复杂词需拆成多个子token |
| GPT-4 (tiktoken) | ~100,000 | 更少 token 表示同样文本 |

**权衡**：词汇表越大 → 表示能力越强、序列越短；但每个 token 都需要对应的嵌入向量 → 计算开销增大。

---

## 第7课 Architectural Overview（架构总览）

### Transformer LLM 三大组件

![三大组件](screenshots/transformer_3_components.jpg)

```
输入文字 → [Tokenizer] → [Transformer Blocks 堆叠] → [Language Modeling Head] → 输出 token
```

1. **Tokenizer**：文字 → token ID → 嵌入向量
2. **Transformer Blocks**：核心计算层，多层堆叠
3. **Language Modeling Head**：为词汇表中每个 token 打分，选出最可能的下一个 token

### Language Modeling Head

![LM Head 评分](screenshots/lm_head_scoring.jpg)

- 为每个候选 token 计算概率（所有概率之和 = 100%）
- 概率最高的 token 成为输出

### 解码策略（Decoding Strategies）

| 策略 | 说明 | 对应参数 |
|------|------|---------|
| Greedy Decoding | 总是选概率最高的 | temperature = 0 |
| Top-P Sampling | 从概率最高的若干 token 中随机选 | temperature > 0 |

> **生动比喻**：想象模型在玩"接龙游戏"。Greedy 就是每次都选最"正确"的词；Top-P 就是每次从几个"不错"的选项中随机挑一个——这就是为什么同一个 prompt 多次运行会得到不同回答。

### 并行处理

- Transformer 可以**同时处理所有输入 token**（不像 RNN 必须顺序）
- 想象多条"轨道"同时穿过 Transformer Block 堆栈
- 轨道数 = 模型的上下文大小（如 16K tokens = 16K 条轨道并行）

### KV Cache（键值缓存）

- 生成第 2、3、... 个 token 时，前面的计算结果可以**缓存复用**
- 大幅加速生成过程
- 关键指标：**Time to First Token (TTFT)** = 处理完整个 prompt 并生成第一个输出 token 的时间

---

## 第8课 The Transformer Block（Transformer 块）

### Transformer Block = Self-Attention + Feed-Forward Network

![Transformer Block 两大组件](screenshots/transformer_block_components.jpg)

### Feed-Forward Neural Network 的直觉

- 可以理解为模型的**"记忆存储"**
- 存储了训练数据中的统计规律
- 例如：看到 "Shawshank" 后面经常跟 "Redemption" → 学会这个模式

> **生动比喻**：Feed-Forward Network 像一本巨大的"常识百科"。它记住了"巴黎是法国首都"、"水在100度沸腾"这类知识。

### Self-Attention 的直觉

![共指消解示例](screenshots/self_attn_coreference.jpg)

- 让模型理解**代词指代关系**等上下文依赖
- 例如 "The dog chased the llama because **it**..."——"it"指谁？
- Self-Attention 让 "it" 的表示中融入它所指代的词的信息

> **生动比喻**：Self-Attention 像一个班级里的"八卦网络"。每个学生（token）都会环顾四周，看看其他同学（token）谁和自己最相关，然后"吸收"相关同学的信息到自己的理解中。

### Self-Attention 两步走

1. **Relevance Scoring（相关性打分）**：评估每个历史 token 对当前 token 的重要程度
2. **Information Combining（信息融合）**：根据分数加权融合相关 token 的信息

---

## 第9课 Self-Attention 详解

### Query、Key、Value

![QKV 矩阵和评分](screenshots/qkv_scoring.jpg)

Self-Attention 使用三个投影矩阵：

| 矩阵 | 作用 | 比喻 |
|------|------|------|
| **Query (Q)** | 当前 token 的"提问" | "我在找什么信息？" |
| **Key (K)** | 每个历史 token 的"标签" | "我能提供什么信息？" |
| **Value (V)** | 每个历史 token 的"内容" | "我的实际信息是什么？" |

> **生动比喻**：想象你在图书馆找书。
> - **Query** = 你心里想找的主题（"我想找关于猫的书"）
> - **Key** = 每本书封面上的标签/标题
> - **Value** = 书里面的实际内容
> 
> 你用自己的需求（Query）去匹配书架上的标签（Key），找到最匹配的书后，取出它的内容（Value）来阅读。

### 计算过程

1. Q × K^T → 相关性分数（scores add up to 100%）
2. Scores × V → 加权值
3. 求和 → 当前 token 的富上下文表示

### Multi-Head Attention（多头注意力）

![多头注意力](screenshots/multi_head_attention.jpg)

- 同一层有**多个独立的注意力头**并行运算
- 每个头有自己的 Q/K/V 矩阵
- 不同的头可以关注不同类型的关系（语法、语义、位置等）
- 最后把所有头的输出合并

> **生动比喻**：就像一个侦探团队——一个人负责看动机，一个人负责看不在场证明，一个人负责看物证。每人用不同视角分析同一个案件，最后把各自发现汇总成完整报告。

### 效率优化：Grouped Query Attention (GQA)

- **Multi-Head Attention**：每个头独立的 Q/K/V
- **Multi-Query Attention**：所有头共享 K/V，只有 Q 独立
- **Grouped Query Attention**：折中——几个头为一组共享 K/V

目的：减少计算量，加速推理。

### Sparse Attention（稀疏注意力）

![稀疏注意力模式](screenshots/sparse_attention.jpg)

- **Full Attention**：每个 token 关注所有前面的 token
- **Sparse Attention**：部分层只关注最近的 N 个 token
- 目的：处理超长上下文（100K+ tokens）时节省计算

实现方式：
- **Strided**：只看最近几个 + 固定间隔的位置
- **Fixed**：固定窗口大小

---

## 第10课 Model Example（模型实例 - Phi-3）

### 用代码探索模型架构

使用 HuggingFace Transformers 库加载 **Phi-3-mini** 模型：

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline

model = AutoModelForCausalLM.from_pretrained("microsoft/Phi-3-mini-4k-instruct")
tokenizer = AutoTokenizer.from_pretrained("microsoft/Phi-3-mini-4k-instruct")

pipe = pipeline("text-generation", model=model, tokenizer=tokenizer, max_new_tokens=50, do_sample=False)
```

### Phi-3 模型结构

- **词汇表**：32,000 tokens
- **模型维度**：3072
- **Transformer 层数**：32
- **FFN 隐藏层维度**：16,000（先扩大再压缩回 3072）
- **Language Modeling Head**：3072 → 32,000（为每个候选 token 打分）

### 关键观察

- 模型**从未真正"看过"人类文字**——它只处理数字 ID 列表
- 输入 "The capital of France is" → tokenizer 转为 [450, 7483, 310, ...] → 模型输出 token ID 3681 → 解码为 "Paris"

---

## 第11课 Recent Improvements（现代改进）

### 原始 Transformer vs 2024 时代 Transformer

| 方面 | 原始 (2017) | 现代 (2024) |
|------|------------|------------|
| 架构 | Encoder-Decoder | 主流为 Decoder-only |
| 位置编码 | 输入时一次性加入（静态） | 每层 Self-Attention 时加入（RoPE） |
| Layer Norm | 在 Attention/FFN 后面 | 在 Attention/FFN **前面**（Pre-Norm） |
| 注意力 | Full Multi-Head | Grouped Query Attention |
| 残差连接 | 有 | 有（更重要） |

### Positional Encoding（位置编码）

**为什么需要？** Transformer 并行处理所有 token，本身不知道词序。如果没有位置信息，"狗咬人"和"人咬狗"对模型来说一样。

**RoPE（Rotary Positional Embeddings）**：
- 在每个 Transformer Block 的 Self-Attention 层中，注入到 Q 和 K 向量
- 编码的是**相对位置**（token A 在 token B 前面 3 个位置）
- 支持 document packing（多个短文档打包进一个训练样本）

> **生动比喻**：静态位置编码像给每个座位固定编号（1号位、2号位...）；RoPE 像给每个人一个指南针，他们可以通过指南针的偏转角度知道彼此的相对距离。

### 训练数据打包（Document Packing）

- 训练时上下文窗口很大（如 16K），但很多文档很短
- **Naive**：一个文档一行，剩余空间用 padding 填充 → 浪费算力
- **Packed**：多个短文档拼接在一行 → 高效利用每个 GPU 周期

### Llama 3.1 8B 架构参数

| 参数 | 值 |
|------|-----|
| Layers | 32 |
| Model dimension | 4,096 |
| FFN dimension | 14,336 |
| Attention heads | 32 |
| KV heads (GQA) | 8 |
| Vocabulary | 128,000 |
| Position encoding | RoPE |

---

## 第12课 Mixture of Experts (MoE)

### 核心思想

![MoE 层结构](screenshots/moe_layer.jpg)

把 FFN 层从**一个大网络**变成**多个小网络（专家）**，每次只激活其中少数几个。

### 两大组件

![Router 路由机制](screenshots/moe_router.jpg)

1. **Experts（专家）**：多个独立的 FFN 网络
2. **Router（路由器）**：一个小分类器，决定每个 token 应该被哪个专家处理

> **生动比喻**：想象一个大医院的分诊台。
> - **Dense 模型**（传统）：每个病人都要看所有科室的医生
> - **MoE 模型**：分诊台（Router）先看看你的症状，然后把你分到最合适的专科医生（Expert）那里
> 
> 结果：每个病人只需要看 1-2 个专科，而不是跑遍所有科室。医院（模型）虽然有很多医生（参数），但每次看诊只需要几个人上班。

### 重要细节

- **Expert 不是领域专家**！它们不是"心理学专家"或"生物学专家"
- 更像是擅长处理不同**类型 token** 的专家（如标点、动词、名词等）
- 每一层独立路由，不同层可能选择不同的专家
- 通常选 top-2 专家，加权合并它们的输出

### 参数计算（以 Mixtral 8x7B 为例）

| 组件 | 参数量 |
|------|--------|
| 共享参数（Attention等） | ~1.2B |
| Router | ~32K（很小） |
| 8个 Expert × 5.6B | ~45B |
| **总参数（加载时）** | **~46B** |
| **活跃参数（推理时）** | **~12.9B**（共享 + 2个专家） |

### 优缺点

| 优点 | 缺点 |
|------|------|
| 推理时计算量小 | 加载模型需要大内存 |
| 性能通常高于同等活跃参数的 Dense 模型 | 训练更复杂 |
| 架构灵活 | 可能过度依赖单个专家（overfitting） |
| 不限于 Transformer（Mamba 等也可用） | — |

---

## 第13课 Conclusion

你现在掌握了 LLM Transformer 的完整知识链：

```
文字 → Tokenizer → Token Embedding → [Transformer Block × N] → LM Head → 下一个词
                                            ↕
                              Self-Attention + Feed-Forward NN
```

---

## 全课程知识架构总结

```
语言的数值表示演进：
  Bag-of-Words (稀疏, 无语义)
    → Word2Vec (稠密, 有语义, 但静态)
      → RNN + Attention (有上下文, 但不能并行)
        → Transformer (有上下文 + 可并行) ★

Transformer 家族：
  ├── Encoder-only (BERT) → 理解/表示
  ├── Decoder-only (GPT/Claude/Llama) → 生成 ★主流
  └── Encoder-Decoder (T5/原始Transformer) → 翻译等

Transformer Block 内部：
  ├── Self-Attention → 融合上下文信息
  │     ├── Query/Key/Value 矩阵
  │     ├── Multi-Head → 多视角
  │     ├── Grouped Query Attention → 效率优化
  │     └── Sparse Attention → 超长上下文
  ├── Feed-Forward Network → 存储知识
  │     └── MoE → 多专家，按需激活
  ├── Layer Normalization
  ├── Residual Connection（残差连接）
  └── Positional Encoding (RoPE)

推理优化：
  ├── KV Cache → 缓存已计算结果
  ├── Sparse Attention → 减少计算
  └── MoE → 只激活部分参数
```

---

## 核心概念速查表

| 概念 | 一句话解释 | 比喻 |
|------|-----------|------|
| Tokenizer | 把文字切成小块 | 裁缝的剪刀 |
| Embedding | 把 token 变成有意义的数字向量 | 给每个词一个"身份证" |
| Self-Attention | 让每个词看看句子里其他词和自己的关系 | 班级里的"八卦网络" |
| Query/Key/Value | 注意力计算的三个角色 | 图书馆找书：需求/书名/书的内容 |
| Multi-Head | 多组独立的注意力并行 | 侦探团队多人分工 |
| Feed-Forward Network | 存储和处理知识 | 一本大百科全书 |
| Masked Attention | 遮住未来，只看过去 | 考试时的挡板 |
| KV Cache | 缓存已算过的内容 | 做笔记避免重复计算 |
| MoE | 多个小专家，按需调用 | 医院分诊台 |
| RoPE | 在注意力中注入位置信息 | 每个人带的指南针 |
| Context Length | 模型一次能处理的最大 token 数 | 桌面大小（放不下就溢出） |
| Temperature | 控制生成的随机性 | 0=确定性选择, 高=更多创意 |
| Greedy Decoding | 总选概率最高的 token | 每步都选最安全的路 |
