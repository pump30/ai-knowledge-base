# Attention 机制

## Summary
Attention 是 Transformer 的核心计算单元，让模型能够动态关注输入序列中不同位置的信息，而非固定窗口。Multi-Head Attention 允许并行捕获不同类型的关系（语法、语义、位置）。

## Key Points
- 数学本质：加权求和，权重由 Query 和 Key 的相似度决定
- 公式：Attention(Q,K,V) = softmax(QK^T / √d_k) × V
- Self-Attention：Q/K/V 来自同一序列（BERT，双向）
- Masked Self-Attention：只看过去 token（GPT，单向生成）
- Cross-Attention：Q 来自 Decoder，K/V 来自 Encoder（翻译任务）
- Multi-Head：多个 attention head 并行，原始论文用 8 头，Llama 3.2-405B 用 126 层

## 优化技术
- Grouped Query Attention (GQA)：多个 Q head 共享一组 K/V
- Sparse Attention：只计算部分位置的 attention
- KV Cache：缓存已计算的 K/V 避免重复计算

## Sources
- [[2026-01-02_attention-in-transformers-pytorch]]
- [[2026-01-01_transformer-llm-how-they-work]]

## Related
- [[transformer-architecture]]
- [[feed-forward-network]]
- [[kv-cache]]
