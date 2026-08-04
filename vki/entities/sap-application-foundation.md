# SAP Application Foundation

## Type
Platform / Framework

## Summary
SAP Application Foundation 是 SAP BTP 上专为构建 **code-based AI agents** 设计的综合平台。它面向需要完全控制 agent 行为的 Python 开发者，提供从开发工具（SDK + 模板 + AI 编码助手 Skills）到部署基础设施（Kyma Runtime）和 AI 服务（SAP AI Core）的完整栈。

## Key Facts
- **定位**：Code-based（非 Low-code/No-code），开发者用 Python 定义 agent 逻辑
- **运行时**：Kyma Runtime（Kubernetes-based），agent 打包为 Docker 容器，自动扩缩容
- **AI 引擎**：通过 SAP AI Core 访问 LLM，OAuth 2.0 认证 + 速率限制
- **SDK**：SAP Cloud SDK for Python（开源），模块包括 Audit Log、Destination Service、Object Store、Telemetry
- **模板**：A2A Agent Template，预配置项目结构 + CI/CD pipelines
- **Skills**：开发时辅助工具（SAP Agent Bootstrap / Instrumentation / Run Local），不是部署到 agent 中的能力
- **多区域**：支持 EU / US / Canada 等多 region 部署，满足数据驻留要求
- **合规**：审计追踪、数据隔离、多监管框架支持
- **IDE**：推荐 VS Code + Cline（AI 编码助手）

## Architecture
```
Developer (Python + VS Code + Cline)
       ↓ (Skills: Bootstrap, Instrumentation, Run Local)
   A2A Agent Template
       ↓ (Docker container)
   Kyma Runtime (BTP, multi-region)
       ↓ (OAuth 2.0)
   SAP AI Core (LLM access)
```

## Sources
- [[2026-08-04_sap-application-foundation-overview]]

## Related
- [[ai-agent-design-patterns]] — Application Foundation 是 Agent 模式的具体实现平台
- [[skills-system]] — 平台使用 Skills 机制扩展 Cline 的 agent 开发能力
- [[mcp-model-context-protocol]] — A2A Protocol 与 MCP 同属 agent 通信标准
