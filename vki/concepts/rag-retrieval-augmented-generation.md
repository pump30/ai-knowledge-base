# RAG (Retrieval-Augmented Generation)

## Summary
RAG 是将外部知识检索与 LLM 生成结合的架构模式。核心流程：用户提问 → 检索相关文档片段 → 将检索结果作为上下文传给 LLM → 生成回答。解决了 LLM 知识过时和幻觉问题。

## Key Points
- 三阶段 Pipeline：Ingestion（文档处理）→ Retrieval（检索）→ Synthesis（生成）
- 评估三角：Answer Relevance、Context Relevance、Groundedness
- 检索方法演进：Naive RAG → Sentence Window → Auto-merging → Hybrid

## Advanced RAG 技术

### Sentence Window Retrieval
- 检索粒度：单句
- 返回上下文：句子 ± window_size 的窗口
- 最优 window_size = 3

### Auto-merging Retrieval
- 层级节点结构（128 → 512 → 2048 字符）
- 当超过阈值的子节点被检索时，自动合并为父节点
- 三层结构比两层 Context Relevance 高 20%，成本降一半

### Two-Stage Retrieval
- 第一阶段：Embedding 快速召回（Bi-Encoder）
- 第二阶段：Cross Encoder 精排
- 结合速度和精度

## 检索优化
- 向量量化：PQ (64x压缩)、SQ (4x)、BQ (32x, 40x加速)
- HNSW 参数调优：M (边数)、ef (候选数)
- Hybrid Search：向量搜索 + 关键词搜索 + Metadata 过滤
- MMR (Maximal Marginal Relevance)：平衡相关性和多样性

## Sources
- [[2026-01-08_building-evaluating-advanced-rag]]
- [[2026-01-09_retrieval-optimization-tokenization-vector-quantization]]
- [[2026-01-07_embedding-models-architecture-implementation]]
- [[2026-01-05_langchain-llm-app-development]]

## Related
- [[embedding-models]]
- [[vector-quantization]]
- [[evaluation-methods]]
