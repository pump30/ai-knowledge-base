# Embedding Models: from Architecture to Implementation 学习笔记

> 课程来源：DeepLearning.AI（与 Vectara 合作）  
> 讲师：Andrew Ng, Amin Ahmad (Vectara联合创始人), Ofer Mendelevitch (Vectara)

---

## 第1课：Introduction（引言）

### 课程概述
- 嵌入模型（Embedding Models）创建的向量使得**语义检索系统**成为可能
- 本课程是技术深度课程，聚焦于构建模块而非应用层
- 课程路径：Word2Vec -> GloVe -> Transformer -> BERT -> Dual Encoder -> RAG

### 嵌入向量发展史
| 时间 | 模型/技术 | 贡献 |
|------|-----------|------|
| 2013 | Word2Vec | 首次用周围词预测目标词，学习词向量 |
| 2014 | GloVe | 简化数学，改进Word2Vec |
| 2017 | Transformer ("Attention is All You Need") | 前馈网络高效处理序列数据 |
| 2018 | BERT | 深度Transformer填空任务，开启现代NLP |
| 2018+ | GPT系列 | Decoder-only架构的LLM |

### 生动比喻
> 嵌入向量就像给每个词/句子分配一个**GPS坐标**——意思相近的词/句子在"语义地图"上距离很近，意思不同的则相距很远。

![课程概览](screenshots/01_course_overview.jpg)
![Word2Vec历史](screenshots/01_word2vec_history.jpg)

---

## 第2课：Introduction to Embedding Models（嵌入模型概述）

### 向量嵌入基础
- 将现实世界实体（词、句子、图像）映射为向量空间中的点
- 核心特性：**语义相似的实体在向量空间中距离相近**
- Word2Vec经典示例：queen - woman + man = king
- Star Wars示例：Yoda - good + evil = Vader

### 词嵌入 vs 句子嵌入
- **词嵌入（Token Embedding）**：单个词/token的向量表示
- **句子嵌入（Sentence Embedding）**：整个句子的向量表示，用于语义搜索

### 嵌入向量的应用
1. **LLM内部**：Transformer中的Token Embedding层
2. **语义搜索**：RAG pipeline中的检索引擎
3. **推荐系统**：基于向量相似度的产品推荐
4. **异常检测**：在嵌入空间中检测离群点

### RAG中的检索方法对比

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| Cross Encoder | 将Q+A拼接输入BERT分类 | 更准确 | 极慢，需逐一对比 |
| Sentence Embedding | 预计算嵌入 + 相似度搜索 | 快速，可扩展 | 略低于Cross Encoder |

### 生动比喻
> Cross Encoder就像**一对一面试**——准确但慢，每个候选人都要和面试官坐下来谈半小时。Sentence Embedding就像**简历筛选**——先看简历关键词（预计算向量），快速筛出TOP候选人，虽不如面试精准但效率极高。

![Word2Vec类比运算](screenshots/02_word2vec_analogy.jpg)
![RAG Pipeline](screenshots/02_rag_pipeline.jpg)
![Cross Encoder vs Bi-Encoder](screenshots/02_cross_encoder_vs_biencoder.jpg)

---

## 第3课：Contextualized Token Embeddings（上下文化Token嵌入）

### 静态词向量的问题
- Word2Vec/GloVe给每个词一个**固定**向量
- 无法处理一词多义：如"bat"（球棒 vs 蝙蝠）在两种语境下向量完全相同
- 需要上下文化（Contextualized）的嵌入

### Transformer架构解决方案
- **Encoder**：双向注意力，同时关注左右两侧的token -> 输出上下文化向量
- **Decoder**：单向注意力，只能关注左侧已生成的token
- GPT = Decoder-only（用于生成）
- BERT = Encoder-only（用于理解和嵌入）

### BERT模型详解
- **BERT Base**：12层Transformer，1.1亿参数
- **BERT Large**：24层Transformer，3.4亿参数
- 预训练数据：33亿词

### BERT的预训练任务
1. **MLM（Masked Language Modeling）**：随机遮住15%的词，让模型预测被遮住的词
   - 这是模型学习上下文化向量的核心任务
2. **NSP（Next Sentence Prediction）**：判断句子B是否紧跟句子A
   - 帮助模型理解句子间关系

### Cross Encoder应用
- 将BERT微调为分类器：输入 = [CLS] + 句子A + [SEP] + 句子B
- 输出相似度分数
- 可用于问答检索（如MS-MARCO数据集）

### 代码验证
- "bat"在两个不同语境下的BERT嵌入余弦相似度只有0.45（说明BERT确实能区分上下文）
- 而GloVe中同一个词的嵌入永远相同（余弦相似度 = 1）

### 生动比喻
> Word2Vec就像**字典**——查"苹果"永远是同一个解释。BERT就像**有语境感知的翻译官**——同样是"苹果"，在"吃苹果"和"苹果手机"中给出完全不同的理解。MLM训练就像**完形填空**——通过大量做"填空题"，模型学会了根据上下文理解每个词。

![Transformer架构](screenshots/03_transformer_architecture.jpg)
![BERT模型](screenshots/03_bert_model.jpg)
![BERT MLM训练](screenshots/03_bert_mlm.jpg)

---

## 第4课：Token vs. Sentence Embedding（Token嵌入 vs 句子嵌入）

### 从Token嵌入到句子嵌入的挑战
- **朴素方法1**：对BERT最后一层所有token嵌入做平均池化（Mean Pooling）-> **失败**
- **朴素方法2**：直接使用CLS token的嵌入 -> **失败**
- 原因：未经句子级训练的BERT嵌入，所有句子的相似度都很高，无法区分

### 实验证据
- Mean Pooling热力图：所有句子之间相似度都是红色（高），毫无区分度
- STS基准测试：Mean Pooling与人类标注的Pearson相关系数极低
- 预训练句子模型（如all-MiniLM-L6-v2）：热力图清晰区分相似/不相似，Pearson相关系数高

### 句子嵌入研究历史
- 2018: Universal Sentence Encoder (Google)
- 2019: SBERT（首个基于句子对训练的模型）
- 后续: DPR, Sentence-T5, E5, ColBERT 等

### 纯相似度 vs 问答检索的区别
- **纯相似度**：找语义最相似的句子（问题 = 问题的复述）
- **问答检索**：找能回答问题的句子（问题 -> 答案）
- RAG需要的是后者！

### Dual Encoder（双编码器）架构
- **Question Encoder**：编码问题
- **Answer Encoder**：编码答案/文档
- 两个独立编码器，用对比损失训练
- 确保"相关问答对"在向量空间中距离近

### 生动比喻
> Mean Pooling失败就像**把一本书所有页的文字堆在一起取平均**——得到的是一锅粥，而非书的摘要。真正有效的句子嵌入需要**专门训练模型去理解"什么是相似"**。
>
> Dual Encoder就像培训两个**专业配对员**——一个专门理解问题（"你想找什么？"），另一个专门理解答案（"这个能满足你吗？"），两人配合远比单人做所有工作更有效。

![BERT Token嵌入流程](screenshots/04_bert_token_embedding.jpg)
![Dual Encoder架构](screenshots/04_dual_encoder_architecture.jpg)

---

## 第5课：Training a Dual Encoder（训练双编码器）

### 双编码器架构细节
- 两个独立的BERT编码器（Question Encoder + Answer Encoder）
- 使用每个编码器最后一层的**CLS嵌入向量**作为句子表示
- 点积相似度衡量问答匹配程度

### 对比损失（Contrastive Loss）
- **正样本对**：真实的问答配对（相似度应高）
- **负样本对**：不匹配的问答组合（相似度应低）
- 训练目标：拉近正样本，推开负样本

### Batch内负样本构造
- 一个batch中有N个问答对
- 对于问题Q_i，正样本是A_i
- 同batch中其他所有答案A_j（j!=i）作为负样本
- 形成N x N的相似度矩阵，对角线应为最大值

### 训练数据
- 使用问答对数据集
- 通过batch内交叉组合高效构造负样本
- 无需额外标注负样本

### 生动比喻
> 对比损失训练就像**相亲配对**——把正确的couple（正样本）拉近，把错误配对（负样本）推远。而Batch内负样本就像**集体相亲**——一桌10对人，每个人和正确对象配对是正样本，和其他9个人的"误配"是负样本，一次训练搞定10个正样本和90个负样本。

![对比损失](screenshots/05_contrastive_loss.jpg)
![编码器架构](screenshots/05_encoder_architecture.jpg)
![相似度矩阵](screenshots/05_similarity_matrix.jpg)

---

## 第6课：Using Embeddings in RAG（在RAG中使用嵌入）

### 生产环境中的双编码器使用

**索引阶段（Ingest）**
1. 将文档切分为文本块
2. 使用**Answer Encoder**编码每个文本块
3. 将向量存入向量数据库

**查询阶段（Query）**
1. 用户输入问题
2. 使用**Question Encoder**编码问题
3. 在向量数据库中进行相似度搜索
4. 返回TOP-K最匹配的文本块
5. 将文本块作为上下文发送给LLM生成回答

### 近似最近邻搜索（ANN）
- 暴力搜索（遍历所有向量）太慢
- 使用ANN算法加速：HNSW、IVF等
- 牺牲少量精度换取巨大速度提升

### 向量数据库
- 专门存储和检索高维向量的数据库
- 支持ANN索引和快速相似度搜索
- 常见：Pinecone, Weaviate, Chroma, FAISS等

### 生动比喻
> RAG中的Dual Encoder就像**图书馆管理系统**——入库时给每本书贴上"内容坐标"标签（Answer Encoder），读者找书时把需求也转换成坐标（Question Encoder），然后在坐标空间中找最近的那些书。ANN就像**图书馆的分区索引**——不用逐本翻找，先定位到相关区域再细找。

![RAG中Dual Encoder的使用](screenshots/06_rag_dual_encoder_usage.jpg)
![完整RAG Pipeline](screenshots/06_full_rag_pipeline.jpg)

---

## 第7课：Conclusion（结语）

### 课程总结
- 简单池化Token嵌入 **不足以** 产生好的句子表示
- 需要专门在**句子对/问答对**上训练的模型
- 训练完成后，句子嵌入模型是RAG和语义搜索的强力工具

### 两阶段检索（Retrieve and Rerank）
1. **第一阶段**：Sentence Embedding Model快速检索TOP-100
2. **第二阶段**：Cross Encoder精排，筛选出TOP-10
3. 兼得速度和精度——两全其美

### 互补检索技术
- **混合搜索（Hybrid Search）**：神经搜索 + 关键词搜索结合
- **元数据过滤**：按作者、日期等字段过滤
- **MMR（Max Marginal Relevance）**：平衡相关性和多样性

### 生动比喻
> 两阶段检索就像**选拔赛制**——初赛（Embedding快速筛选）淘汰大部分不相关文档，决赛（Cross Encoder精排）从候选者中选出最佳答案。这比让每个文档都参加"决赛"快得多。

![两阶段检索](screenshots/07_two_stage_retrieval.jpg)
![混合搜索](screenshots/07_hybrid_search.jpg)

---

## 总结表格

| 课时 | 主题 | 核心技能 |
|------|------|----------|
| 01 | Introduction | 嵌入模型发展史概览 |
| 02 | Intro to Embeddings | 词/句嵌入概念、RAG中的检索 |
| 03 | Contextualized Embeddings | BERT上下文化嵌入、Cross Encoder |
| 04 | Token vs Sentence Embedding | Mean Pooling失败、Dual Encoder引入 |
| 05 | Training Dual Encoder | 对比损失、Batch负样本构造 |
| 06 | Embeddings in RAG | 生产部署、ANN搜索、向量数据库 |
| 07 | Conclusion | 两阶段检索、混合搜索 |

---

## 知识树 (ASCII)

```
Embedding Models: from Architecture to Implementation
├── 词嵌入基础
│   ├── Word2Vec（预测周围词 → 学习词向量）
│   ├── GloVe（简化数学）
│   └── 特性：向量代数运算（king - man + woman = queen）
├── 上下文化嵌入
│   ├── 问题：静态词向量无法处理一词多义
│   ├── Transformer架构
│   │   ├── Encoder（双向注意力）→ BERT
│   │   └── Decoder（单向注意力）→ GPT
│   └── BERT
│       ├── 预训练任务：MLM（完形填空）+ NSP（下句预测）
│       ├── 输出：上下文化的Token嵌入
│       └── 应用：Cross Encoder（拼接Q+A做分类）
├── 句子嵌入
│   ├── 朴素方法（Mean Pooling / CLS Token）→ 失败
│   ├── 原因：需要专门的句子级训练
│   ├── 纯相似度 vs 问答检索（不同目标！）
│   └── Dual Encoder架构
│       ├── Question Encoder（编码问题）
│       ├── Answer Encoder（编码答案）
│       └── 对比损失训练
│           ├── 正样本：真实问答对
│           ├── 负样本：Batch内交叉组合
│           └── 目标：拉近正样本，推开负样本
├── 生产部署
│   ├── 索引：Answer Encoder编码文本块 → 向量数据库
│   ├── 查询：Question Encoder编码问题 → ANN搜索
│   └── ANN算法：HNSW、IVF等
└── 高级检索策略
    ├── 两阶段检索（Retrieve and Rerank）
    │   ├── 第一阶段：Embedding快速召回TOP-100
    │   └── 第二阶段：Cross Encoder精排TOP-10
    ├── 混合搜索（Neural + Keyword）
    ├── 元数据过滤
    └── MMR（最大边际相关性）
```
