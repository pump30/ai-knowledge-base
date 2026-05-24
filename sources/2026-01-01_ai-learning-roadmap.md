# AI 学习路线科技树

> 适合人群：程序员，熟练使用AI工具，但对AI底层逻辑不够熟悉
> 总计122门课，主路线20门课（约30小时），支线根据兴趣方向选择

---

## 🌳 科技树总览

```
                        ┌─────────────────────────────────────────┐
                        │         Level 0: 基础认知                 │
                        │  Generative AI for Everyone (5h)         │
                        │  (非技术概览，可选跳过)                    │
                        └────────────────────┬────────────────────┘
                                             │
                        ┌────────────────────▼────────────────────┐
                        │         Level 1: 底层原理 ⭐必修           │
                        │  How Transformer LLMs Work (1.5h)        │
                        │  Attention in Transformers (1h)           │
                        └────────────────────┬────────────────────┘
                                             │
              ┌──────────────────────────────┼──────────────────────────────┐
              │                              │                              │
   ┌──────────▼──────────┐    ┌─────────────▼─────────────┐   ┌───────────▼───────────┐
   │  Level 2A: 训练篇    │    │  Level 2B: 使用篇 ⭐必修   │   │  Level 2C: 嵌入/检索   │
   │                      │    │                           │   │                       │
   │ Pretraining (1.3h)   │    │ ChatGPT Prompt Eng (1.5h) │   │ Embedding Models (50m)│
   │ Finetuning (1.4h)    │    │ Building Systems (1.7h)   │   │ Vector Databases (55m)│
   │ Post-training (1.3h) │    │                           │   │                       │
   │ RLHF (1.2h)          │    └─────────────┬─────────────┘   └───────────┬───────────┘
   │ GRPO (1.4h)           │                   │                              │
   └──────────┬──────────┘                   │                              │
              │                              │                              │
              │              ┌───────────────▼───────────────┐              │
              │              │  Level 3: 框架与工具 ⭐必修      │              │
              │              │                               │              │
              │              │ LangChain for LLM Dev (1.6h)  │◄─────────────┘
              │              │ Functions, Tools & Agents(1.7h)│
              │              └───────────────┬───────────────┘
              │                              │
              │              ┌───────────────▼───────────────┐
              │              │  Level 4: RAG深入 ⭐必修        │
              │              │                               │
              │              │ Building Advanced RAG (2h)     │
              │              │ Retrieval Optimization (1.5h)  │
              │              └───────────────┬───────────────┘
              │                              │
              │              ┌───────────────▼───────────────┐
              │              │  Level 5: Agent系统 ⭐必修      │
              │              │                               │
              │              │ AI Agents in LangGraph (1.5h)  │
              │              │ Multi AI Agent crewAI (2.7h)   │
              │              │ Evaluating AI Agents (2.3h)    │
              │              └───────────────┬───────────────┘
              │                              │
              │              ┌───────────────▼───────────────┐
              │              │  Level 6: 前沿工具 ⭐必修       │
              │              │                               │
              │              │ MCP with Anthropic (1.6h)      │
              │              │ Claude Code (1.8h)             │
              │              │ Agent Skills (2.3h)            │
              │              └───────────────────────────────┘
```

---

## 📚 主路线详细（按学习顺序）

### Phase 1: 理解LLM是什么（2.5小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 1 | **How Transformer LLMs Work** | 94min | Transformer架构全景：从词嵌入→注意力→MoE，理解LLM的"大脑结构" |
| 2 | **Attention in Transformers: Concepts and Code in PyTorch** | 66min | 用PyTorch手写注意力机制，真正理解Self-Attention的数学原理 |

### Phase 2: 高效使用LLM（3.2小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 3 | **ChatGPT Prompt Engineering for Developers** | 90min | Prompt工程经典入门，Andrew Ng亲授，系统化你已有的使用经验 |
| 4 | **Building Systems with the ChatGPT API** | 105min | 从单次prompt到构建完整系统：分类→审核→推理→链式prompt |

### Phase 3: 应用开发框架（3.3小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 5 | **LangChain for LLM Application Development** | 98min | LLM应用开发标准框架：Memory、Chains、RAG、Agents |
| 6 | **Functions, Tools and Agents with LangChain** | 104min | Function Calling + 工具使用 + 对话式Agent，进阶核心 |

### Phase 4: RAG（检索增强生成）（3.5小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 7 | **Embedding Models: from Architecture to Implementation** | 50min | 理解向量嵌入的原理：token vs sentence embedding，双编码器 |
| 8 | **Building and Evaluating Advanced RAG** | 115min | 高级RAG技术：句子窗口检索、自动合并、评估pipeline |
| 9 | **Retrieval Optimization: Tokenization to Vector Quantization** | 93min | 检索优化全链路：分词→向量压缩→高效搜索 |

### Phase 5: Agent系统（6.5小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 10 | **AI Agents in LangGraph** | 92min | 图结构Agent：状态管理、持久化、流式输出、人机协作 |
| 11 | **Multi AI Agent Systems with crewAI** | 161min | 多Agent协作：角色定义、任务分配、团队协作模式 |
| 12 | **Evaluating AI Agents** | 136min | Agent评估体系：路由/技能/轨迹评估，LLM-as-judge |

### Phase 6: 前沿Agent工具（6.1小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 13 | **MCP: Build Rich-Context AI Apps with Anthropic** | 98min | 模型上下文协议：让AI连接任意外部系统的标准协议 |
| 14 | **Claude Code: A Highly Agentic Coding Assistant** | 110min | 你正在用的工具的深度教程！hooks、GitHub集成、多任务并行 |
| 15 | **Agent Skills with Anthropic** | 139min | 给Agent装备"技能"：Skills vs Tools vs MCP，自定义技能开发 |

### Phase 7（选修进阶）: LLM训练原理（5.2小时）
| # | 课程 | 时长 | 为什么学 |
|---|------|------|---------|
| 16 | **Pretraining LLMs** | 79min | 预训练全流程：数据准备→模型初始化→训练→评估 |
| 17 | **Finetuning Large Language Models** | 85min | 微调为什么有效、何时该微调、LoRA等技术 |
| 18 | **Post-training of LLMs** | 76min | 后训练三件套：SFT→DPO→Online RL |
| 19 | **Reinforcement Fine-Tuning LLMs With GRPO** | 83min | GRPO强化学习微调，DeepSeek使用的关键技术 |
| 20 | **Reinforcement Learning From Human Feedback** | 72min | RLHF原理：reward model→PPO优化→对齐人类偏好 |

---

## 🌿 支线路线（按兴趣方向）

### 支线A: 多模态AI
- Large Multimodal Model Prompting with Gemini (2h)
- Multimodal RAG: Chat with Videos (1h)
- Introducing Multimodal Llama 3.2 (1.3h)
- Building Multimodal Search and RAG (1.4h)
- Prompt Engineering for Vision Models (1.4h)
- Multi-vector Image Retrieval (1.5h)

### 支线B: AI编码与Vibe Coding
- Vibe Coding 101 with Replit (1.5h)
- Build Apps with Windsurf's AI Coding Agents (1.2h)
- Gemini CLI: Code & Create (1.2h)
- Building Coding Agents with Tool Execution (1.4h)
- Jupyter AI: AI Coding in Notebooks (44min)

### 支线C: 高级Agent架构
- Practical Multi AI Agents (crewAI进阶) (2.7h)
- LLMs as Operating Systems: Agent Memory (1.4h)
- Long-Term Agentic Memory With LangGraph (1h)
- Agent Memory: Building Memory-Aware Agents (2h)
- Building AI Browser Agents (55min)
- Building toward Computer Use with Anthropic (1.6h)
- DSPy: Build and Optimize Agentic Apps (49min)
- A2A: The Agent2Agent Protocol (1.5h)

### 支线D: 生产部署与安全
- Improving Accuracy of LLM Applications (1.7h)
- Safe and reliable AI via guardrails (1.5h)
- Red Teaming LLM Applications (1.3h)
- Governing AI Agents (1.5h)
- LLMOps (1.4h)
- Automated Testing for LLMOps (52min)

### 支线E: 模型优化与推理效率
- Quantization Fundamentals (1h)
- Quantization in Depth (2.2h)
- Efficiently Serving LLMs (2.5h)
- Introduction to on-device AI (1h)
- Efficient Inference with SGLang (1.3h)

### 支线F: 特定框架/工具深入
- Building AI Applications With Haystack (1.4h)
- Building Agentic RAG with Llamaindex (44min)
- Pydantic for LLM Workflows (1.3h)
- Getting Structured LLM Output (1.2h)
- Build AI Apps with MCP Server: Box Files (36min)
- Orchestrating Workflows for GenAI Applications (1.5h)
- Building with Llama 4 (1h)

### 支线G: 知识图谱
- Knowledge Graphs for RAG (2h)
- Agentic Knowledge Graph Construction (3h)
- Knowledge Graphs for AI Agent API Discovery (1.2h)

### 支线H: 语音与实时AI
- Building AI Voice Agents for Production (50min)
- Building Live Voice Agents with Google's ADK (1.3h)

---

## ⏱️ 时间估算

| 路线 | 总时长 | 建议周期 |
|------|--------|---------|
| 主路线 Phase 1-6 | ~25小时 | 2-3周（每天1-2小时） |
| 主路线 + Phase 7 | ~30小时 | 3-4周 |
| 主路线 + 1条支线 | ~35小时 | 4周 |
| 全部 | ~200+小时 | 3-4个月 |

---

## 💡 学习建议

1. **Phase 1 不要跳过** - 你说对底层不熟，这两门课能在2.5小时内给你一个完整的Transformer心智模型
2. **Phase 2-3 可能觉得简单** - 因为你已经在熟练使用AI了，但系统化知识能帮你发现之前的盲区
3. **Phase 4-5 是你的核心价值区** - 作为程序员，RAG和Agent是你能立刻落地的技术
4. **Phase 6 最前沿** - MCP和Claude Code是2025年最热的方向，你已经在用了，学完能用得更好
5. **Phase 7 按需** - 如果你只想"用"AI不需要"训"AI，可以推迟这部分
