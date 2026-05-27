# Chain of Thought (CoT)

## Summary
Chain of Thought 是让 LLM 逐步推理的技术，通过在 prompt 中要求模型"一步一步思考"来显著提升复杂推理任务的准确率。与 Chaining Prompts（多步 pipeline）是互补而非替代关系。

## Key Points
- 核心思想：让模型展示中间推理步骤，而非直接跳到答案
- 触发方式："Let's think step by step" 或 few-shot 示范推理过程
- Inner Monologue：隐藏推理过程，只展示最终答案给用户
- 适用场景：数学推理、逻辑判断、多步决策

## CoT vs Chaining Prompts
| 维度 | Chain of Thought | Chaining Prompts |
|------|-----------------|------------------|
| 范围 | 单个 prompt 内 | 多个 prompt 串联 |
| 可调试性 | 较低 | 高（每步独立） |
| Token 消耗 | 单次较多 | 分散到多步 |
| 适用场景 | 单任务推理 | 复杂系统流程 |

## Sources
- [[2026-01-04_building-systems-chatgpt-api]]
- [[2026-01-03_chatgpt-prompt-engineering]]

## Related
- [[prompt-engineering]]
- [[react-pattern]]
- [[llm-system-architecture]]
