# KV Cache（键值缓存）

## Summary
KV Cache 是 Transformer 推理时的关键加速机制：生成第 N 个 token 时，前面已经计算过的 Key/Value 矩阵被缓存复用，避免重复计算。是当前主流 LLM 推理框架（vLLM、TGI、SGLang）显存优化的核心对象。

## Key Points
- **何时启用**：仅在自回归 generation 阶段，prefill 后逐 token 解码时
- **缓存什么**：每层 [[attention-mechanism]] 的 K 和 V 矩阵（不缓存 Q，因为 Q 每步重算）
- **显存占比**：推理时 KV Cache 常常**比模型权重还大**（长上下文场景），是 OOM 的主要源头
- **配套优化**：
  - PagedAttention（vLLM）：把 KV Cache 切成固定大小 page，按需分配
  - GQA / MQA：减少 KV head 数 → 直接缩 KV Cache 体积
  - Sliding Window / Sparse KV：丢弃远距离 KV 节省显存

## Why It Matters
没有 KV Cache，生成第 N 个 token 是 O(N²)；有 KV Cache 是 O(N)。这是为什么"长输出比长输入更慢"在很多框架里被反过来——长输出意味着 KV Cache 持续增长，每步显存压力上升。

## Sources
- [[2026-01-01_transformer-llm-how-they-work]]

## Related
- [[attention-mechanism]]
- [[transformer-architecture]]
