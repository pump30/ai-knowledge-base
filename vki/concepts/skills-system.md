# Skills 系统

## Summary
Skills 是指令文件夹形式的 Agent 能力扩展机制（Anthropic 提出），遵循 Progressive Disclosure 原则：name/description 始终加载，body 按需触发，references 按需读取。定位介于 Tools（底层能力）和完整 Agent 之间。

## Key Points
- 结构：SKILL.md = YAML frontmatter (name + description) + Markdown body
- Progressive Disclosure：三级加载，节省上下文
- 跨平台兼容：Claude Code、Claude AI、Codex、Gemini CLI
- Skill Creator：元技能，用于创建新 Skills

## Skills 定位对比
| 维度 | Skills | Tools | MCP | Subagents |
|------|--------|-------|-----|-----------|
| 本质 | 知识/指令 | 底层能力 | 外部数据访问 | 并行隔离执行 |
| 持久性 | 跨会话 | 即时调用 | 实时连接 | 跨会话 |
| 控制 | 用户触发 | 模型决定 | 标准协议 | 父 Agent 派发 |

## 三大价值
1. **领域专业知识**：将专家知识编码为可复用指令
2. **可重复工作流**：标准化复杂流程（TDD、调试、代码审查）
3. **新能力**：扩展 Agent 能做的事

## 企业应用：SAP Application Foundation Skills
SAP Application Foundation 将 Skills 概念应用于企业 Agent 开发辅助：
- Skills 是**开发时**辅助 Cline（AI 编码助手）的工具，不是部署到 agent 中的运行时能力
- 预置 Skills：SAP Agent Bootstrap（脚手架）、SAP Agent Instrumentation（可观测性）、SAP Agent Run Local（本地测试）
- 让 Cline 更懂 Application Foundation 的项目结构和配置方式

## 注意事项
- Subagents 不继承父 Agent 的 Skills（需显式指定）
- Agent 架构演进：单一用途 → 简单脚手架 → Skills 增强脚手架（最优平衡）
- SAP Application Foundation 中 Skills ≠ Agent 的运行时能力（仅限开发时辅助）

## Sources
- [[2026-01-15_agent-skills-anthropic]]
- [[2026-01-14_claude-code-agentic-coding-assistant]]
- [[2026-08-04_sap-application-foundation-overview]]

## Related
- [[mcp-model-context-protocol]]
- [[claude-code]]
- [[ai-agent-design-patterns]]
- [[sap-application-foundation]]
