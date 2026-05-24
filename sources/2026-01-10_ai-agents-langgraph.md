# LangGraph 中的 AI Agent

> 课程来源：DeepLearning.AI & LangChain & Tavily  
> 讲师：Harrison Chase（LangChain CEO）、Rotem Weiss（Tavily CEO）  
> 课程主题：使用 LangGraph 构建 AI Agent 的完整流程

---

## 第1课：课程介绍

### Agent 与 Agentic Workflow
- **传统 LLM 使用**：一次性从头到尾生成（zero-shot）
- **Agentic Workflow**：迭代式工作流程，多步骤协作产出更好结果

### 五大 Agent 设计模式
1. **规划（Planning）**：思考并分解步骤
2. **工具使用（Tool Use）**：调用外部工具（如搜索）
3. **反思（Reflection）**：迭代改进，自我批判
4. **多 Agent 通信（Multi-agent Communication）**：不同角色协作
5. **记忆（Memory）**：跟踪进度和结果

### 关键改进
- Function Calling LLM 使工具使用更稳定可预测
- 专为 Agent 设计的搜索工具（返回结构化答案而非链接）

> **生动比喻**：Agent 工作流就像三个人合写论文——一人列提纲，一人做调研，一人写初稿，然后互相审阅修改，反复迭代直到满意。这比让一个人一口气写完要好得多。

![Agent 示例](screenshots/01_agent_examples.jpg)
![Agentic 设计模式](screenshots/01_agentic_design_patterns.jpg)

---

## 第2课：从零构建 Agent

### ReAct 模式（Reasoning + Acting）
核心循环：**思考(Thought) -> 行动(Action) -> 暂停(Pause) -> 观察(Observation) -> 重复**

### 实现要点
1. **Agent 类**：维护消息列表，参数化系统提示
2. **系统提示**：定义可用工具、输出格式和示例
3. **工具注册**：函数名到函数的字典映射
4. **循环执行**：用正则表达式解析 LLM 输出，判断是"Action"还是"Answer"

### 代码架构
```python
class Agent:
    def __init__(self, system_prompt):
        self.messages = [system_message]
    
    def __call__(self, message):
        # 追加消息 -> 调用 LLM -> 返回结果
    
    def execute(self):
        # 调用 OpenAI API
```

### 自动化循环（query 函数）
- 用正则匹配 `Action:` 格式
- 从 known_actions 字典查找并执行工具
- 将结果格式化为 `Observation:` 传回 LLM
- 设置 `max_turns` 防止无限循环

> **生动比喻**：ReAct Agent 像一个不断"自问自答"的侦探——先想"我需要什么线索"，然后去调查（调用工具），看到结果后再想"下一步该做什么"，直到破案。

![ReAct 模式](screenshots/02_react_pattern.jpg)

---

## 第3课：LangGraph 组件

### LangGraph 核心概念

| 概念 | 说明 | 对应代码 |
|------|------|---------|
| **Node（节点）** | Agent 或函数 | `graph.add_node("llm", call_openai)` |
| **Edge（边）** | 连接节点的固定路径 | `graph.add_edge("action", "llm")` |
| **Conditional Edge（条件边）** | 根据条件选择下一个节点 | `graph.add_conditional_edges(...)` |
| **State（状态）** | 在所有节点间共享的数据 | `class AgentState(TypedDict)` |
| **Entry Point** | 图的入口 | `graph.set_entry_point("llm")` |

### Agent State 设计
```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    # operator.add 表示追加而非覆盖
```

### LangGraph 构建流程
1. 定义 State（包含 messages 列表）
2. 创建 StateGraph
3. 添加节点（LLM 节点 + Action 节点）
4. 添加条件边（检查是否有 tool_calls）
5. 添加固定边（action -> llm 循环）
6. 设置入口点并编译图
7. 使用 `model.bind_tools(tools)` 让模型感知可用工具

### 关键特性
- **并行工具调用**：现代 LLM 支持一次请求多个工具
- **顺序依赖**：当第二个查询依赖第一个结果时，自动串行执行
- **可视化**：`graph.get_graph().draw_png()` 自动生成图结构

> **生动比喻**：LangGraph 就像一个地铁线路图——节点是站点（执行任务的地方），边是轨道（固定路线），条件边是换乘站（根据情况决定去哪条线）。

![Agent 图结构](screenshots/03_agent_graph_structure.jpg)
![LangGraph 概念](screenshots/03_langgraph_concepts.jpg)

---

## 第4课：Agentic 搜索工具

### 传统搜索 vs Agentic 搜索

| 特性 | 传统搜索 | Agentic 搜索 (Tavily) |
|------|---------|---------------------|
| 返回内容 | 链接列表 | 结构化 JSON 答案 |
| 需要额外处理 | 抓取网页、解析 HTML | 直接可用 |
| 信息提取 | 手动 | 自动提取相关信息 |
| 适合对象 | 人类浏览 | Agent 消费 |

### Agentic 搜索内部流程
1. **理解问题**：将复杂查询分解为子问题
2. **选择数据源**：从多个集成中选最佳来源（如天气用天气 API）
3. **提取相关信息**：对源文档分块 + 向量搜索 top-K
4. **评分过滤**：去除不相关信息，返回精炼结果

> **生动比喻**：传统搜索像给你一堆书让你自己翻；Agentic 搜索像一个助手帮你查完资料后直接告诉你答案和出处。

![Agentic 搜索流程](screenshots/04_agentic_search_flow.jpg)
![搜索工具实现](screenshots/04_search_tool_implementation.jpg)

---

## 第5课：持久化与流式输出

### 持久化（Persistence）
- **Checkpointer**：在每个节点之间自动保存状态快照
- 实现：`SqliteSaver` / `AsyncSqliteSaver`（也支持 Redis、Postgres）
- **Thread ID**：区分不同会话，支持多用户并发
- 用途：多轮对话记忆、调试、生产部署

### 流式输出（Streaming）
- **消息级流式**：`graph.stream()` 返回每步的中间消息
- **Token 级流式**：`astream_events()` 逐 token 输出
- 事件类型：`on_chat_model_stream` 对应新 token 产生

### 实战效果
```
Thread 1: "SF天气如何?" -> 工具调用 -> 回答
Thread 1: "LA呢?" -> 自动理解上下文（因为同一thread）
Thread 2: "哪个更暖?" -> 困惑（新thread无历史）
```

> **生动比喻**：Checkpointer 就像游戏存档——你可以随时存档、读档、甚至回到之前的存档重新来过。Thread ID 则像不同玩家的存档槽位。

![Checkpointer 概念](screenshots/05_checkpointer_concept.jpg)

---

## 第6课：人机协作（Human in the Loop）

### 核心机制
- **interrupt_before**：在指定节点执行前暂停，等待人类审批
  ```python
  graph.compile(checkpointer=memory, interrupt_before=["action"])
  ```

### 高级操作

| 操作 | 说明 | 用途 |
|------|------|------|
| **暂停并审批** | 工具调用前暂停 | 安全关键操作审核 |
| **修改状态** | 更改 Agent 即将执行的操作 | 纠正 Agent 误解 |
| **时间旅行** | 回到历史状态重新执行 | 调试、探索不同分支 |
| **手动注入结果** | 模拟工具返回值 | 测试、Mock |

### 状态管理
- `get_state(config)`：获取当前状态
- `get_state_history(config)`：获取所有历史快照
- `update_state(config, values)`：修改状态
- `update_state(config, values, as_node="action")`：以某节点身份注入结果

> **生动比喻**：Human in the Loop 就像自动驾驶中的人类接管——系统自动运行，但在关键决策点人类可以审批、修改方向、甚至"倒车"回到之前的路口重新选择。

![状态记忆](screenshots/06_state_memory.jpg)

---

## 第7课：Essay Writer（多步 Agent 项目）

### 架构设计
```
Plan -> Research -> Generate -> [Reflect -> Research -> Generate]* -> End
```

### 各节点功能
1. **Plan（规划）**：生成文章大纲，只执行一次
2. **Research（研究）**：调用 Tavily 搜索相关文档
3. **Generate（生成）**：基于大纲和研究资料写文章
4. **Reflect（反思）**：对当前文章生成批评和改进建议
5. **条件判断**：是否需要继续迭代

### 复杂状态设计
```python
class AgentState(TypedDict):
    task: str              # 写作任务
    plan: str              # 文章大纲
    draft: str             # 当前草稿
    critique: str          # 批评反馈
    content: List[str]     # 研究文档（追加模式）
    revision_number: int   # 迭代次数
    max_revisions: int     # 最大迭代数
```

> **生动比喻**：Essay Writer 就像一个写作工作坊——先列提纲（Plan），然后查资料（Research），写初稿（Generate），请人评审（Reflect），再补充资料重写，直到满意为止。

![Essay Writer 流程](screenshots/07_essay_writer_flow.jpg)

---

## 第8课：LangChain 生态资源

- **LangChain 文档**：核心包 + 社区包 + 合作伙伴包
- **LangChain Hub**：社区贡献的提示模板
- **LangServe**：将 LangChain 应用部署为 Web 服务
- **LangSmith**：调试、监控、Playground
- **LangGraph 仓库**：详细文档和教程

---

## 第9课：总结与进阶架构

### 进阶 Agent 架构

| 架构 | 特点 | 适用场景 |
|------|------|---------|
| **Multi-Agent** | 多个 Agent 共享同一状态 | 协作任务 |
| **Supervisor Agent** | 一个主管分配子任务给子 Agent | 需要集中协调 |
| **Flow Engineering** | 精心设计的有向图+关键节点循环 | 特定领域优化（如编码） |
| **Plan & Execute** | 先规划后执行，可重规划 | 复杂多步任务 |
| **Language Agent Tree Search** | 树搜索 + 反思 + 回溯 | 需要探索多条路径 |

> **生动比喻**：Flow Engineering 就像建筑师设计水管系统——大部分管道方向固定（pipeline），但在关键节点设置了循环泵（迭代改进），确保整体系统高效可控。

![Flow Engineering](screenshots/09_flow_engineering.jpg)
![多 Agent 架构](screenshots/09_multi_agent_architecture.jpg)

---

## 总结表

| 课程主题 | 关键技术 | 实用建议 |
|---------|---------|---------|
| Agent 基础 | ReAct 模式（思考-行动-观察循环） | 从简单工具开始，逐步增加复杂度 |
| LangGraph | 节点+边+条件边+状态 | 先画图再写代码 |
| 搜索工具 | Tavily Agentic Search | 返回结构化数据，非链接 |
| 持久化 | Checkpointer + Thread ID | 生产环境必备 |
| Human in Loop | interrupt_before + 状态修改 | 安全关键场景加审批 |
| 复杂项目 | 多节点、复杂状态、迭代循环 | 分解为独立的 Plan/Research/Generate/Reflect |
| 进阶架构 | Multi-Agent / Supervisor / Tree Search | 根据任务复杂度选择架构 |

---

## 知识树

```
AI Agents in LangGraph
├── Agent 基础概念
│   ├── Agentic Workflow vs Zero-shot
│   ├── 五大设计模式（Planning/Tool/Reflection/Multi-agent/Memory）
│   └── ReAct 模式（Thought -> Action -> Observation 循环）
├── LangGraph 框架
│   ├── 核心三要素：Node / Edge / Conditional Edge
│   ├── Agent State（带注解的 TypedDict）
│   ├── StateGraph 创建与编译
│   └── model.bind_tools() 工具绑定
├── 工具生态
│   ├── Tavily Agentic Search（结构化结果）
│   ├── LangChain Community 工具集
│   └── 并行 vs 顺序工具调用
├── 生产化能力
│   ├── Persistence（Checkpointer 状态持久化）
│   ├── Streaming（消息级 + Token 级）
│   ├── Thread ID（多会话隔离）
│   └── Human in the Loop
│       ├── interrupt_before 暂停审批
│       ├── 状态修改与注入
│       └── 时间旅行（回溯历史状态）
├── 实战项目：Essay Writer
│   ├── Plan -> Research -> Generate -> Reflect 循环
│   ├── 复杂状态管理
│   └── 可控迭代次数
└── 进阶架构
    ├── Multi-Agent（共享状态协作）
    ├── Supervisor Agent（主管调度）
    ├── Flow Engineering（有向图+关键循环）
    ├── Plan & Execute（先规划后执行）
    └── Language Agent Tree Search（树搜索+回溯）
```
