# AI 评估方法

## Summary
评估是 AI 系统开发的核心环节。从 Prompt 开发到 Agent 系统，都需要系统化的评估方法。核心理念是 Evaluation-Driven Development：先定义评估标准，再开发系统。

## 评估层次

### 1. Prompt/Model 层
- 人工评估 → 规则评估 → LLM-as-Judge → 大规模测试
- Temperature 0 用于评估以保证可重复性

### 2. RAG 系统评估（RAG Triad）
- **Answer Relevance**：回答是否切题
- **Context Relevance**：检索的上下文是否相关（最关键，低 CR 导致低 Groundedness）
- **Groundedness**：回答是否有上下文支撑（非模型自行编造）
- TruLens Feedback Functions 自动化评估

### 3. Agent 评估
- **Router 评估**：工具选择正确性 + 参数提取准确性
- **Skill 评估**：单个能力的输出质量
- **Trajectory 评估**：执行路径效率
- **Convergence Score**：多次运行结果一致性
- 分解评估：将 Agent 拆分为 Router + Skills + Memory 分别评估

## 三种评估技术
| 方法 | 准确性 | 可扩展性 | 适用场景 |
|------|---------|----------|----------|
| Code Evaluation | 100% | 高 | 有标准答案时 |
| LLM-as-Judge | <100% | 高 | 开放式回答 |
| Human Annotation | 高 | 低 | 金标准、验证 LLM Judge |

## Best Practices
- LLM Judge 用离散标签（correct/incorrect）远优于连续分数（1-100）
- 先小规模人工验证 LLM Judge 的准确性
- 生产环境：OpenTelemetry 追踪 + 持续监控
- 数据闭环：生产问题 → 评估数据集 → 改进 → 部署

## Sources
- [[evaluating-ai-agents]]
- [[building-evaluating-advanced-rag]]
- [[building-systems-chatgpt-api]]

## Related
- [[rag-retrieval-augmented-generation]]
- [[ai-agent-design-patterns]]
- [[llm-system-architecture]]
