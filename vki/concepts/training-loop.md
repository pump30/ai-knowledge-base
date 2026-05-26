# 训练循环 (Training Loop)

## Summary
Transformer 模型训练的通用机制：每一步将一批数据前向传播得到预测，与目标对比计算 loss，反向传播算出每个参数的梯度，再用优化器按梯度方向微调参数。重复几百万到几亿次后，随机初始化的权重收敛为有意义的语言模型。

## Key Points
- 自监督学习：文本本身就是答案，无需人工标注
- 五步循环：Forward → Loss → Backward → Optimizer Step → Repeat
- Forward Pass：输入 token 序列，输出每个位置对下一个 token 的概率分布
- Loss 函数：Cross-Entropy，公式 `Loss = -log(P(正确答案))`
- Backward Pass：链式法则计算所有参数的梯度
- Optimizer：AdamW 最常用，按 `新参数 = 旧参数 - 学习率 × 梯度` 更新

## 关键技术细节
- **Batch**：批量训练，几千到几百万 token 一起算（GPU 并行）
- **Teacher Forcing**：训练时用真实答案作输入，加速收敛但有 exposure bias
- **学习率调度**：Warmup（开头升温）+ Decay（后期降温）
- **混合精度训练**：FP16/BF16 减半显存、加快速度

## 训练阶段
现代 LLM 训练分多阶段：
1. **Pretraining** 预训练 — 万亿 token，学习语言和世界知识，占算力 95%+
2. **SFT** 监督微调 — 几万条人工高质量问答，学会回答格式
3. **RLHF / DPO** — 偏好对比，学会符合人类偏好

## 为什么能学到东西
模型参数容量大但被强迫"压缩"几万亿 token 的文本到这些参数里。要在有限参数下尽可能准确预测下一个词，模型别无选择，只能学会语法、事实、推理模式——预测下一个词 = 被迫理解世界。

## 规模感
GPT-4 级别训练估算：
- ~25,000 张 A100 GPU
- 训练 90-100 天
- 成本 ~1 亿美元
- 处理 ~13T token

## Sources
- [[2026-05-26_transformer-training-conversation]]
- [[transformer-llm-how-they-work]]

## Related
- [[transformer-architecture]]
- [[next-token-prediction]]
- [[masked-language-modeling]]
- [[attention-mechanism]]
