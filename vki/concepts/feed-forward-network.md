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

## How FFN is Trained

**FFN 没有独立训练过程**——和 Attention、Embedding 一起端到端训练，普通反向传播，无特殊算法。详见 [[training-loop]]。

### 训练中的角色分化（涌现）
随着训练进行，两个 Linear 层自发分化：
- **W_up 的每一行 → 模式探测器**："输入像不像 Paris？""是不是过去时？"
- **W_down 的每一列 → 知识响应**："如果是法国地名，输出 France/Europe 特征"

这就是 [[feed-forward-network]] 键值存储结构的来源——**不是设计的，是训练中涌现的**（Geva et al. 2021）。

### FFN 训练的几个特殊性
| 维度 | 表现 |
|------|------|
| 梯度行为 | 比 Attention 平滑稳定（无 softmax 饱和） |
| 学习速度 | 比 Attention 慢，事实知识需反复看几千次 |
| "遗忘"难度 | 最难被覆盖——这是微调改风格易、改事实难的原因 |
| 计算占比 | 反向传播 ~60-70% 计算量在 FFN |
| 显存占比 | 中间激活值占大头（gradient checkpointing 主要救这部分）|
| 正则化 | Dropout 主要加在 FFN 内部和后面 |

### 一个具体训练故事
输入 "Paris is the capital of"：
- 第 100 步：FFN 随机，预测 France 1%（瞎猜）
- 第 100K 步：W_up 开始探测 "Paris" pattern → 预测 France 30%
- 第 10M 步：探测器与响应矩阵紧密耦合 → 预测 France 95%

事实知识就这样**逐步编码进 FFN 权重**。

### MoE 训练的特殊挑战
[[mixture-of-experts]] 引入 Router 后多了几个问题：
- **Router 不可微**（argmax 无梯度）→ 用 top-k 加权 + Straight-Through Estimator
- **负载不均**：Router 偏心 → 加 auxiliary load balance loss
- **Expert Capacity**：限制每 Expert 处理 token 数，超出的丢弃

这些让 MoE 训练比 Dense 模型更脆弱。

## 常见误解
> "Attention is All You Need" 让人以为 Attention 是 Transformer 的全部

不对：
- **Attention = Transformer 的创新点**（取代 RNN）
- **FFN = 工作主力**（占 2/3 参数和大部分计算）

## Sources
- [[2026-01-01_transformer-llm-how-they-work]]
- [[2026-01-01_understanding-language-models-transformers]]
- [[2026-05-26_transformer-training-conversation]]

## Related
- [[transformer-architecture]]
- [[attention-mechanism]]
- [[mixture-of-experts]]
