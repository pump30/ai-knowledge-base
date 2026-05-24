# Transformer 架构

## Summary
Transformer 是现代 LLM 的核心架构，由 Google Brain 在 2017 年论文 "Attention is All You Need" 中提出。核心创新是 Self-Attention 机制，完全取代了 RNN/LSTM 的序列处理方式，支持并行计算。

## Key Points
- 完整流程：Tokenizer → Embeddings → Transformer Blocks → LM Head
- 核心公式：softmax(QK^T / √d_k) × V
- 三种 Attention：Self-Attention、Masked Self-Attention、Cross-Attention
- 现代改进：RoPE 位置编码、Pre-Norm、Grouped Query Attention、Sparse Attention
- 架构变体：Encoder-only (BERT)、Decoder-only (GPT/Claude/Llama)、Encoder-Decoder (T5)

## Evolution
1. Bag-of-Words → Word2Vec → RNN+Attention → Transformer (2017)
2. Original Transformer → Modern Transformer (RoPE, Pre-Norm, MoE)

## Sources
- [[transformer-llm-how-they-work]]
- [[understanding-language-models-transformers]]
- [[attention-in-transformers-pytorch]]

## Related
- [[attention-mechanism]]
- [[embedding-models]]
- [[mixture-of-experts]]
