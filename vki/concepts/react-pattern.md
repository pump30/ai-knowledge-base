# ReAct Pattern

## Summary
ReAct (Reasoning + Acting) 是 AI Agent 的核心执行模式。Agent 交替进行推理（Thought）和行动（Action），通过观察（Observation）获得反馈后决定下一步。是大多数 Agent 框架的底层循环。

## 执行循环
```
Thought → Action → Pause → Observation → Thought → ... → Final Answer
```

## 实现要点
- LLM 负责：推理、决定调用哪个工具、解读结果
- Runtime 负责：执行工具调用、管理消息历史、控制循环终止
- 终止条件：AgentFinish（直接回答）vs AgentAction（需要工具）

## AgentExecutor vs 手动循环
| 特性 | 手动循环 | AgentExecutor |
|------|----------|---------------|
| 错误处理 | 自己实现 | 内置 |
| 超时控制 | 自己实现 | max_iterations |
| 重试 | 自己实现 | 内置 |
| 日志/追踪 | 自己实现 | 集成 LangSmith |

## Sources
- [[ai-agents-langgraph]]
- [[functions-tools-agents-langchain]]
- [[langchain-llm-app-development]]

## Related
- [[ai-agent-design-patterns]]
- [[langgraph]]
- [[chain-of-thought]]
