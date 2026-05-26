# VKI Index - AI/LLM Engineering Knowledge Network

> AI 维护的知识网络索引。所有页面、来源、关系一览。

---

## Concepts (18)

| Page | Description | Sources | Updated |
|------|-------------|---------|---------|
| [[transformer-architecture]] | Transformer 核心架构：Self-Attention、Q/K/V、现代改进 | 3篇 | 2026-05-26 |
| [[attention-mechanism]] | Attention 机制：Self/Masked/Cross、Multi-Head、GQA | 2篇 | 2026-05-26 |
| [[feed-forward-network]] | FFN：Transformer 知识存储主力，占 67% 参数 | 3篇 | 2026-05-26 |
| [[mixture-of-experts]] | MoE：拆 FFN 为多专家，激活 top-k，GPT-4/Mixtral 核心 | 2篇 | 2026-05-26 |
| [[tokenization]] | 分词：BPE/WordPiece/SentencePiece，词表训练 vs 算法设计 | 2篇 | 2026-05-26 |
| [[training-loop]] | 训练循环：Forward/Loss/Backward/Optimizer + Pretrain/SFT/RLHF | 2篇 | 2026-05-26 |
| [[next-token-prediction]] | 自回归训练：Decoder-only 模型的核心训练范式 | 4篇 | 2026-05-26 |
| [[masked-language-modeling]] | MLM 训练方式：BERT 的双向遮盖预测 | 4篇 | 2026-05-26 |
| [[prompt-engineering]] | Prompt 设计两大原则、六大策略、迭代开发 | 2篇 | 2026-05-24 |
| [[rag-retrieval-augmented-generation]] | RAG 全流程：检索增强生成、Sentence Window、Auto-merging | 4篇 | 2026-05-24 |
| [[embedding-models]] | Embedding 演进：Word2Vec→BERT→Dual Encoder、对比学习 | 2篇 | 2026-05-24 |
| [[ai-agent-design-patterns]] | Agent 五大模式：Planning、Tool Use、Reflection、Multi-agent、Memory | 4篇 | 2026-05-24 |
| [[mcp-model-context-protocol]] | MCP 开放标准：Tools/Resources/Prompts 三大原语 | 3篇 | 2026-05-24 |
| [[llm-system-architecture]] | LLM 系统多步架构：输入评估→处理→输出检查、Chain/LCEL | 3篇 | 2026-05-24 |
| [[evaluation-methods]] | AI 评估：RAG Triad、LLM-as-Judge、Agent 分解评估 | 3篇 | 2026-05-24 |
| [[skills-system]] | Skills 指令扩展：Progressive Disclosure、跨平台兼容 | 2篇 | 2026-05-24 |
| [[react-pattern]] | ReAct 执行循环：Thought→Action→Observation | 3篇 | 2026-05-24 |
| [[chain-of-thought]] | CoT 逐步推理 vs Chaining Prompts 多步 pipeline | 2篇 | 2026-05-24 |

## Entities (6)

| Page | Type | Description | Sources | Updated |
|------|------|-------------|---------|---------|
| [[langchain]] | Framework | LLM 应用开发框架：Models/Prompts/Memory/Chains/Agents/LCEL | 3篇 | 2026-05-24 |
| [[langgraph]] | Framework | Agent 图编排框架：Nodes/Edges/State/Checkpointer | 2篇 | 2026-05-24 |
| [[crewai]] | Framework | 角色扮演多 Agent 框架：Agent/Task/Crew | 1篇 | 2026-05-24 |
| [[claude-code]] | Tool | Anthropic 自主编程助手：Agentic Search、CLAUDE.md、MCP | 3篇 | 2026-05-24 |
| [[anthropic]] | Organization | Claude 模型/MCP/Skills 的开发公司 | 3篇 | 2026-05-24 |
| [[deeplearning-ai]] | Education | Andrew Ng 的 AI 教育平台，短课程合作方 | 6篇 | 2026-05-24 |

## Comparisons (4)

| Page | Description | Sources | Updated |
|------|-------------|---------|---------|
| [[langgraph-vs-crewai]] | 低层图编排 vs 高层角色协作框架 | 2篇 | 2026-05-24 |
| [[encoder-only-vs-decoder-only]] | BERT(理解) vs GPT/Claude(生成) 架构对比 | 3篇 | 2026-05-24 |
| [[rag-vs-fine-tuning]] | 外部检索 vs 权重修改的知识注入方式 | 2篇 | 2026-05-24 |
| [[tools-vs-skills-vs-mcp]] | 四种 Agent 能力扩展机制定位对比 | 3篇 | 2026-05-24 |

## Articles (0)

| Page | Description | Updated |
|------|-------------|---------|
| (暂无 - 等待 Query 产出) | | |

---

## Sources Ingested (19)

| File | Title | Date |
|------|-------|------|
| transformer-llm-how-they-work | Transformer LLM 工作原理 | 2026-01-01 |
| understanding-language-models-transformers | 语言模型理解：Transformers | 2026-01-01 |
| ai-learning-roadmap | AI 学习路线科技树 | 2026-01-01 |
| attention-in-transformers-pytorch | Attention 机制与 PyTorch 实现 | 2026-01-02 |
| chatgpt-prompt-engineering | ChatGPT Prompt Engineering | 2026-01-03 |
| building-systems-chatgpt-api | 用 ChatGPT API 构建系统 | 2026-01-04 |
| langchain-llm-app-development | LangChain LLM 应用开发 | 2026-01-05 |
| functions-tools-agents-langchain | LangChain 函数/工具/Agent | 2026-01-06 |
| embedding-models-architecture-implementation | Embedding 模型架构与实现 | 2026-01-07 |
| building-evaluating-advanced-rag | 高级 RAG 构建与评估 | 2026-01-08 |
| retrieval-optimization-tokenization-vector-quantization | 检索优化：分词到向量量化 | 2026-01-09 |
| ai-agents-langgraph | LangGraph AI Agent | 2026-01-10 |
| ai-agents-langgraph-extended | LangGraph AI Agent (扩展) | 2026-01-10 |
| multi-agent-systems-crewai | CrewAI 多智能体系统 | 2026-01-11 |
| evaluating-ai-agents | AI Agent 评估 | 2026-01-12 |
| mcp-rich-context-ai-apps-anthropic | MCP 富上下文 AI 应用 | 2026-01-13 |
| claude-code-agentic-coding-assistant | Claude Code 自主编程助手 | 2026-01-14 |
| agent-skills-anthropic | Anthropic Agent Skills | 2026-01-15 |
| 2026-05-26_transformer-training-conversation | Transformer 训练原理对话 | 2026-05-26 |

---

*Last health check: 2026-05-26 (added FFN, MoE concepts; fixed broken link)*
*Total VKI pages: 28*
*Total sources: 19*
