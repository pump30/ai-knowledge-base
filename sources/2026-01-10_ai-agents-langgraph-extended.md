# AI Agents in LangGraph - 课程学习笔记

> **课程来源**: DeepLearning.AI x LangChain x Tavily  
> **讲师**: Harrison Chase (LangChain CEO), Rotem Weiss (Tavily CEO), Andrew Ng  
> **核心主题**: 使用 LangGraph 构建 AI Agent（智能体）的完整方法论

---

## 目录

1. [课程概述与 Agent 设计模式](#lesson-1)
2. [从零构建 Agent](#lesson-2)
3. [LangGraph 核心组件](#lesson-3)
4. [Agentic Search 搜索工具](#lesson-4)
5. [持久化与流式输出](#lesson-5)
6. [Human-in-the-Loop 人机协作](#lesson-6)
7. [Essay Writer 实战项目](#lesson-7)
8. [LangChain 资源指南](#lesson-8)
9. [总结与高级架构](#lesson-9)

---

## Lesson 1: 课程概述与 Agent 设计模式 {#lesson-1}

### 核心观点

课程介绍了 AI Agent 的概念以及近一年来使其变得实用的两个关键进步：
1. **Function Calling LLMs** - 使工具调用更加可预测和稳定
2. **专为 Agent 设计的搜索工具** - 返回结构化答案而非链接

### Agentic Workflow（智能体工作流）

传统 LLM 使用方式是"一次性生成"（类似让人从头写到尾不能回退），而 **Agentic Workflow** 则是**迭代式**的协作流程。

> **生动比喻**: 想象三个人合作写论文 - 一个人做计划，一个人查资料，一个人写初稿，然后互相审阅修改。这种反复迭代的方式就是 Agentic Workflow，比"一气呵成"好得多。

### 五大设计模式

| 设计模式 | 说明 | 比喻 |
|---------|------|------|
| **Planning（规划）** | 思考要采取的步骤 | 写论文前先列大纲 |
| **Tool Use（工具使用）** | 知道有哪些工具及如何使用 | 知道去哪个图书馆找什么书 |
| **Reflection（反思）** | 迭代改进结果 | 写完初稿后自我审阅修改 |
| **Multi-agent（多智能体）** | 多个 Agent 各司其职 | 团队中每人承担不同角色 |
| **Memory（记忆）** | 跟踪多步骤的进度和结果 | 记住之前做了什么决策 |

### Agent 示例架构

课程展示了三种 Agent 架构：
- **ReAct** (Reasoning + Acting) - 早期范式
- **Self-Refine** - 迭代精化
- **AlphaCoding** - 流程工程（Flow Engineering）

**核心发现**: 所有这些 Agent 的行为都由**循环图（Cyclic Graph）**定义，这正是 LangGraph 诞生的原因。

---

## Lesson 2: 从零构建 Agent {#lesson-2}

### ReAct 模式详解

**ReAct = Reasoning + Acting**

循环流程：
```
Thought(思考) -> Action(行动) -> PAUSE(暂停) -> Observation(观察) -> 重复...直到 Answer(回答)
```

> **生动比喻**: ReAct 模式就像一个侦探办案 - 先分析线索（Thought），然后去调查某个方向（Action），等待结果回来（PAUSE），看到新证据（Observation），再决定下一步调查什么，直到破案（Answer）。

### Agent 代码结构

Agent 类的核心设计：
1. **系统消息（System Message）**: 定义 Agent 的行为规则和可用工具
2. **消息列表（Messages）**: 累积所有交互历史
3. **执行方法（Execute）**: 调用 LLM 获取响应

### 自动化循环

关键代码逻辑：
- 用**正则表达式**解析 LLM 的输出，判断是"Action"还是"Answer"
- 如果是 Action，执行对应工具，将结果作为 Observation 反馈
- 如果是 Answer，结束循环返回结果
- 设置 `max_turns` 防止无限循环

### 重要区分：LLM vs Runtime

| 职责归属 | 具体工作 |
|---------|---------|
| **LLM（大模型）** | 思考、推理、决定调用哪个工具 |
| **Runtime（运行时代码）** | 执行工具、管理消息列表、控制循环 |

---

## Lesson 3: LangGraph 核心组件 {#lesson-3}

### LangGraph 三大核心概念

| 概念 | 说明 | 比喻 |
|------|------|------|
| **Nodes（节点）** | Agent 或函数，执行具体操作 | 流水线上的工位 |
| **Edges（边）** | 连接节点，定义执行顺序 | 流水线上的传送带 |
| **Conditional Edges（条件边）** | 根据条件决定下一步走向 | 流水线上的分拣器 |

### Agent State（智能体状态）

Agent State 是 LangGraph 最重要的概念之一：
- 在图的所有节点和边中都可以访问
- 可以持久化存储，支持随时恢复
- 通过 **annotation（注解）** 控制更新方式

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    # operator.add 表示新消息是"追加"而非"覆盖"
```

> **生动比喻**: Agent State 就像一个共享白板 - 团队每个成员都能看到上面的内容，每个人完成工作后在白板上添加新信息。`operator.add` 就像"只能往白板上加内容"的规则。

### LangGraph Agent 结构

构建一个基于 LangGraph 的 Agent 需要三个函数：
1. **call_openai** - LLM 节点，调用语言模型
2. **exists_action** - 条件边，判断是否需要执行工具
3. **take_action** - Action 节点，执行工具调用

### 关键特性

- **bind_tools**: 告诉模型有哪些工具可用
- **parallel tool calling**: 现代模型支持并行调用多个工具
- **graph.compile()**: 将图编译为可执行的 LangChain Runnable
- 可视化：`graph.get_graph().draw_png()` 自动生成图的可视化

---

## Lesson 4: Agentic Search 搜索工具 {#lesson-4}

### 传统搜索 vs Agentic Search

| 对比维度 | 传统搜索 | Agentic Search |
|---------|---------|----------------|
| **返回内容** | 链接列表 | 结构化答案 + 来源 |
| **适用对象** | 人类（需要点击链接） | Agent（需要直接答案） |
| **数据格式** | HTML 页面 | JSON 结构化数据 |
| **处理复杂查询** | 需要人工分步 | 自动分解子问题 |

### Agentic Search 内部工作原理

Tavily 搜索工具的工作步骤：
1. **理解问题** - 将复杂查询分解为子问题
2. **选择最佳来源** - 根据问题类型选择合适的 API（如天气问题用天气 API）
3. **提取相关信息** - 通过 chunking + 向量搜索获取 top-K 相关片段
4. **评分过滤** - 对结果打分，过滤不相关内容

> **生动比喻**: 传统搜索就像去图书馆，管理员只告诉你"去第三排书架找"（给你链接）。而 Agentic Search 像一个研究助理，直接帮你把答案找出来、整理好交给你。

### 代码对比

**传统搜索（DuckDuckGo）**: 
- 返回链接 -> 需要抓取网页 -> 需要解析 HTML -> 还是很冗长

**Agentic Search（Tavily）**: 
- 一步到位返回简洁的结构化 JSON 数据，Agent 可以直接使用

---

## Lesson 5: 持久化与流式输出 {#lesson-5}

### 持久化（Persistence）

**为什么需要持久化？**
- 长时间运行的 Agent 需要保存中间状态
- 支持多会话（多用户）并行
- 可以随时恢复到之前的状态

**Checkpointer（检查点器）** 在每个节点之间自动保存状态快照。

```python
from langgraph.checkpoint.sqlite import SqliteSaver
memory = SqliteSaver.from_conn_string(":memory:")
# 传入 graph.compile(checkpointer=memory)
```

**Thread ID（线程ID）** 用于区分不同会话：
```python
thread_config = {"configurable": {"thread_id": "1"}}
```

> **生动比喻**: Checkpointer 就像游戏的"存档系统"。每走一步都自动存档，你可以随时读档回到之前的状态。Thread ID 就是不同玩家的存档槽位。

### 流式输出（Streaming）

两种流式输出模式：

| 模式 | 说明 | 使用方法 |
|------|------|---------|
| **消息流** | 每完成一个节点就输出 | `graph.stream(input, config)` |
| **Token 流** | 每生成一个 token 就输出 | `graph.astream_events(input, config)` |

Token 流需要使用异步版本（`AsyncSqliteSaver`），通过过滤 `on_chat_model_stream` 事件获取实时 token。

---

## Lesson 6: Human-in-the-Loop 人机协作 {#lesson-6}

### 核心机制：interrupt_before

通过在编译图时设置 `interrupt_before`，可以在执行特定节点前暂停：

```python
graph.compile(checkpointer=memory, interrupt_before=["action"])
```

> **生动比喻**: 这就像一个需要上级审批的流程 - Agent 准备好了要执行某个操作，但在真正执行前先"举手"等人类确认。人类可以说"继续"、"修改后继续"或"不要做了"。

### 状态快照与时间旅行

**状态管理命令**：

| 命令 | 功能 |
|------|------|
| `get_state(config)` | 获取当前状态 |
| `get_state_history(config)` | 获取所有状态快照的迭代器 |
| `update_state(config, values)` | 更新状态 |

### 三大 Human-in-the-Loop 模式

1. **审批/拒绝**: 在工具调用前暂停，等待人类确认
2. **修改状态**: 人类可以修改 Agent 即将执行的操作（如修正搜索词）
3. **时间旅行**: 回到之前的状态重新执行，或从某个历史状态分支出去

**高级操作 - 模拟工具响应**:
```python
# 假装我们就是 action 节点，直接给出"工具结果"
graph.update_state(config, state_update, as_node="action")
```

> **生动比喻**: 时间旅行就像 Git 的版本控制 - 你可以 checkout 到之前的 commit，在那个基础上创建新的分支，走一条不同的路。

---

## Lesson 7: Essay Writer 多 Agent 实战项目 {#lesson-7}

### 整体架构

这是一个完整的**多 Agent 协作系统**，包含以下步骤：

```
Plan(规划) -> Research(研究) -> Generate(生成) -> [判断] 
                                                    |-> 结束
                                                    |-> Reflect(反思) -> Research Critique(批判研究) -> Generate(重新生成)
```

### 复杂 Agent State

```python
class AgentState(TypedDict):
    task: str           # 用户输入的任务
    plan: str           # 规划 Agent 生成的计划
    draft: str          # 当前草稿
    critique: str       # 反思 Agent 的批评
    content: List[str]  # Tavily 搜索得到的文档
    revision_number: int # 当前修订次数
    max_revisions: int   # 最大修订次数
```

### 五个专业 Agent（节点）

| Agent | 职责 | Prompt 角色 |
|-------|------|------------|
| **Planner** | 生成论文大纲 | 规划者 |
| **Research Plan** | 根据计划生成搜索查询并调用 Tavily | 研究员 |
| **Generator** | 根据计划和研究内容撰写论文 | 写手 |
| **Reflector** | 对草稿进行批评和建议 | 审稿人 |
| **Research Critique** | 根据批评意见做补充研究 | 补充研究员 |

### 图可视化

### 关键设计细节

- 使用 **Pydantic + structured output** 确保 LLM 返回结构化的查询列表
- 使用 **Tavily Client**（而非 Tool）进行更灵活的搜索
- **should_continue** 条件边：`revision_number > max_revisions` 时结束
- 支持 GUI 界面进行交互式论文写作

> **生动比喻**: 这个系统就像一个编辑部 - 有人定选题（Planner），有人查资料（Researcher），有人写稿（Generator），有编辑审稿（Reflector），审稿后再查漏补缺（Research Critique），然后重写，直到定稿发布。

---

## Lesson 8: LangChain 生态资源 {#lesson-8}

### 推荐学习资源

| 资源 | 说明 |
|------|------|
| **LangChain 文档** | 包和服务的高层概览 |
| **LangChain Hub** | 社区贡献的 Prompt 模板 |
| **LangGraph Repo** | 深入的文档、教程和 how-to 指南 |
| **LangSmith** | 调试、监控和 Playground |
| **LangServe** | 将 LangChain 应用变成 Web 服务器 |
| **DeepLearning.AI 前序课程** | "Functions, Tools and Agents with LangChain" |

### LangChain 生态包结构

- **LangChain Core**: 核心抽象
- **LangChain Community**: 社区集成（如 Tavily）
- **LangChain Partner Packages**: 合作伙伴包（如 LangChain-OpenAI）

---

## Lesson 9: 总结与高级 Agent 架构 {#lesson-9}

### 多 Agent 架构

| 架构 | 特点 | 适用场景 |
|------|------|---------|
| **Multi-Agent（多智能体）** | 多个 Agent 共享同一状态 | 协作完成复杂任务 |
| **Supervisor（监督者）** | 一个主 Agent 协调子 Agent | 需要智能路由和规划 |

### Flow Engineering（流程工程）

来自 AlphaCodium 论文的概念：
- 大部分是**有向管道**（Pipeline）
- 在关键节点有**迭代循环**
- 为特定问题设计定制化的信息流

> **生动比喻**: Flow Engineering 就像设计工厂的生产流水线 - 大部分工序是固定顺序的（流水线），但某些质检环节如果不合格就需要返工（循环），整体设计要根据具体产品来定制。

### Plan-and-Execute 模式

先做显式的规划，然后逐步执行：
1. 制定计划（列出步骤）
2. 子 Agent 执行第一步 -> 返回结果
3. 可能更新计划
4. 执行下一步... 直到计划完成
5. 检查是否需要重新规划

### Language Agent Tree Search (LATS)

一种在动作空间中做**树搜索**的方法：
- 生成动作 -> 反思 -> 选择分支
- 可以回溯（backpropagate）更新父节点的信息
- 持久化在这里尤为关键（需要回到之前的状态）

---

## 核心概念总结表

| 概念 | 定义 | 关键代码/接口 |
|------|------|--------------|
| **ReAct** | Reasoning + Acting 循环模式 | Thought -> Action -> Observation |
| **LangGraph** | 用于构建循环图的 Agent 框架 | `StateGraph`, `add_node`, `add_edge` |
| **Node** | 图中的执行单元（函数/Agent） | `graph.add_node("name", func)` |
| **Edge** | 节点间的连接 | `graph.add_edge("a", "b")` |
| **Conditional Edge** | 条件路由 | `graph.add_conditional_edges(...)` |
| **Agent State** | 图中共享的状态对象 | `TypedDict` + `Annotated` |
| **Checkpointer** | 持久化状态的检查点器 | `SqliteSaver`, `graph.compile(checkpointer=...)` |
| **Thread ID** | 区分不同会话的标识 | `{"configurable": {"thread_id": "1"}}` |
| **Streaming** | 实时输出中间结果 | `graph.stream()`, `graph.astream_events()` |
| **interrupt_before** | 在节点前暂停等待人类输入 | `graph.compile(interrupt_before=["action"])` |
| **Time Travel** | 回到历史状态重新执行 | `get_state_history()` + `stream(None, old_config)` |
| **Tavily** | 专为 Agent 设计的搜索引擎 | 返回结构化 JSON 而非链接 |
| **bind_tools** | 告诉 LLM 可用的工具 | `model.bind_tools(tools)` |
| **Structured Output** | 强制 LLM 返回特定格式 | `model.with_structured_output(Schema)` |

---

## 知识架构图

```
AI Agents in LangGraph
|
|-- 基础概念
|   |-- Agentic Workflow（迭代式工作流 vs 一次性生成）
|   |-- 五大设计模式: Planning / Tool Use / Reflection / Multi-agent / Memory
|   |-- ReAct Pattern: Thought -> Action -> PAUSE -> Observation -> ...
|   |-- LLM 职责 vs Runtime 职责
|
|-- LangGraph 框架
|   |-- 核心三要素: Nodes / Edges / Conditional Edges
|   |-- Agent State（状态管理）
|   |   |-- TypedDict 定义
|   |   |-- Annotated[..., operator.add] 追加模式
|   |   |-- 无注解 = 覆盖模式
|   |-- Graph 构建流程
|   |   |-- StateGraph(AgentState)
|   |   |-- add_node / add_edge / add_conditional_edges
|   |   |-- set_entry_point
|   |   |-- compile() -> Runnable
|   |-- 可视化: get_graph().draw_png()
|
|-- 工具生态
|   |-- Tavily Search（Agentic Search）
|   |   |-- 分解子问题 -> 选择来源 -> 提取信息 -> 评分过滤
|   |   |-- 返回结构化 JSON（非 HTML 链接）
|   |-- Function Calling / bind_tools
|   |-- Parallel Tool Calling（并行工具调用）
|   |-- Structured Output（结构化输出）
|
|-- 生产级特性
|   |-- Persistence（持久化）
|   |   |-- Checkpointer: SqliteSaver / Redis / Postgres
|   |   |-- Thread ID: 多会话管理
|   |   |-- State Snapshot: 状态快照
|   |-- Streaming（流式输出）
|   |   |-- 消息级: graph.stream()
|   |   |-- Token 级: graph.astream_events()
|   |-- Human-in-the-Loop（人机协作）
|       |-- interrupt_before: 审批/拒绝
|       |-- update_state: 修改 Agent 行为
|       |-- Time Travel: 回到历史状态
|       |-- Mock Tool Response: 模拟工具返回
|
|-- 高级架构模式
|   |-- Multi-Agent（共享状态）
|   |-- Supervisor Agent（监督者模式）
|   |-- Flow Engineering（流程工程）
|   |-- Plan-and-Execute（规划执行）
|   |-- Language Agent Tree Search (LATS)
|
|-- 实战项目: Essay Writer
    |-- 5 个专业 Agent: Planner / Researcher / Generator / Reflector / Critique Researcher
    |-- 复杂 State: task / plan / draft / critique / content / revision_number
    |-- 条件循环: should_continue 判断是否继续修订
    |-- GUI 交互界面
```

---

## 学习建议

1. **先理解 ReAct 模式** - 这是所有 Agent 的基础思想
2. **动手实现 Lesson 2 的纯 Python Agent** - 理解 LLM 和 Runtime 的分工
3. **掌握 LangGraph 的三要素** - Node, Edge, Conditional Edge
4. **重点实践 Persistence + Human-in-the-Loop** - 这是生产级 Agent 的关键
5. **用 Essay Writer 作为模板** - 学习如何设计多 Agent 协作系统

---

*笔记生成日期: 2026-05-20*
