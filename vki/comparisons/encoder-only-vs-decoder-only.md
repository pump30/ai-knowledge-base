# Encoder-Only vs Decoder-Only 模型

## Summary
Encoder-only（如 BERT）擅长理解/分类任务，使用双向 Self-Attention 看全部上下文；Decoder-only（如 GPT/Claude/Llama）擅长生成任务，使用 Masked Self-Attention 只看过去 token。当前主流 LLM 全部采用 Decoder-only。

## Comparison Table

| Dimension | Encoder-Only (BERT) | Decoder-Only (GPT/Claude/Llama) |
|-----------|--------------------|---------------------------------|
| Attention | 双向 Self-Attention | Masked Self-Attention（单向） |
| 训练方式 | MLM（遮盖预测） | Next-token Prediction |
| 擅长任务 | 分类、NER、语义相似度 | 文本生成、对话、代码 |
| 上下文 | 看到全部 token | 只看过去 token |
| 代表模型 | BERT, RoBERTa | GPT-4, Claude, Llama |
| 当前趋势 | 主要用于 Embedding | 主流 LLM 架构 |

## When to Use Which
- **Encoder-Only**：文本分类、语义搜索的 embedding 编码器、NER
- **Decoder-Only**：通用 AI 助手、代码生成、对话、创作

## 历史演进
- 2017: Transformer (Encoder + Decoder)
- 2018: BERT (Encoder-only) / GPT-1 (Decoder-only)
- 2020+: Decoder-only 成为绝对主流

## Sources
- [[2026-01-01_transformer-llm-how-they-work]]
- [[2026-01-01_understanding-language-models-transformers]]
- [[2026-01-02_attention-in-transformers-pytorch]]

## Related
- [[transformer-architecture]]
- [[attention-mechanism]]
- [[embedding-models]]
