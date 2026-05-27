# Claude Code

## Type
Tool / AI Assistant

## Summary
Claude Code 是 Anthropic 推出的高度自主编程助手（CLI + Desktop + Web）。核心架构：轻量 harness + 模型 + 工具 + 环境。最大特点是 Agentic Search（无需预索引，按需搜索）和 CLAUDE.md 记忆系统。

## Key Facts
- 架构：lightweight harness + Claude model + built-in tools + local environment
- 内置工具：Read, Edit, Write, Grep, Glob, Bash, SubAgent
- 记忆系统：CLAUDE.md（三个范围：global / project / directory）
- Agentic Search：不做代码索引，按需搜索（vs 传统语义嵌入索引）
- Plan Mode：先规划架构蓝图再执行
- Git Worktrees：并行开发，多实例独立工作
- MCP 集成：Playwright（浏览器测试）、Figma（设计转代码）
- SubAgent：隔离上下文的子任务执行

## 设计哲学
- 轻量 harness，重模型能力
- 本地优先，无需服务器
- 工具组合 > 单一能力
- 上下文管理是核心挑战

## Sources
- [[2026-01-14_claude-code-agentic-coding-assistant]]
- [[2026-01-15_agent-skills-anthropic]]
- [[2026-01-13_mcp-rich-context-ai-apps-anthropic]]

## Related
- [[mcp-model-context-protocol]]
- [[skills-system]]
- [[ai-agent-design-patterns]]
