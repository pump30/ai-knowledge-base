# Attention in Transformers: Concepts and Code in PyTorch - 完整课程笔记

> 课程来源：DeepLearning.AI  
> 讲师：Josh Starmer (StatQuest) & Andrew Ng  
> 主题：Transformer 中注意力机制的原理与 PyTorch 实现

---

## 第1课 Introduction（课程介绍）

### 核心要点

- 注意力机制（Attention）是最终催生 Transformer 架构的关键技术突破
- Transformer 架构和注意力算法对大语言模型的发展至关重要
- 本课程覆盖三种注意力类型：Self-Attention、Masked Self-Attention、Encoder-Decoder Attention

### 注意力机制的历史

![编码器-解码器历史](screenshots/01_encoder_decoder_history.jpg)

**2014年**：Yoshua Bengio（蒙特利尔大学）和 Chris Manning（斯坦福大学）的团队独立提出了注意力机制，用于机器翻译任务。

**核心问题**：翻译时，不同语言的词序不同，句子长度也不同。例如：
- 英语 "the European Economic Area was..." 翻译为法语时词序改变
- 英语 "They arrived late"（3词）→ 法语需要5个词

**早期方案**：Encoder-Decoder 架构
- Encoder 逐词读取输入，产出每个词的向量表示
- 这些向量是**上下文嵌入**（contextual embeddings）——向量不仅取决于词本身，还取决于周围的词
- Decoder 利用这些向量生成输出，并通过**注意力权重**决定关注输入的哪个部分

### "Attention is All You Need"（2017）

![Attention is All You Need 论文](screenshots/01_attention_is_all_you_need.jpg)

- 来自 Google Brain 团队的开创性论文
- 引入了 Transformer 架构和更通用的注意力形式
- 专门设计为可在 GPU 上高度并行扩展
- 原始论文仅使用6层注意力，而现代模型如 Llama 3.2-405B 使用126层

### 后续影响

| 架构 | 来源 | 代表模型 | 用途 |
|------|------|---------|------|
| Encoder (BERT) | Transformer 的编码器部分 | 各类 Embedding 模型 | RAG、推荐系统 |
| Decoder (GPT) | Transformer 的解码器部分 | ChatGPT、Claude、Llama | 文本生成 |

### 课程大纲

![课程大纲](screenshots/01_course_outline.jpg)

1. Transformer 和 Attention 的核心思想
2. Self-Attention 的矩阵数学和代码
3. Self-Attention vs Masked Self-Attention
4. Masked Self-Attention 的矩阵数学和代码
5. Encoder-Decoder Attention 和 Multi-Head Attention

---

## 第2课 The Main Ideas Behind Transformers and Attention（Transformer 和注意力的核心思想）

### Transformer 的三大基础构件

![Transformer 三大组件](screenshots/02_transformer_three_parts.jpg)

| 组件 | 功能 | 作用 |
|------|------|------|
| Word Embedding（词嵌入） | 将词/token 转换为数字 | 神经网络只能处理数字输入 |
| Positional Encoding（位置编码） | 跟踪词序 | 区分 "狗咬人" 和 "人咬狗" |
| Attention（注意力） | 建立词与词之间的关系 | 正确理解代词指代等语义关系 |

> **生动比喻**：想象你在读一本外文小说。
> - **Word Embedding** = 查字典，把每个外文词翻译成你能理解的数字代码
> - **Positional Encoding** = 给每个词标上"第几个出现"的顺序号
> - **Attention** = 读到代词"他"时，回头看看前文谁最可能是"他"

### Self-Attention（自注意力）的直觉

![Self-Attention 相似度计算](screenshots/02_self_attention_similarity.jpg)

**示例句子**："The pizza came out of the oven and it tasted good"

- "it" 可能指 "pizza" 也可能指 "oven"
- Self-Attention 计算每个词与所有其他词（包括自身）的**相似度**
- 如果训练数据中 "it" 更常与 "pizza" 关联，则 "pizza" 的相似度分数更高
- 最终 "pizza" 对 "it" 的编码产生更大影响

> **生动比喻**：Self-Attention 就像一个班级里的"关系网"。每个学生（词）都环顾四周，看看和其他同学的亲密程度——是闺蜜？是同桌？还是仅仅认识？亲密度越高（相似度越大），对方对你的影响越大。

---

## 第3课 The Matrix Math for Calculating Self-Attention（Self-Attention 的矩阵数学）

### Self-Attention 方程

![Self-Attention 方程](screenshots/03_self_attention_equation.jpg)

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

看起来复杂，但可以拆成几个简单步骤。

### Query、Key、Value 的来源

![QKV 矩阵计算](screenshots/03_qkv_matrices.jpg)

**数据库类比**：

| 术语 | 数据库含义 | 注意力中的含义 |
|------|-----------|--------------|
| Query（查询） | 搜索词 | 当前词在"问"什么 |
| Key（键） | 数据库中的索引名 | 每个词的"身份标签" |
| Value（值） | 数据库返回的结果 | 每个词的实际信息 |

> **生动比喻**：想象你住酒店。你告诉前台你姓"张"（Query），前台在电脑里搜所有客人的姓名（Keys），找到最匹配的那个，然后告诉你房间号（Value）。Self-Attention 里每个词都在做类似的事——用自己的 Query 去匹配所有词的 Key，然后获取对应的 Value。

### 计算步骤详解

**以 "Write a poem" 为例**（每个词用2个数字表示）：

**第一步：生成 Q、K、V 矩阵**

```
编码矩阵 × 权重矩阵_Q^T = Q (查询矩阵)
编码矩阵 × 权重矩阵_K^T = K (键矩阵)  
编码矩阵 × 权重矩阵_V^T = V (值矩阵)
```

- 权重矩阵的大小由 d_model 决定（示例中为 2x2）
- 实际应用中常见 512x512 的权重矩阵

**第二步：计算 Q × K^T（点积相似度）**

- 得到所有 Query-Key 对的**未缩放点积相似度**
- 点积是衡量两个向量相似程度的方法（与余弦相似度密切相关）
- 结果矩阵中每个值表示：某个词的 Query 与另一个词的 Key 有多相似

**第三步：除以 sqrt(d_k) 进行缩放**

- d_k = 每个 key 的维度数（示例中为2）
- 原始论文作者发现缩放能改善性能
- 防止点积值过大导致 softmax 梯度消失

**第四步：对每行取 Softmax**

![Softmax 百分比结果](screenshots/03_softmax_percentages.jpg)

- 每行加和等于1（转化为百分比）
- 表示每个词对其他词的"关注程度"
- 例如："write" 对自己的关注度36%，对 "a" 的关注度40%，对 "poem" 的关注度24%

**第五步：百分比矩阵 × V**

- 用百分比来加权 Value 矩阵
- 最终得到每个词的 Self-Attention 分数
- 含义：每个词的最终编码是所有词的 Value 的**加权平均**，权重由相似度决定

> **生动比喻**：想象做一杯混合果汁。每种水果的添加量由它和你的"口味偏好"（Query-Key 相似度）决定。你最喜欢苹果（40%），还行的是橘子（36%），不太喜欢柠檬（24%）——最终果汁的味道（Attention输出）就是按这些比例混合出来的。

---

## 第4课 Coding Self-Attention in PyTorch（PyTorch 实现 Self-Attention）

### 完整代码结构

![Self-Attention 类代码](screenshots/04_self_attention_class.jpg)

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class SelfAttention(nn.Module):
    def __init__(self, d_model, row_dim=0, col_dim=1):
        super().__init__()
        
        # 创建权重矩阵（用 nn.Linear 自动管理）
        self.W_q = nn.Linear(in_features=d_model, out_features=d_model, bias=False)
        self.W_k = nn.Linear(in_features=d_model, out_features=d_model, bias=False)
        self.W_v = nn.Linear(in_features=d_model, out_features=d_model, bias=False)
        
        self.row_dim = row_dim
        self.col_dim = col_dim
    
    def forward(self, token_encodings):
        # 计算 Q, K, V
        q = self.W_q(token_encodings)
        k = self.W_k(token_encodings)
        v = self.W_v(token_encodings)
        
        # 计算注意力
        similarities = torch.matmul(q, k.transpose(self.row_dim, self.col_dim))
        scaled = similarities / torch.sqrt(torch.tensor(k.size(self.col_dim), dtype=torch.float32))
        attention_percents = F.softmax(scaled, dim=self.col_dim)
        attention_scores = torch.matmul(attention_percents, v)
        
        return attention_scores
```

### 关键代码解读

| 代码 | 作用 |
|------|------|
| `nn.Linear(d_model, d_model, bias=False)` | 创建权重矩阵，不加偏置（遵循原始论文） |
| `torch.matmul(q, k.transpose(...))` | Q 乘以 K 的转置，得到相似度矩阵 |
| `/ torch.sqrt(...)` | 按 sqrt(d_k) 缩放 |
| `F.softmax(scaled, dim=col_dim)` | 对每行取 softmax，转为百分比 |
| `torch.matmul(attention_percents, v)` | 用百分比加权 V，得最终注意力分数 |

### 验证方式

- 可通过 `self_attention.W_q.weight.T` 提取权重矩阵
- 手动计算验证结果是否与类输出一致

---

## 第5课 Self-Attention vs Masked Self-Attention（自注意力 vs 掩码自注意力）

### 词嵌入（Word Embedding）深入理解

课程首先深入解释了词嵌入的训练过程：

1. 给每个唯一词分配随机数（初始嵌入）
2. 训练一个简单网络：让每个词预测下一个词
3. 训练后，语义相似的词（如 "great" 和 "awesome"）会聚类到一起

> **生动比喻**：想象你新到一个城市，不认识路。一开始，你给每条街道随机贴标签。但随着你走得多了，你发现"火锅街"和"烧烤路"经常同时出现在美食区——于是在你的心理地图中，它们越来越靠近。词嵌入的训练就是这样一个"画心理地图"的过程。

### Self-Attention 创建上下文感知嵌入

![上下文感知嵌入](screenshots/05_context_aware_embeddings.jpg)

**普通词嵌入**：只聚类单个词（"great" 和 "awesome" 靠近）

**上下文感知嵌入**（Context-Aware Embeddings）：
- 通过 Self-Attention + Positional Encoding 创建
- 可以聚类**相似的句子**甚至**相似的文档**
- 同一个词在不同上下文中得到不同的嵌入

### 核心区别

![Self-Attention vs Masked Self-Attention](screenshots/05_self_vs_masked.jpg)

| 特性 | Self-Attention | Masked Self-Attention |
|------|---------------|----------------------|
| 看哪些词 | 前面和后面**所有**词 | 只看**前面**的词（包括自身） |
| 对应架构 | Encoder-Only Transformer | Decoder-Only Transformer |
| 代表模型 | BERT | GPT / ChatGPT |
| 典型用途 | 文本分类、语义搜索、NER | 文本生成 |
| 输出 | 上下文感知嵌入 | 生成性输入 |

### Encoder-Only Transformer 的应用

- 聚类相似句子/文档
- 作为分类模型的输入（如情感分析）
- 作为逻辑回归模型的变量

### Decoder-Only Transformer 为何能生成文本

- 使用 Masked Self-Attention，永远不能"偷看"后面的词
- 训练时：给模型句子前半部分，让它生成后半部分
- 这就是 ChatGPT 被称为"生成式模型"的原因

> **生动比喻**：
> - **Self-Attention（编码器）** 像一个读完整本侦探小说的读者——他知道凶手是谁，可以回头审视每一条线索。
> - **Masked Self-Attention（解码器）** 像一个正在写侦探小说的作家——他只能基于已写好的内容来决定下一句话，不能"穿越"到结尾去看答案。

---

## 第6课 The Matrix Math for Calculating Masked Self-Attention（Masked Self-Attention 的矩阵数学）

### 与 Self-Attention 的唯一区别

Masked Self-Attention 的公式只多了一个**掩码矩阵 M**：

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}} + M\right)V$$

### 掩码矩阵的构造

![掩码矩阵](screenshots/06_mask_matrix.jpg)

以 "Write a poem" 为例，掩码规则：
- "Write"（第1个词）：只能看自己 → 遮住 "a" 和 "poem"
- "A"（第2个词）：能看自己和 "write" → 遮住 "poem"
- "Poem"（第3个词）：能看所有词 → 不遮挡

掩码矩阵 M：
```
[  0,    -inf,  -inf ]
[  0,     0,    -inf ]
[  0,     0,      0  ]
```

- **加0**：不改变原始相似度值 → 保留该位置的注意力
- **加负无穷**：使该位置的值变为负无穷 → softmax 后变为0%

### Softmax 后的效果

![Masked Softmax 结果](screenshots/06_masked_softmax_result.jpg)

- "Write" 对自己的注意力 = 100%，对其他词 = 0%
- "A" 对 "poem" 的注意力 = 0%
- "Poem" 对所有词都有正常的注意力分配

> **生动比喻**：掩码就像一个不断增大的"阅读窗口"。
> - 写第一个词时，你只能看空白纸（只有自己）
> - 写第二个词时，你能回头看第一个词
> - 写第三个词时，你能看前面所有已写的内容
> 
> 这就像考试时老师一次只发一道题，你不能提前看后面的题目。

---

## 第7课 Coding Masked Self-Attention in PyTorch（PyTorch 实现 Masked Self-Attention）

### 代码结构

![Masked Self-Attention 类](screenshots/07_masked_attention_class.jpg)

```python
class MaskedSelfAttention(nn.Module):
    def __init__(self, d_model, row_dim=0, col_dim=1):
        super().__init__()
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.row_dim = row_dim
        self.col_dim = col_dim
    
    def forward(self, token_encodings, mask=None):
        q = self.W_q(token_encodings)
        k = self.W_k(token_encodings)
        v = self.W_v(token_encodings)
        
        similarities = torch.matmul(q, k.transpose(self.row_dim, self.col_dim))
        scaled = similarities / torch.sqrt(torch.tensor(k.size(self.col_dim), dtype=torch.float32))
        
        # 关键区别：添加掩码
        if mask is not None:
            scaled = scaled.masked_fill(mask, -1e9)
        
        attention_percents = F.softmax(scaled, dim=self.col_dim)
        attention_scores = torch.matmul(attention_percents, v)
        
        return attention_scores
```

### 创建掩码的方法

```python
# 创建下三角掩码
mask = torch.ones(3, 3)          # 3x3 全1矩阵
mask = torch.tril(mask)          # 保留下三角，上三角变0
mask = (mask == 0)               # 0变True（需遮挡），1变False（保留）
```

### 设计亮点

- `mask=None` 作为默认参数 → 同一个类既能做 Self-Attention 又能做 Masked Self-Attention
- 使用 `masked_fill` 方法：True 位置填充 -1e9（近似负无穷），False 位置填充 0
- 加到 scaled similarities 上后，softmax 会将 -1e9 对应位置变为近似 0%

---

## 第8课 Encoder-Decoder Attention（编码器-解码器注意力）

### 三种 Transformer 架构的来龙去脉

![Encoder-Decoder 架构](screenshots/08_encoder_decoder_architecture.jpg)

**历史脉络**：

1. **最初的 Transformer**（2017）：同时有 Encoder 和 Decoder
   - Encoder 用 Self-Attention
   - Decoder 用 Masked Self-Attention
   - 两者通过 Encoder-Decoder Attention 连接

2. **Encoder-Only**（如 BERT）：人们发现只用编码器就能做很多事

3. **Decoder-Only**（如 GPT）：人们发现只用解码器也能生成文本（包括翻译）

### Encoder-Decoder Attention 的特殊之处

| 组件 | 来源 |
|------|------|
| Keys (K) | 来自 **Encoder** 的输出 |
| Values (V) | 来自 **Encoder** 的输出 |
| Queries (Q) | 来自 **Decoder** 的 Masked Self-Attention 输出 |

计算方式与普通 Self-Attention 完全相同（使用所有相似度，不加掩码），唯一的区别是 Q、K、V 来自不同的源。

### 应用场景

![Cross-Attention 在多模态中的应用](screenshots/08_cross_attention.jpg)

- **经典用途**：机器翻译（英语→西班牙语）
- **现代用途**：多模态模型
  - 图像/音频 Encoder → 生成上下文感知嵌入
  - 通过 Encoder-Decoder Attention 馈入文本 Decoder
  - 实现图像描述生成、音频问答等功能

> **生动比喻**：Encoder-Decoder Attention 就像一个同声传译员。
> - **Encoder**（听众）：听完一整段英语演讲，理解了每个词的含义和上下文
> - **Decoder**（翻译员）：在翻译过程中，不断"回头看"听到的内容（通过注意力），确定当前翻译哪个部分最相关
> - **Cross-Attention** 就是翻译员"回头看原文"的那个动作

---

## 第9课 Multi-Head Attention（多头注意力）

### 为什么需要多个注意力头？

![Multi-Head Attention 概念](screenshots/09_multi_head_attention.jpg)

- 简单句子中，单个注意力头足够
- 复杂句子/长段落中，词与词之间存在**多种类型的关系**
- 多个注意力头可以**同时、独立**地捕获不同类型的关系

### Multi-Head Attention 的结构

- 每个 Head 有**独立的**权重矩阵 W_q、W_k、W_v
- 所有 Head 接收相同的输入，但因为权重不同，关注点不同
- 原始 Transformer 论文使用 **8个 Head**

### 输出维度问题

![Multi-Head 全连接层](screenshots/09_multi_head_fc_layer.jpg)

问题：多个 Head 的输出拼接后维度变大（如 3个Head × 2个值 = 6个值）

解决方案两种：

| 方案 | 方法 | 优点 |
|------|------|------|
| 全连接层 | 所有 Head 输出接入一个 FC 层，压缩回原始维度 | 灵活 |
| 调整 Value 矩阵形状 | 减少每个 Head 输出的值数量 | 更高效 |

> **生动比喻**：Multi-Head Attention 就像一个审稿委员会。
> - **Head 1** 可能专注于语法关系（主谓宾）
> - **Head 2** 可能专注于语义关系（代词指代）
> - **Head 3** 可能专注于位置关系（相邻词的搭配）
> 
> 每个委员独立审稿，最后把意见汇总（concatenate + FC layer），形成最终的"综合审稿意见"。

---

## 第10课 Coding Encoder-Decoder Attention and Multi-Head Attention in PyTorch

### 统一注意力类（支持三种注意力）

```python
class Attention(nn.Module):
    def __init__(self, d_model, row_dim=0, col_dim=1):
        super().__init__()
        self.W_q = nn.Linear(d_model, d_model, bias=False)
        self.W_k = nn.Linear(d_model, d_model, bias=False)
        self.W_v = nn.Linear(d_model, d_model, bias=False)
        self.row_dim = row_dim
        self.col_dim = col_dim
    
    def forward(self, encodings_for_q, encodings_for_k, encodings_for_v, mask=None):
        # 关键变化：Q、K、V 可以来自不同的输入
        q = self.W_q(encodings_for_q)
        k = self.W_k(encodings_for_k)
        v = self.W_v(encodings_for_v)
        
        similarities = torch.matmul(q, k.transpose(self.row_dim, self.col_dim))
        scaled = similarities / torch.sqrt(torch.tensor(k.size(self.col_dim), dtype=torch.float32))
        
        if mask is not None:
            scaled = scaled.masked_fill(mask, -1e9)
        
        attention_percents = F.softmax(scaled, dim=self.col_dim)
        attention_scores = torch.matmul(attention_percents, v)
        
        return attention_scores
```

**三种注意力的调用方式**：

| 类型 | 调用方式 |
|------|---------|
| Self-Attention | `attention(x, x, x)` — Q、K、V 都来自同一输入 |
| Masked Self-Attention | `attention(x, x, x, mask=mask)` — 加掩码 |
| Encoder-Decoder Attention | `attention(decoder_out, encoder_out, encoder_out)` — Q 来自 Decoder，K/V 来自 Encoder |

### Multi-Head Attention 类

![Multi-Head Attention 类代码](screenshots/10_multi_head_class.jpg)

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, d_model, num_heads, row_dim=0, col_dim=1):
        super().__init__()
        # 创建 num_heads 个独立的 Attention 对象
        self.heads = nn.ModuleList([
            Attention(d_model, row_dim, col_dim)
            for _ in range(num_heads)
        ])
        self.col_dim = col_dim
    
    def forward(self, encodings_for_q, encodings_for_k, encodings_for_v, mask=None):
        # 每个 Head 独立计算，然后拼接
        return torch.cat([
            head(encodings_for_q, encodings_for_k, encodings_for_v, mask)
            for head in self.heads
        ], dim=self.col_dim)
```

**关键设计点**：
- 使用 `nn.ModuleList` 存储多个 Head（确保参数被正确注册）
- 每个 Head 共享相同的 d_model、row_dim、col_dim
- 各 Head 的输出通过 `torch.cat` 在列维度上拼接
- num_heads=1 时结果与单个 Attention 完全一致
- num_heads=2 时输出维度翻倍

---

## 第11课 Conclusion（课程总结）

### 本课程你学到了：

1. **三种注意力类型**：Self-Attention、Masked Self-Attention、Encoder-Decoder Attention
2. **深入理解**了 Self-Attention vs Masked Self-Attention 的优劣势及适用场景
3. **用 PyTorch 编码**了所有三种注意力类型并验证了计算正确性
4. **Multi-Head Attention** 的原理和实现

---

## 全课程知识架构总结

```
Attention in Transformers
├── 基础概念
│   ├── Word Embedding（词嵌入）→ 词 → 数字向量
│   ├── Positional Encoding（位置编码）→ 保留词序信息
│   └── Attention（注意力）→ 建立词间关系
│
├── Self-Attention（自注意力）
│   ├── 核心公式：softmax(QK^T / sqrt(d_k)) × V
│   ├── Q/K/V 由输入 × 权重矩阵生成
│   ├── 点积 → 缩放 → Softmax → 加权求和
│   ├── 所有词可看前后所有词
│   ├── 产出：上下文感知嵌入
│   └── 应用：Encoder-Only (BERT) → 分类、搜索、聚类
│
├── Masked Self-Attention（掩码自注意力）
│   ├── 公式在 Self-Attention 基础上 + 掩码矩阵 M
│   ├── M：下三角为 0，上三角为 -inf
│   ├── 每个词只能看到之前的词
│   ├── 产出：生成性输入
│   └── 应用：Decoder-Only (GPT/ChatGPT) → 文本生成
│
├── Encoder-Decoder Attention（编码器-解码器注意力 / Cross-Attention）
│   ├── Q 来自 Decoder，K/V 来自 Encoder
│   ├── 计算方式同 Self-Attention（无掩码）
│   ├── 连接两个模块的桥梁
│   └── 应用：翻译、多模态模型
│
└── Multi-Head Attention（多头注意力）
    ├── 多个独立的 Attention Head 并行计算
    ├── 每个 Head 有独立权重，关注不同类型关系
    ├── 输出拼接后可接 FC 层降维
    └── 原始论文用 8 个 Head
```

---

## 核心概念速查表

| 概念 | 一句话解释 | 比喻 |
|------|-----------|------|
| Self-Attention | 每个词看所有词（前后）的相似度来编码自己 | 班级关系网，环顾四周看谁和自己最亲 |
| Masked Self-Attention | 每个词只看前面的词来编码自己 | 写作时只能回顾已写内容，不能偷看结尾 |
| Encoder-Decoder Attention | Q 来自 Decoder，K/V 来自 Encoder | 同声传译：翻译时回头看原文 |
| Multi-Head Attention | 多组独立注意力并行工作 | 审稿委员会多人分工审稿 |
| Query (Q) | 当前词的"搜索请求" | 你在图书馆想找什么类型的书 |
| Key (K) | 每个词的"身份标签" | 书架上每本书的标题 |
| Value (V) | 每个词携带的实际信息 | 书里面的具体内容 |
| Dot Product | 衡量两个向量的相似程度 | 两个人的兴趣重合程度 |
| Softmax | 将任意数字转为概率分布(和为1) | 把打分转换为"百分比" |
| sqrt(d_k) 缩放 | 防止点积值过大 | 给过热的引擎降温 |
| 掩码矩阵 (-inf) | 遮挡不该看到的未来位置 | 考试时的答案挡板 |
| Context-Aware Embedding | 融合了上下文信息的词向量 | 根据语境理解一词多义 |
| Encoder-Only (BERT) | 只用 Self-Attention 的 Transformer | 全文阅读理解专家 |
| Decoder-Only (GPT) | 只用 Masked Self-Attention 的 Transformer | 创意写作专家 |
| nn.Linear | PyTorch 中创建权重矩阵的工具 | 一个可训练的"变换器" |
| nn.ModuleList | 存储多个子模块的列表 | 工具箱里的多把工具 |

---

## PyTorch 代码要点总结

| 操作 | PyTorch 代码 | 作用 |
|------|-------------|------|
| 创建权重矩阵 | `nn.Linear(d_model, d_model, bias=False)` | 初始化 Q/K/V 的可训练权重 |
| 矩阵乘法 | `torch.matmul(a, b)` | 计算 QK^T 和 attention×V |
| 转置 | `k.transpose(dim0, dim1)` | 转置 K 矩阵 |
| 缩放 | `/ torch.sqrt(tensor)` | 除以 sqrt(d_k) |
| Softmax | `F.softmax(x, dim=col_dim)` | 转为概率分布 |
| 掩码填充 | `x.masked_fill(mask, -1e9)` | 用极小值填充被遮挡位置 |
| 创建掩码 | `torch.tril(torch.ones(n,n))` | 下三角矩阵 |
| 拼接张量 | `torch.cat([...], dim=col_dim)` | 多 Head 输出拼接 |
| 模块列表 | `nn.ModuleList([...])` | 存储多个注意力 Head |
