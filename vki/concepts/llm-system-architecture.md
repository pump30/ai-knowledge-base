# LLM 系统架构

## Summary
构建基于 LLM 的生产系统需要多步骤架构：输入评估 → 处理 → 输出检查。核心思想是将 LLM 作为系统中的组件，而非单次调用，通过 pipeline 编排多个 LLM 调用完成复杂任务。

## Key Points
- 输入层：分类（路由）、Moderation API（安全）、Prompt Injection 防御
- 处理层：Chain of Thought、Chaining Prompts（多步 pipeline）
- 输出层：内容审核 + LLM 自验证
- Inner Monologue：隐藏推理过程，只向用户展示最终结果

## 设计模式

### Chain of Thought vs Chaining Prompts
- CoT：单个 prompt 内多步推理
- Chaining：多个 prompt 组成 pipeline，每步专注一件事
- Chaining 优势：可调试、可独立优化、可并行

### LangChain Chain Types
- LLMChain：单步
- SequentialChain：顺序 pipeline
- RouterChain：动态路由到不同子链

### LCEL (LangChain Expression Language)
- Runnable 协议：invoke / stream / batch
- Pipe 操作符：`chain = prompt | model | parser`
- 内置优势：async/batch/streaming/fallback/parallelization

## Sources
- [[2026-01-04_building-systems-chatgpt-api]]
- [[2026-01-05_langchain-llm-app-development]]
- [[2026-01-06_functions-tools-agents-langchain]]

## Related
- [[prompt-engineering]]
- [[ai-agent-design-patterns]]
- [[evaluation-methods]]
