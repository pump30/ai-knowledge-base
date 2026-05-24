# crewAI

## Type
Framework / Tool

## Summary
crewAI 是专注于角色扮演多智能体协作的框架，创始人 Joao Moura。核心哲学是"Manager Mindset"：定义目标 → 定义流程 → 招聘人才 → 设定角色。比 LangGraph 更高层抽象，聚焦于基于角色的多 Agent 协作。

## Key Facts
- 三大核心组件：Agent, Task, Crew
- 六大设计要素：Role Playing, Focus, Tools, Cooperation, Guardrails, Memory
- 协作模式：Sequential（接力赛）、Hierarchical（交响乐指挥）、Parallel
- Memory：Short-term, Long-term, Entity
- 工具品质要求：Versatile, Fault-tolerant, Cached
- 不同 Agent 可用不同模型（如 GPT-4 做 Manager，Llama-3 做 Worker）
- 支持 Human Input 和 Delegation

## vs LangGraph
| 维度 | crewAI | LangGraph |
|------|--------|-----------|
| 抽象层级 | 高（角色/任务） | 低（图/节点） |
| 适用场景 | 多人协作模拟 | 任意 Agent 工作流 |
| 灵活性 | 中 | 高 |
| 上手难度 | 低 | 中 |

## Sources
- [[multi-agent-systems-crewai]]

## Related
- [[langgraph]]
- [[ai-agent-design-patterns]]
