# AI Agent 设计模式

## Summary
AI Agent 是能够自主规划、使用工具、反思和协作的 LLM 系统。五大核心设计模式：Planning、Tool Use、Reflection、Multi-agent、Memory。核心循环是 ReAct：Thought → Action → Observation → 重复。

## 五大设计模式

### 1. Planning（规划）
- Plan-and-Execute：先制定计划再逐步执行，支持 re-planning
- LATS (Language Agent Tree Search)：树搜索 + 反思，多路径探索

### 2. Tool Use（工具使用）
- Function Calling：LLM 输出结构化的函数调用
- 工具选择：auto / none / force 模式
- 工具品质要求：Versatile、Fault-tolerant、Cached

### 3. Reflection（反思）
- 生成 → 评估 → 改进 的循环
- Self-Refine、Reflexion 等方法

### 4. Multi-agent（多智能体）
- Shared State：多个 Agent 共享状态
- Supervisor：中央协调者分配任务
- Hierarchical：层级式管理

### 5. Memory（记忆）
- Short-term：当前对话上下文
- Long-term：跨会话持久化（向量存储）
- Entity Memory：实体关系追踪

## Architecture Patterns
- Single Agent：简单任务，一个 Agent + 多工具
- Flow Engineering：主要是 pipeline + 关键节点有循环
- Multi-Agent：复杂任务分解给专业 Agent
- Supervisor：一个 Manager 协调多个 Worker

## Sources
- [[2026-01-10_ai-agents-langgraph]]
- [[2026-01-10_ai-agents-langgraph-extended]]
- [[2026-01-11_multi-agent-systems-crewai]]
- [[2026-01-06_functions-tools-agents-langchain]]

## Related
- [[react-pattern]]
- [[langgraph]]
- [[crewai]]
- [[evaluation-methods]]
