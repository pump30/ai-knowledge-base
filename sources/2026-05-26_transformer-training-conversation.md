# Transformer 训练原理对话

> 来源：2026-05-26 与 Claude 的对话沉淀
> 主题：Transformer 训练机制、数据来源、Self-Attention vs Masked Self-Attention 训练方式

---

## 1. Attention Head 是不是训练出来的？

**Attention head 的"行为"完全是训练出来的，但"结构"是人设计的。**

### 结构（人设计的超参数）
- 有几个 head（8、12、16 等）
- 每个 head 的维度（d_model / num_heads）

### 参数（训练出来的）
每个 head 里的三个投影矩阵 W_Q、W_K、W_V（以及输出端 W_O）：
- 训练前：随机初始化（Xavier/Kaiming）
- 训练后：通过反向传播 + 梯度下降学到具体权重

不同 head 的"分工"（语法依赖、指代关系、位置邻近性）**不是人指定的，是训练中自发涌现的**。Anthropic 的 induction head 研究就是事后分析"这个 head 学到了什么"。

类比：
- 结构 = 大脑有多少个区域（先天）
- 参数 = 每个区域学会处理什么信息（后天训练）

---

## 2. 训练需要多少数据？

### 量级
| 模型 | 训练数据量 |
|------|-----------|
| GPT-2 (2019) | ~10B token |
| GPT-3 (2020) | ~300B token |
| LLaMA 2 (2023) | ~2T token |
| LLaMA 3 (2024) | ~15T token |
| GPT-4 / Claude 3+ | 估计 10T+ |

1T token ≈ 7500 亿英文单词 ≈ 把英文维基百科读 150 遍

### Chinchilla 法则
DeepMind 2022 发现：**训练数据量(token) ≈ 参数量 × 20** 是最优比例
- 7B 模型 → ~140B token
- 70B 模型 → ~1.4T token

数据太少 → 模型欠训练（参数浪费）
数据太多 → 边际收益递减（算力浪费）

### 例外
- 微调：几千到几万条数据
- LoRA / PEFT：只训部分参数，数据需求更少

---

## 3. 数据从哪里来？

### 公开网页爬取（60-80%）
- **Common Crawl** — 每月爬全网，PB 级数据
- **C4 / RefinedWeb / FineWeb** — Common Crawl 清洗版
- 直接爬特定网站

### 高质量精选语料
- Wikipedia（多语言）
- 书籍：Books3、Project Gutenberg、Anna's Archive（**版权争议最大**）
- 学术论文：arXiv、PubMed、Semantic Scholar
- 代码：GitHub（The Stack 数据集 6TB+）

### 社区/对话数据
- Reddit（API 现在收费）
- StackExchange / StackOverflow
- Twitter/X（受限）

### 专有/购买数据
- OpenAI 与 AP、Axel Springer、Reddit 签约
- Google 用 YouTube 字幕、Google Books
- 出版社授权图书

### 合成数据（增长最快）
- Phi 系列（微软）大量使用
- 解决"互联网数据快被用完"问题

### 人工标注（后训练用）
- RLHF：人类对回答打分
- SFT：人工写问答对

### 数据处理流程
```
爬取 → 去重 → 语言识别 → 质量过滤 → 去黄暴 →
去隐私(PII) → 去版权风险 → 分词 → 训练
```
清洗后通常只剩 **10-20%**。

### 当前热点问题
1. 数据快用完（高质量英文 2026-2028 耗尽）
2. 版权诉讼（NYT 告 OpenAI）
3. 数据污染（AI 生成内容污染互联网，可能"模型崩溃"）
4. 多模态数据成新增长点

---

## 4. 训练原理是什么？

### 核心：自监督学习
不需要人工标注，文本自己就是答案：
```
输入: "今天天气真"     目标: "好"
输入: "今天天气真好"   目标: "，"
```

### 训练循环（5 步）

**① Forward Pass**
输入 batch，模型输出每个位置对下一个 token 的概率分布

**② Loss（Cross-Entropy）**
`Loss = -log(P(正确答案))`
- 正确答案概率 35% → loss 中等
- 概率 99% → loss 接近 0

**③ Backward Pass**
链式法则计算每个参数的梯度（70B 模型每步算 700 亿个梯度）

**④ Optimizer Step（AdamW）**
`新参数 = 旧参数 - 学习率 × 梯度`

**⑤ 重复**
百万到亿次

### 关键技术细节
- **Batch**：几千到几百万 token 并行
- **Teacher Forcing**：训练用真实答案做输入，导致 exposure bias
- **Causal Mask**：第 N 个位置不能偷看后面 token（注意力矩阵右上三角设 -∞）
- **学习率调度**：Warmup + Decay
- **混合精度**：FP16/BF16 减半显存

### 训练阶段
| 阶段 | 数据 | 目标 | 算力占比 |
|------|------|------|---------|
| Pretraining | 万亿 token | 学语言+知识 | 95%+ |
| SFT | 几万条 QA | 学回答格式 | <5% |
| RLHF/DPO | 几十万偏好对 | 符合人类偏好 | <5% |

### 规模感（GPT-4 估算）
- ~25,000 张 A100 GPU
- 训练 90-100 天
- 成本 ~1 亿美元
- 处理 ~13T token

### 为什么能学到东西
模型参数容量大，但被强迫"压缩"几万亿 token 到这些参数里。要在有限参数下尽可能准确预测下一个词，模型别无选择，只能学会语法、事实、推理模式——**预测下一个词 = 被迫理解整个世界**。

---

## 5. Self-Attention vs Masked Self-Attention 训练方式

**核心结论：训练方式完全不一样。**

### Self-Attention → MLM（Masked Language Modeling）
BERT 的训练方式：
```
原始: 今天 天气 真 好 ， 我们 去 公园 玩
遮盖: 今天 [MASK] 真 好 ， 我们 去 [MASK] 玩
任务: 预测被遮盖的词
```
- 随机遮盖 ~15% token
- 双向看上下文
- 每个样本只产生少数预测目标
- 训练效率低（85% 位置不贡献 loss）

BERT 的 80/10/10 策略：80% 替换为 [MASK]、10% 随机词、10% 保留原词，防止训练-推理 gap。

### Masked Self-Attention → Next-Token Prediction
GPT/Claude 的训练方式：
```
输入:    今 天  天  气  真
目标:       天  天  气  真  好
```
- 每个位置都是预测目标
- 通过 causal mask 保证单向
- 一个序列产生 N-1 个训练信号
- 训练效率高

### Mask 命名陷阱
**两个 mask 完全是两回事：**
- **MLM 里的 "Masked"** = 数据层面把词挖空
- **Masked Self-Attention 里的 "Masked"** = 计算层面把注意力矩阵右上三角设为 -∞

### 训练效率对比（序列长度 1024）
| | BERT (MLM) | GPT (Next-token) |
|---|---|---|
| 每序列预测数 | ~150 | 1023 |
| 训练效率 | 低 | **高 6-7 倍** |

这也是 decoder-only 架构最终赢的原因——同样算力下学得更快、scale 更好。

---

## 关键洞察

1. **Attention head 的功能是涌现的，不是设计的** — 这是 LLM 可解释性研究的核心难点
2. **预测下一个词 = 被迫理解世界** — 这是 LLM 涌现智能的根本原理
3. **Decoder-only 赢在训练效率** — 6-7 倍效率差距决定了架构演进方向
4. **数据问题正在成为瓶颈** — 高质量数据有限，合成数据是未来方向

---

## 6. Tokenizer 是训练出来的还是预定义的？

**答案：词表是训练出来的，但训练算法是人设计的。** 三层区分：

| 层级 | 例子 | 谁决定 |
|------|------|--------|
| 算法 | BPE/WordPiece/SentencePiece/Unigram | 人设计 |
| 超参数 | 词表大小、特殊 token | 人配置 |
| 词表内容 | 具体哪些 subword 进词表 | **训练出来的** |

### Tokenizer 训练 ≠ 模型训练
两个独立训练过程：
- Tokenizer 训练（几小时）：语料 → vocab.json + merges.txt
- 模型训练（几个月）：用切好的 token IDs 训练权重
Tokenizer 训完后冻结。

### BPE 训练流程
```
初始: 词表 = 所有单字符
迭代: 找最高频相邻字符对 → 合并为新 token
重复: 直到达到目标词表大小
```

### 主流模型 Tokenizer
- GPT-4: BPE (cl100k_base), 100K
- Llama 3: TikToken (BPE), 128K（从 Llama 2 的 32K 扩展，为更好编码非英语和代码）
- BERT: WordPiece, 30K
- Claude: 类似 BPE（未公开），~100K

### 关键洞察
- Tokenizer 是模型的"输入语言"，决定能看到什么粒度
- 换 tokenizer = 换模型（embedding 全失效）
- 中文/CJK 用 SentencePiece 直接在字节上训练，无需预分词
- Glitch Tokens 现象：词表中"误收纳"的训练不充分 token 会让模型行为异常

---

## 7. FFN 是怎么训练的？

**核心结论：FFN 没有独立训练过程**——和 Attention、Embedding 一起端到端训练，普通反向传播，无特殊算法。

### 训练中的角色分化（涌现）
随训练进行，两个 Linear 层自发分化：
- W_up 每行 → 模式探测器
- W_down 每列 → 知识响应

事实知识"巴黎是法国首都"就这样逐步编码到 FFN 权重里（Geva et al. 2021 键值存储理论）。

### FFN 训练的特殊性
| 维度 | 表现 |
|------|------|
| 梯度 | 比 Attention 平滑稳定 |
| 学习速度 | 慢，事实知识需反复看几千次 |
| 遗忘难度 | 最难被覆盖（微调改风格易、改事实难） |
| 计算占比 | 反向传播 ~60-70% 在 FFN |
| 显存占比 | 中间激活值占大头 |
| 正则化 | Dropout 主要加在 FFN |

### 训练故事
"Paris is the capital of" → 第 100 步预测 France 1% → 100K 步 30% → 10M 步 95%。

### MoE 训练特殊挑战
- Router 不可微 → top-k 加权 + Straight-Through Estimator
- 负载不均 → auxiliary load balance loss
- Expert Capacity → 超量 token 被丢弃
