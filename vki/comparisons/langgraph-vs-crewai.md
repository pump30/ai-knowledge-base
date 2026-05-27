# LangGraph vs crewAI

## Summary
LangGraph 是低层图编排框架，灵活性高；crewAI 是高层角色扮演多 Agent 框架，上手快。选择取决于任务复杂度和对控制粒度的需求。

## Comparison Table

| Dimension | LangGraph | crewAI |
|-----------|-----------|--------|
| 抽象层级 | 低（图/节点/边） | 高（角色/任务/团队） |
| 核心隐喻 | 有向图（DAG） | 公司团队协作 |
| 灵活性 | 极高，任意拓扑 | 中等，预定义模式 |
| 状态管理 | 显式 TypedDict | 自动管理 |
| 多 Agent 模式 | 自定义（Supervisor/Shared State） | 内置（Sequential/Hierarchical/Parallel） |
| 持久化 | Checkpointer (SQLite/Redis) | 内置 Memory |
| Human-in-the-Loop | interrupt_before | human_input=True |
| 学习曲线 | 中等 | 低 |
| 适用场景 | 复杂自定义工作流 | 标准化多人协作模拟 |
| 生态 | LangChain 全家桶 | 独立框架 |

## When to Use Which
- **LangGraph**：需要精细控制流程、自定义状态、复杂条件路由、time travel 调试
- **crewAI**：快速搭建多 Agent 系统、角色分工明确、标准协作模式足够

## Sources
- [[2026-01-10_ai-agents-langgraph]]
- [[2026-01-11_multi-agent-systems-crewai]]

## Related
- [[langgraph]]
- [[crewai]]
- [[ai-agent-design-patterns]]
