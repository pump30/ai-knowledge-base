# Transformer 架构

## Summary
Transformer 是现代 LLM 的核心架构，由 Google Brain 在 2017 年论文 "Attention is All You Need" 中提出。核心创新是 Self-Attention 机制，完全取代了 RNN/LSTM 的序列处理方式，支持并行计算。

## Key Points
- 完整流程：Tokenizer → Embeddings → Transformer Blocks → LM Head
- **每个 Block = Attention + FFN**（两大子模块，分工明确）
- 核心公式：softmax(QK^T / √d_k) × V
- 三种 Attention：Self-Attention、Masked Self-Attention、Cross-Attention
- **参数分布：Attention ~33%, FFN ~67%**（FFN 是工作主力，存储事实知识）
- 现代改进：RoPE 位置编码、Pre-Norm、Grouped Query Attention、Sparse Attention、SwiGLU、MoE
- 架构变体：Encoder-only (BERT)、Decoder-only (GPT/Claude/Llama)、Encoder-Decoder (T5)

## Evolution
1. Bag-of-Words → Word2Vec → RNN+Attention → Transformer (2017)
2. Original Transformer → Modern Transformer (RoPE, Pre-Norm, MoE)

## Sources
- [[2026-01-01_transformer-llm-how-they-work]]
- [[2026-01-01_understanding-language-models-transformers]]
- [[2026-01-02_attention-in-transformers-pytorch]]

## Related
- [[attention-mechanism]]
- [[feed-forward-network]]
- [[embedding-models]]
- [[tokenization]]
- [[mixture-of-experts]]
