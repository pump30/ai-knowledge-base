# Tools vs Skills vs MCP vs Subagents

## Summary
四种 AI Agent 能力扩展机制，各有定位：Tools 是底层能力，Skills 是知识/指令，MCP 是标准化外部连接，Subagents 是隔离执行单元。

## Comparison Table

| Dimension | Tools | Skills | MCP | Subagents |
|-----------|-------|--------|-----|-----------|
| 本质 | 可调用的函数 | 知识/指令文档 | 标准化连接协议 | 独立 Agent 实例 |
| 谁触发 | 模型决定 | 用户/条件触发 | 通过 Tools/Resources | 父 Agent 派发 |
| 持久性 | 即时调用 | 跨会话持久 | 实时连接 | 任务期间存在 |
| 上下文影响 | 小（函数签名） | 中（按需加载 body） | 小（工具定义） | 无（隔离上下文） |
| 扩展难度 | 需要代码 | 只需写 Markdown | 需要实现 Server | 需要定义 prompt |

## When to Use Which
- **Tools**：具体操作（读文件、调 API、执行命令）
- **Skills**：领域知识、工作流规范、最佳实践指导
- **MCP**：连接外部系统（数据库、SaaS、浏览器）
- **Subagents**：并行独立任务、需要隔离上下文

## Sources
- [[2026-01-15_agent-skills-anthropic]]
- [[2026-01-13_mcp-rich-context-ai-apps-anthropic]]
- [[2026-01-14_claude-code-agentic-coding-assistant]]

## Related
- [[skills-system]]
- [[mcp-model-context-protocol]]
- [[claude-code]]
- [[ai-agent-design-patterns]]
