# Next-Token Prediction (自回归训练)

## Summary
Decoder-only 模型（GPT/Claude/Llama）的核心训练方式：给定前 N 个 token，预测第 N+1 个 token。通过 Masked Self-Attention 的 causal mask 确保模型只能看到过去的 token。这是当前主流 LLM 的训练范式。

## Key Points
- 训练目标：每个位置都预测"下一个词"
- 一个长度 N 的序列产生 **N-1 个训练信号**，效率极高
- 必须配合 Masked Self-Attention（causal mask），否则模型能"作弊"看答案
- Loss：所有位置的 Cross-Entropy 加和（或平均）
- 推理阶段：自回归生成，预测出的词作为下一步输入

## 训练示例
```
输入序列: 今 天  天  气  真
预测目标:    天  天  气  真  好
            ↑   ↑   ↑   ↑   ↑
位置 1 看到「今」预测「天」
位置 2 看到「今天」预测「天」
位置 3 看到「今天天」预测「气」
...
```

## 与 MLM 对比
| | Next-Token Prediction | [[masked-language-modeling]] |
|---|---|---|
| 每序列预测数 | ~N-1 个 | ~15% × N 个 |
| 视野 | 单向（只看过去） | 双向 |
| 训练效率 | **高 6-7 倍** | 低 |
| 适合任务 | 生成 | 理解 |

## 为什么 Decoder-only 赢了
- 训练效率高 → 同样算力学得更多
- 天然适合生成 → 推理时直接续写
- 规模化更好 → Scaling Law 在 next-token prediction 上验证最充分
- 通用性强 → 生成任务可以"包装"理解任务（QA 形式）

## "Mask" 命名陷阱
**注意：** Next-token prediction 用的 "Masked Self-Attention" 里的 mask，是计算层面的 causal mask（注意力矩阵右上三角设为 -∞）；和 MLM 里的 "Masked"（数据层面把词挖空）**完全是两回事**。

## Sources
- [[2026-05-26_transformer-training-conversation]]
- [[2026-01-01_transformer-llm-how-they-work]]
- [[2026-01-01_understanding-language-models-transformers]]
- [[2026-01-02_attention-in-transformers-pytorch]]

## Related
- [[masked-language-modeling]]
- [[training-loop]]
- [[attention-mechanism]]
- [[transformer-architecture]]
- [[encoder-only-vs-decoder-only]]
