# LangGraph

## Type
Framework / Tool

## Summary
LangGraph 是 LangChain 生态的 Agent 编排框架，基于图（Graph）结构定义 Agent 工作流。核心概念：Nodes（节点/步骤）、Edges（边/连接）、Conditional Edges（条件路由）、Agent State（状态管理）。

## Key Facts
- 核心 API：StateGraph, add_node, add_edge, add_conditional_edges
- 状态管理：TypedDict + operator.add（追加模式）
- 持久化：Checkpointer (SqliteSaver) → 支持 time travel
- Human-in-the-Loop：interrupt_before 让人类审批
- Streaming：实时流式输出
- 多 Agent：Supervisor 模式、共享状态模式

## 高级架构
- Flow Engineering：主要 pipeline + 关键节点循环（AlphaCodium 论文）
- Plan-and-Execute：显式规划 → 执行 → 重新规划
- LATS：树搜索 + 反思，多路径探索
- Essay Writer：Plan → Research → Generate → Reflect → 迭代

## vs 手动 Agent 循环
- 状态管理：内置
- 持久化/恢复：内置（Checkpointer）
- Human-in-the-Loop：内置（interrupt）
- Time Travel：内置
- 追踪/调试：集成 LangSmith

## Sources
- [[ai-agents-langgraph]]
- [[ai-agents-langgraph-extended]]

## Related
- [[langchain]]
- [[ai-agent-design-patterns]]
- [[react-pattern]]
- [[crewai]]
