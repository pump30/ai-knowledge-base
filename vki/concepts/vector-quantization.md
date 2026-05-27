# Vector Quantization（向量量化）

## Summary
向量量化把高维 float 向量压缩成更小的表示（int8 / 1-bit / cluster ID），用**少量精度损失换显著的内存和速度收益**。是向量数据库（Qdrant、Milvus、pgvector）应对亿级向量的核心手段，常配合 HNSW 等 ANN 算法使用（HNSW 解决"找哪些"，量化解决"存得下、算得快"）。

## 三种量化技术对比

| 方法 | 压缩率 | 原理 | 精度影响 | 速度提升 |
|------|--------|------|----------|----------|
| **标量量化 (SQ)** | 4× | float32 → int8（每维独立线性映射） | 小 | 明显 |
| **乘积量化 (PQ)** | 可达 64× | 切子向量 → 每段做 K-means → 用簇 ID 替代 | 较大 | 中等 |
| **二值量化 (BQ)** | 32× | 正数 → 1，负数/零 → 0 | 最大 | 最快（~40×） |

## 各方法细节

### 标量量化 SQ
- 把 float32 映射到 [0, 255]
- 4 字节 → 1 字节，固定 4× 压缩
- **通常是首选**：精度损失小、实现简单、速度提升明显

### 乘积量化 PQ
1. 把 d 维向量切成 m 段子向量
2. 每段独立做 K-means（典型 K=256，刚好 1 字节存簇 ID）
3. 用簇 ID 序列替代原始子向量
4. 名字由来：用各子空间的**笛卡尔积**逼近原始向量空间

### 二值量化 BQ
- 每维 32 bit → 1 bit
- 必须配 **oversampling + rescoring**：先取多于 K 个候选，再用原始 float 向量重算距离精排
- 适合超大规模、对精度要求不极端的场景

## When to Use Which
- 默认 → **SQ**（性价比最高）
- 内存极度紧张 → **PQ**（接受较大精度损失）
- 极致速度 + 大量候选可承受 rescoring → **BQ**

## Sources
- [[2026-01-07_embedding-models-architecture-implementation]]
- [[2026-01-09_retrieval-optimization-tokenization-vector-quantization]]

## Related
- [[embedding-models]]
- [[rag-retrieval-augmented-generation]]
