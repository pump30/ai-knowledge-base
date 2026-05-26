# Mixture of Experts (MoE)

## Summary
MoE 是现代大模型的关键架构创新：把 Transformer Block 中**单个大 FFN 拆成多个小 FFN（专家）**，每次推理只激活其中少数几个。让模型总参数量大但推理算力小，是 GPT-4 / Mixtral / DeepSeek V3 等顶级模型的核心架构。

## Key Points
- 替换的是 **FFN 层**（不是 Attention）
- 多个并行的"专家"FFN + 一个 Router 路由器
- 推理时只激活 top-k 个专家（通常 k=2）
- **总参数大，活跃参数小** → 推理高效
- 每一层独立路由，不同层可选不同专家

## 结构对比
```
传统 Transformer Block:           MoE Transformer Block:
   Attention                          Attention
       ↓                                  ↓
    1 个 FFN              [Expert 1] [Expert 2] ... [Expert N]
                                ↑ Router 选 top-k 激活 ↑
```

## 工作流程
1. 每个 token 经过 Attention 层（与传统一样）
2. 进入 MoE 层：
   - **Router**（小分类器）根据 token 的特征决定派给哪些专家
   - 选出 top-k 专家（如 top-2）
   - 加权合并这几个专家的输出
3. 进入下一层

## 参数 vs 活跃参数
| 模型 | 总参数 | 推理活跃参数 | 专家数 |
|------|--------|-------------|--------|
| Mixtral 8x7B | 47B | ~13B | 8（激活 2） |
| DeepSeek V3 | 671B | 37B | 256（激活 8） |
| GPT-4（推测） | ~1.8T | ~280B | 16（激活 2） |

## 关键洞察：Expert ≠ 领域专家
**容易误解：** 听到"专家"会以为 Expert 1 是"心理学专家"、Expert 2 是"生物学专家"——**不是这样**。

实际上：
- Expert 更像擅长处理不同**类型 token**的专家（如标点 / 动词 / 罕见词 / 代码符号）
- 路由是 **token 级**的，不是 query 级
- 一个查询里不同 token 可能被分到不同专家
- 每一层独立路由，模式更复杂

## 比喻
| 模型类型 | 比喻 |
|---------|------|
| Dense（传统） | 全科医生：每个问题都自己看 |
| **MoE** | **医院分诊台：Router 看症状 → 分到对应专科医生（Expert）** |

## 优缺点

### 优点
- **算力效率高**：1.8T 参数模型推理只用 280B
- **可扩展性强**：加 Expert 比加深层数便宜
- **不同模式分工**：减少参数干扰

### 缺点
- **训练不稳定**：Router 可能"偏心"某些 Expert
- **显存需求高**：所有 Expert 都要加载到内存
- **负载均衡难**：需要额外 loss 强制 Expert 使用均衡
- **可能过拟合**：某 Expert 见的数据太单一

## 训练技巧
- **Auxiliary Load Balancing Loss**：惩罚 Router 过度依赖少数 Expert
- **Expert Capacity**：限制每个 Expert 每批最多处理多少 token
- **Top-k 路由**：通常 k=2，平衡性能和效率

## 当前趋势（2024-2026）
- **MoE 成为大模型标配**：GPT-4, Mixtral, DeepSeek, Qwen MoE
- **Expert 数量持续增加**：从 8 → 64 → 256 → 更多
- **Fine-grained MoE**：DeepSeek V3 把 Expert 拆得更小、数量更多
- **共享 Expert + 路由 Expert**：DeepSeek 等模型保留少量"通用专家"

## Sources
- [[transformer-llm-how-they-work]]
- [[2026-05-26_transformer-training-conversation]]

## Related
- [[transformer-architecture]]
- [[feed-forward-network]]
- [[attention-mechanism]]
