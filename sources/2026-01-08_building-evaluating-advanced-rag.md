# 构建与评估高级 RAG 系统 (Building and Evaluating Advanced RAG)

> **课程来源**: DeepLearning.AI  
> **讲师**: Jerry Liu (LlamaIndex 联合创始人兼 CEO) & Anupam Datta (TruEra 联合创始人兼首席科学家, CMU 教授)  
> **核心工具**: LlamaIndex + TruLens  

---

## 第一课: 课程介绍 (Introduction)

![课程概览](screenshots/01_course_overview.jpg)

### 核心要点

RAG (检索增强生成) 已成为让 LLM 基于用户自有数据回答问题的关键方法。但要构建**生产级别**的高质量 RAG 系统，需要:

1. **有效的检索技术** - 为 LLM 提供高度相关的上下文
2. **有效的评估框架** - 帮助你高效迭代和改进系统

### 课程涵盖的两种高级检索方法

| 方法 | 核心思想 |
|------|----------|
| **句子窗口检索 (Sentence Window Retrieval)** | 检索最相关的句子，同时提供该句子前后的上下文窗口 |
| **自动合并检索 (Auto-merging Retrieval)** | 将文档组织为树状结构，当多个子节点被检索时，自动合并为父节点 |

### RAG 三元组评估指标

![RAG三元组介绍](screenshots/01_rag_triad_intro.jpg)

- **上下文相关性 (Context Relevance)** - 检索到的文本块与用户问题的相关程度
- **基础性 (Groundedness)** - 回答是否有检索到的上下文支撑
- **答案相关性 (Answer Relevance)** - 最终回答与用户问题的相关程度

> **生动比喻**: 想象你是一个图书管理员（检索系统），读者（用户）问了一个问题。上下文相关性就像你是否找对了书，基础性就像你的回答是否真的来自那本书而不是你自己编的，答案相关性就像你的回答是否真正解答了读者的疑问。

---

## 第二课: 高级 RAG 流水线 (Advanced RAG Pipeline)

![基本RAG流水线](screenshots/02_rag_pipeline.jpg)

### 基本 RAG 流水线的三个阶段

```
文档 → [摄入 Ingestion] → [检索 Retrieval] → [合成 Synthesis] → 回答
```

#### 1. 摄入阶段 (Ingestion)
- 加载文档集合
- 使用文本分割器将文档切分为文本块 (chunks)
- 为每个文本块生成向量嵌入 (embedding)
- 将嵌入存储到索引/向量数据库中

#### 2. 检索阶段 (Retrieval)
- 接收用户查询
- 在索引中搜索与查询最相似的 Top-K 文本块

#### 3. 合成阶段 (Synthesis)
- 将检索到的文本块与用户查询组合
- 放入 LLM 的提示窗口中生成最终回答

### 评估基准设置

![RAG三元组指标](screenshots/02_rag_triad_metrics.jpg)

使用 TruLens 定义评估指标，建立基准线 (baseline)，然后对比高级技术的改进效果:

```python
# 评估三元组
answer_relevance    # 回答是否与问题相关
context_relevance   # 检索到的上下文是否与问题相关  
groundedness        # 回答是否基于检索到的上下文
```

### 句子窗口检索概述

![句子窗口检索概念](screenshots/02_sentence_window_concept.jpg)

**工作原理**: 嵌入和检索单个句子（更细粒度的块），但检索后用原始句子周围的更大窗口替换，为 LLM 提供更多上下文。

> **生动比喻**: 就像在一本书中用荧光笔标记了最关键的一句话，但读的时候会把这句话前后几段一起读，这样才能真正理解上下文。

### 自动合并检索概述

![自动合并检索概念](screenshots/02_auto_merging_concept.jpg)

**工作原理**: 构建父节点-子节点层级结构。如果一个父节点的大多数子节点都被检索到，就用父节点替换这些子节点。

> **生动比喻**: 如同拼图游戏 -- 如果你已经找到了一幅拼图中大部分碎片，不如直接把整幅画拿来。这保证了上下文的连贯性。

### 实验对比结果

对比基本 RAG 和句子窗口检索:
- **Groundedness**: 提升约 8 个百分点
- **Context Relevance**: 有所提升
- **总成本**: 更低（更高效）

---

## 第三课: RAG 三元组评估指标详解 (RAG Triad of Metrics)

### 反馈函数 (Feedback Functions) 概念

反馈函数是对 LLM 应用的**程序化评估**，在 0-1 分数范围内评分，并审查应用的输入、输出和中间结果。

### 1. 答案相关性 (Answer Relevance)

![答案相关性](screenshots/03_answer_relevance.jpg)

**定义**: 检查最终回答是否与用户提出的查询相关。

**结构**:
- **Provider**: 评估用的 LLM (如 OpenAI GPT-3.5)
- **输入**: 用户问题 + 最终回答
- **输出**: 0-1 评分 + 链式思维推理

```python
f_qa_relevance = Feedback(openai.relevance_with_cot_reasons, name="Answer Relevance")
    .on_input()
    .on_output()
```

### 2. 上下文相关性 (Context Relevance)

![上下文相关性](screenshots/03_context_relevance.jpg)

**定义**: 评估检索步骤的质量 -- 给定查询，每段检索到的上下文与问题的相关程度。

**特点**:
- 对每段检索到的上下文**分别评分**
- 最终取所有上下文评分的**平均值**
- 输入: 用户问题 + 中间结果（检索到的上下文）

```python
f_qs_relevance = Feedback(openai.qs_relevance_with_cot_reasons, name="Context Relevance")
    .on_input()
    .on(context_selection)
    .aggregate(np.mean)
```

### 3. 基础性 (Groundedness)

**定义**: 最终回答中的每个声明是否能在检索到的上下文中找到支持证据。

**特点**:
- 将最终回答**拆分为多个句子**
- 每个句子分别检查是否有证据支撑
- 聚合所有句子的评分得到最终分数

```python
f_groundedness = Feedback(openai.groundedness_measure_with_cot_reasons, name="Groundedness")
    .on(context_selection)
    .on_output()
```

### 评估与迭代工作流

![迭代工作流](screenshots/03_iteration_workflow.jpg)

```
基本 RAG → 评估(RAG三元组) → 发现失败模式 → 
    → 高级 RAG(句子窗口) → 重新评估 → 
    → 关注: Context Relevance 是否提升? 其他指标如何?
    → 实验不同窗口大小
```

**关键洞察**: 
- Context Relevance 低时，LLM 倾向于使用预训练知识填补空白，导致 Groundedness 也下降
- 窗口太小 → 上下文不够 → Context Relevance 和 Groundedness 低
- 窗口太大 → 不相关内容混入 → Groundedness 和 Answer Relevance 可能下降

### 评估方法的分类

| 方法 | 特点 | 局限性 |
|------|------|--------|
| Ground Truth 评估 | 专家标注，高质量 | 昂贵，难扩展 |
| 人类评估 | 较有意义 | 扩展性有限，一致性~80% |
| LLM 评估 | 可编程化，可扩展 | 与人类评估一致性约80-85% |
| 传统 NLP 指标 (ROUGE/BLEU) | 计算简单 | 纯语法层面，不理解语义 |

---

## 第四课: 句子窗口检索深入 (Sentence-window Retrieval)

![句子窗口检索原理图](screenshots/04_sentence_window_diagram.jpg)

### 核心问题与解决方案

**问题**: 标准 RAG 中，嵌入和合成使用相同的文本块。但嵌入检索在小块上效果好，LLM 合成需要大块提供足够上下文。

**解决方案**: 解耦检索和合成的粒度:
1. 嵌入**单个句子**（小粒度，精准匹配）
2. 存储每个句子前后的上下文窗口到元数据
3. 检索时找到相关句子
4. 合成前用更大的上下文窗口**替换**句子

### 实现组件

#### 1. 句子窗口节点解析器 (SentenceWindowNodeParser)

```python
node_parser = SentenceWindowNodeParser.from_defaults(
    window_size=3,          # 窗口大小: 前后各3个句子
    window_metadata_key="window",
    original_text_metadata_key="original_sentence"
)
```

将文档拆分为单个句子，每个句子节点的元数据中包含周围的上下文窗口。

#### 2. 元数据替换后处理器 (MetadataReplacementPostProcessor)

检索后、发送给 LLM 前，将节点文本替换为元数据中的完整窗口文本。

#### 3. 重排序器 (Sentence Transformer Re-rank)

![重排序器](screenshots/04_reranker.jpg)

```python
rerank = SentenceTransformerRerank(
    top_n=2,                # 最终保留前2个
    model="BAAI/bge-reranker-base"
)
```

**策略**: 初始 Top-K 设较大值(如6)，然后重排序器精选 Top-N(如2)，确保最相关的块被选中。

> **生动比喻**: 重排序器就像一个严格的面试官 -- 初选放宽标准让更多候选人进来，然后面试官仔细挑选最匹配的人。

### 窗口大小实验结果

![窗口大小权衡](screenshots/04_window_size_tradeoff.jpg)

| 窗口大小 | Context Relevance | Groundedness | Answer Relevance | 成本 |
|----------|-------------------|--------------|------------------|------|
| 1 | 低 (差) | 低 | 一般 | 低 |
| **3** | **高 (最佳)** | **高** | **高** | **中** |
| 5 | 持平 | 下降 | 持平 | 高 |

**结论**: 窗口大小 3 是最佳选择。

### 关键权衡关系

```
窗口太小 → Context Relevance 低 → LLM 使用预训练知识填补 → Groundedness 低
窗口太大 → Token 成本上升 → LLM 被过多信息淹没 → Groundedness 可能下降
```

---

## 第五课: 自动合并检索深入 (Auto-merging Retrieval)

![自动合并检索原理图](screenshots/05_auto_merging_diagram.jpg)

### 核心问题与解决方案

**问题**: 标准 RAG 检索到的多个文本块可能来自同一区域但顺序混乱、碎片化，这会阻碍 LLM 的合成能力。

**解决方案**: 
1. 定义层级结构 -- 小块链接到大的父块
2. 检索时，如果子节点被检索的比例超过阈值，自动合并为父节点

### 层级节点解析器 (HierarchicalNodeParser)

![层级节点结构](screenshots/05_hierarchy_nodes.jpg)

```python
node_parser = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128]  # 从大到小的层级
)
```

**结构**:
```
父节点 (2048 tokens)
├── 中间节点 (512 tokens)
│   ├── 叶子节点 (128 tokens)
│   ├── 叶子节点 (128 tokens)
│   ├── 叶子节点 (128 tokens)
│   └── 叶子节点 (128 tokens)
├── 中间节点 (512 tokens)
│   └── ...
└── ...
```

> **生动比喻**: 就像俄罗斯套娃 -- 最小的娃娃（叶子节点）用来精准匹配搜索，但如果你发现同一个中号娃娃里的多个小娃娃都被选中了，那就直接用中号娃娃（父节点），因为它包含了完整的连贯上下文。

### 索引构建策略

关键设计决策:
- **只对叶子节点做向量索引**（嵌入 + 检索）
- 所有中间节点和父节点存储在 **DocStore** 中
- 检索时动态合并

```python
# 只索引叶子节点
automerging_index = VectorStoreIndex(
    leaf_nodes,
    storage_context=storage_context,  # 包含所有节点的DocStore
    service_context=service_context
)
```

### 自动合并检索器 (AutoMergingRetriever)

```python
automerging_retriever = AutoMergingRetriever(
    base_retriever,
    storage_context,
    simple_ratio_thresh=0.5  # 如果>50%子节点被检索，合并为父节点
)
```

**流程**: Top-12 叶子节点 → 合并（可能得到 ~10 个节点）→ 重排序取 Top-6

### 两层 vs 三层结构对比

| 配置 | 层级 | 叶子大小 | Context Relevance | 成本 |
|------|------|----------|-------------------|------|
| App 0 (两层) | 512→2048 | 512 | 低 | 高 |
| **App 1 (三层)** | **128→512→2048** | **128** | **提升约20%** | **约一半** |

**三层结构优势**:
- 更小的叶子节点 → 更精准的初始检索
- 更多合并机会 → 更连贯的上下文
- Token 用量更少 → 成本更低

### 句子窗口 vs 自动合并的互补性

![互补技术](screenshots/05_complementary_techniques.jpg)

两种技术是**互补的**，而非替代关系:

- **句子窗口**: 适合扩展**连续**文本的上下文
- **自动合并**: 适合合并**非连续**但同属一个主题的文本块

例如: 如果子节点 1 和子节点 4 都与查询相关（中间隔了子节点 2 和 3），自动合并可以将它们合并为父节点，而句子窗口做不到这一点。

---

## 第六课: 总结与下一步 (Conclusion)

![总结](screenshots/06_conclusion.jpg)

### 课程总结

1. 学会了如何**构建、评估和迭代** RAG 应用以使其更接近生产级别
2. 减少 LLM 幻觉是每个开发者的**首要优先级**
3. 这两种技术只是冰山一角

### 推荐的下一步探索方向

- **数据流水线**: 理解你的数据预处理
- **检索策略**: 混合搜索 (Hybrid Search)
- **LLM 推理**: 链式思维 (Chain of Thought)
- **评估深入**: 模型置信度、校准、不确定性、可解释性、隐私、公平性、毒性

### TruLens 提供的更广泛评估

确保 LLM 应用:
- **诚实 (Honest)**: 不编造信息
- **无害 (Harmless)**: 不产生有害内容
- **有用 (Helpful)**: 切实解答用户问题

---

## 核心概念总结表

| 概念 | 定义 | 关键参数 | 实际效果 |
|------|------|----------|----------|
| RAG 流水线 | 摄入→检索→合成 | chunk_size, top_k | 基础问答能力 |
| 句子窗口检索 | 细粒度检索+扩展上下文 | window_size (最佳=3) | Context Relevance + Groundedness 提升 |
| 自动合并检索 | 层级结构+动态合并 | chunk_sizes, ratio_thresh | 上下文连贯性提升，成本降低 |
| 重排序器 | 二次精选检索结果 | top_n, model | 过滤不相关结果 |
| Context Relevance | 检索质量评估 | - | 诊断检索问题 |
| Groundedness | 回答有据评估 | - | 检测幻觉 |
| Answer Relevance | 回答相关性评估 | - | 确保回答切题 |
| Feedback Function | 程序化评估抽象 | provider, selector | 可扩展的评估体系 |

---

## 知识架构图

```
构建与评估高级 RAG
│
├── 基础架构
│   ├── 摄入 (Ingestion)
│   │   ├── 文档加载
│   │   ├── 文本分块 (Chunking)
│   │   ├── 向量嵌入 (Embedding) ── BGE-Small-EN
│   │   └── 索引存储 (VectorStoreIndex)
│   ├── 检索 (Retrieval)
│   │   └── Top-K 相似度搜索
│   └── 合成 (Synthesis)
│       └── LLM 生成回答 ── GPT-3.5-Turbo
│
├── 高级检索技术
│   ├── 句子窗口检索 (Sentence Window)
│   │   ├── SentenceWindowNodeParser
│   │   ├── MetadataReplacementPostProcessor
│   │   ├── SentenceTransformerRerank
│   │   └── 参数调优: window_size = 1/3/5
│   │
│   └── 自动合并检索 (Auto-Merging)
│       ├── HierarchicalNodeParser
│       ├── AutoMergingRetriever
│       ├── 层级结构: 128→512→2048
│       └── 合并阈值: ratio_thresh
│
├── 评估框架 (TruLens)
│   ├── RAG 三元组 (RAG Triad)
│   │   ├── Answer Relevance (问题↔回答)
│   │   ├── Context Relevance (问题↔上下文)
│   │   └── Groundedness (上下文↔回答)
│   ├── 反馈函数 (Feedback Functions)
│   │   ├── LLM 评估 (主流方式)
│   │   ├── 传统 NLP (ROUGE/BLEU)
│   │   └── 人类评估
│   └── 实验追踪 (Experiment Tracking)
│       └── StreamLit Dashboard
│
└── 生产化最佳实践
    ├── 建立基准线 → 评估 → 迭代
    ├── 分析失败模式
    ├── 权衡: 质量 vs 成本 vs 延迟
    └── 针对文档类型选择最佳超参数
```

---

## 实践要点速查

1. **起步**: 先用基础 RAG 建立 baseline，用 RAG 三元组评估
2. **优化检索**: 尝试句子窗口(window=3)或自动合并(三层128/512/2048)
3. **评估驱动**: 每次改动都用 TruLens 对比指标变化
4. **关注 Context Relevance**: 它是其他指标的基础，低则全低
5. **成本意识**: 窗口/层级越大，token 越多，成本越高
6. **文档适配**: 不同类型文档（合同vs发票）最佳参数不同
