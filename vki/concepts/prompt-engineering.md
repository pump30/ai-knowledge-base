# Prompt Engineering

## Summary
Prompt Engineering 是设计和优化 LLM 输入指令的技术，通过结构化的提示让模型产生更准确、更有用的输出。核心是两个原则：清晰具体的指令 + 给模型思考时间。

## Key Points
- 两大原则：Clear & Specific Instructions + Give Model Thinking Time
- 六大策略：分隔符、结构化输出、条件检查、Few-shot、步骤拆分、让模型自我推理
- 迭代开发：Prompt 是实验性的，需要反复测试调整
- 应用方向：Summarizing、Inferring、Transforming、Expanding
- Chat 格式：System（设定角色）/ User（提问）/ Assistant（回答）
- Temperature：0=确定性 vs >0=创造性

## 与传统 ML 的对比
- 传统 ML 开发周期：数月
- Prompt Engineering：分钟到小时
- 但需要注意：幻觉问题、Prompt Injection 攻击

## Sources
- [[chatgpt-prompt-engineering]]
- [[building-systems-chatgpt-api]]

## Related
- [[chain-of-thought]]
- [[llm-system-architecture]]
- [[prompt-injection-defense]]
