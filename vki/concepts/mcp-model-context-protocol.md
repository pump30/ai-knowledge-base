# MCP (Model Context Protocol)

## Summary
MCP 是 Anthropic 推出的开放标准协议，用于标准化 LLM 与外部系统的集成。类比 USB 接口——解决 M 个应用 × N 个数据源的集成问题，变为 M + N。灵感来自 LSP (Language Server Protocol) 的标准化成功。

## Key Points
- 架构：Host > Client > Server（1:1 连接）
- 三大原语：
  - **Tools**（Model-controlled）：LLM 决定何时调用，类似 POST 请求
  - **Resources**（App-controlled）：应用决定加载，类似 GET 请求
  - **Prompts**（User-controlled）：用户选择触发的模板
- 传输方式：StdIO（本地子进程）、HTTP+SSE（远程独立进程）、Streamable HTTP（未来）
- Sampling：反向 LLM 请求，Server 可以请求 Host 的 LLM 能力

## SDK & Tools
- FastMCP：Python SDK，极简实现
- MCP Inspector：调试工具
- 支持的 Client：Claude Desktop、Cursor、Windsurf、VS Code

## 未来方向
- Registry API：服务发现
- Dynamic Discovery：动态工具发现
- Multi-agent Architecture：Agent 间通过 MCP 通信
- OAuth 2.1：标准化认证
- 递归组合：Client 可以是 Server，Server 可以是 Client

## Sources
- [[mcp-rich-context-ai-apps-anthropic]]
- [[claude-code-agentic-coding-assistant]]
- [[agent-skills-anthropic]]

## Related
- [[ai-agent-design-patterns]]
- [[claude-code]]
- [[skills-system]]
