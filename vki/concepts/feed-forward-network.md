# Feed-Forward Network (FFN)

## Summary
FFN 是 Transformer Block 的两大子模块之一（另一个是 Attention），位于每层 Self-Attention 之后。结构很简单——两层全连接 + 中间激活函数，但**承载了模型 ~67% 的参数和绝大多数事实知识**。Attention 负责跨 token 信息路由，FFN 负责单 token 内的深度处理和知识存储。

## Key Points
- 位置：每个 Transformer Block 内，紧跟 Attention 后面
- 结构：`Linear ↑ (扩 4 倍) → 激活 → Linear ↓ (压回)`
- Position-wise：每个 token 位置独立处理，不跨 token 交换信息
- 参数占比：~67%（远超 Attention 的 ~33%）
- 角色：知识存储 / 模式匹配（Attention 是路由）

## 标准实现
```python
def FFN(x):  # x: shape (d_model,)
    h = Linear_up(x)       # d_model → 4 × d_model
    h = activation(h)      # GeLU / SwiGLU / ReLU
    output = Linear_down(h)  # 4 × d_model → d_model
    return output
```

## 维度示例
| 模型 | d_model | FFN 中间层 | 倍数 |
|------|---------|-----------|------|
| GPT-3 175B | 12288 | 49152 | 4× |
| Llama 3 70B | 8192 | 28672 | 3.5× (SwiGLU) |
| BERT-base | 768 | 3072 | 4× |

## Attention vs FFN 分工
| | Attention | FFN |
|---|-----------|-----|
| 操作单位 | 跨 token | 单 token 内 |
| 角色 | 信息路由 / 关系建模 | 知识存储 / 模式匹配 |
| 比喻 | 图书馆员 | 百科全书 |
| 参数占比 | ~33% | ~67% |

## FFN = 键值存储器（Geva et al. 2021）
- **第一层 Linear ↑** = "查询键"：输入像不像某个模式？
- **激活函数** = 阈值过滤：保留高匹配度
- **第二层 Linear ↓** = "返回值"：输出对应知识

事实知识（"巴黎是法国首都"）基本都编码在 FFN 神经元权重里。

## 实验印证
- 删除特定 FFN 神经元 → 模型遗忘特定事实
- 删除特定 Attention head → 模型推理能力受损但事实记忆还在
- → 印证"知识存 FFN、推理靠 Attention"的分工

## 现代变体

### SwiGLU（Llama / Gemma / Claude）
带门控的 FFN，多一个分支控制信息流：
```python
h1 = Linear_up_a(x)
h2 = Linear_up_b(x)
h = SiLU(h1) * h2      # 门控
output = Linear_down(h)
```
效果更好，参数稍多。

### Mixture of Experts (MoE)
把单个大 FFN 拆成多个小 FFN（专家），每次只激活几个 → 见 [[mixture-of-experts]]

## 常见误解
> "Attention is All You Need" 让人以为 Attention 是 Transformer 的全部

不对：
- **Attention = Transformer 的创新点**（取代 RNN）
- **FFN = 工作主力**（占 2/3 参数和大部分计算）

## Sources
- [[transformer-llm-how-they-work]]
- [[understanding-language-models-transformers]]
- [[2026-05-26_transformer-training-conversation]]

## Related
- [[transformer-architecture]]
- [[attention-mechanism]]
- [[mixture-of-experts]]
