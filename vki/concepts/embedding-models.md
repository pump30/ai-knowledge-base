# Embedding Models

## Summary
Embedding 将文本映射为稠密向量表示，是语义搜索和 RAG 的基础。从静态词嵌入演进到上下文感知的句子嵌入，核心挑战是如何让向量真正捕获语义相似性。

## Key Points
- 演进路线：Word2Vec → GloVe → BERT → Dual Encoder (Sentence Transformers)
- Token Embedding vs Sentence Embedding：不能简单平均 token embedding 得到好的句子表示
- 为什么 BERT CLS/Mean Pooling 效果差：未经过相似度任务训练
- 解决方案：Dual Encoder + Contrastive Loss + In-batch Negatives

## Dual Encoder 架构
- Question Encoder + Answer Encoder（可以相同或不同模型）
- 训练目标：正样本对相似度高，负样本对相似度低
- In-batch Negatives：同 batch 内其他样本作为负例，高效利用数据

## 关键模型
- Word2Vec / GloVe：静态，一词一向量
- BERT Base/Large：上下文感知，但非为相似度优化
- all-MiniLM-L6-v2：轻量高效的句子嵌入
- DPR (Dense Passage Retrieval)：Facebook 的 QA 专用模型
- ColBERT：token-level 交互 + late interaction
- E5：Microsoft 的通用嵌入模型

## Sources
- [[embedding-models-architecture-implementation]]
- [[retrieval-optimization-tokenization-vector-quantization]]

## Related
- [[rag-retrieval-augmented-generation]]
- [[transformer-architecture]]
- [[vector-quantization]]
