# Masked Language Modeling (MLM)

## Summary
Encoder-only 模型（BERT 系列）的核心训练方式：随机遮盖输入序列中 ~15% 的 token，让模型根据**双向**上下文预测被遮盖的词。与 Next-Token Prediction 对应，是另一大类自监督训练范式。

## Key Points
- 遮盖比例：15%（BERT 原论文）
- 双向上下文：模型可以同时看到被遮位置的左侧和右侧 token
- 用 `[MASK]` 特殊 token 替代被遮的词
- 每个序列只产生少数预测目标（被遮位置）
- Loss 只在被遮位置上计算

## 训练示例
```
原始: 今天 天气 真 好 ， 我们 去 公园 玩
遮盖: 今天 [MASK] 真 好 ， 我们 去 [MASK] 玩
任务: 预测 [MASK] 位置应该是「天气」「公园」
```

## BERT 的 80/10/10 策略
为防止训练-推理 gap（推理时输入没有 [MASK]），BERT 实际遮盖时：
- 80% 替换为 `[MASK]`
- 10% 替换为随机词
- 10% 保留原词

让模型不能假设"看到 [MASK] 才需要预测"。

## 与 Next-Token Prediction 对比
| | Masked Language Modeling | [[next-token-prediction]] |
|---|---|---|
| 视野 | 双向 | 单向（只看过去） |
| 每序列预测数 | ~15% × N | ~N-1 |
| 训练效率 | 低 | **高 6-7 倍** |
| 适合任务 | 理解、分类、Embedding | 生成、对话 |
| 代表模型 | BERT, RoBERTa | GPT, Claude, Llama |

## 为什么需要双向
理解任务（情感分类、NER、句子相似度）需要看到完整句子才能做出判断。例如判断 "苹果" 是水果还是公司，需要看后文 "公司发布了 iPhone"。单向模型无法在 "苹果" 位置利用后文信息。

## 当前定位
- **主要用途**：训练 Embedding 模型（[[embedding-models]]）
- **不再是主流**：通用 LLM 全部转向 decoder-only
- **仍然重要**：搜索、RAG、语义匹配场景的核心技术

## Sources
- [[2026-05-26_transformer-training-conversation]]
- [[2026-01-01_transformer-llm-how-they-work]]
- [[2026-01-01_understanding-language-models-transformers]]
- [[2026-01-02_attention-in-transformers-pytorch]]

## Related
- [[next-token-prediction]]
- [[training-loop]]
- [[attention-mechanism]]
- [[encoder-only-vs-decoder-only]]
- [[embedding-models]]
