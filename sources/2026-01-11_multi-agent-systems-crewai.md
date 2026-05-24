# Multi AI Agent Systems with crewAI - 学习笔记

> **课程来源**: DeepLearning.AI  
> **讲师**: Joao Moura (crewAI 创始人兼CEO), Andrew Ng  
> **核心主题**: 使用 crewAI 框架构建多智能体协作系统

---

## 课程总览

本课程系统讲解了如何使用 crewAI 开源框架构建多 AI Agent 系统。核心学习内容包括：角色扮演(Role-Playing)、工具使用(Tool Use)、记忆(Memory)、护栏(Guardrails)和协作(Collaboration)。通过多个实战项目（文章撰写、客户支持、销售外展、活动策划、金融分析、简历定制），逐步深入掌握多智能体系统的设计与实现。

![课程核心构建模块](screenshots/01_building_blocks.jpg)

---

## 第1课: 课程介绍

Andrew Ng 和 Joao Moura 共同介绍了课程背景。

**核心观点**:
- AI Agent 工作流将是近期 AI 进步的关键驱动力
- 多 Agent 工作流将复杂任务分解为子任务，每个 Agent 扮演特定角色
- 设计多 Agent 系统就像成为一名"Agent 经理"——定义目标、角色和成功标准

**生动比喻**: 把自己想象成被提拔的经理人——你不再亲自写代码，而是识别目标、分配角色、定义期望。这是从"工程师思维"到"管理者思维"的转变。

---

## 第2课: 概述 - 什么是 Agentic Automation

![Agentic Automation 对比传统自动化](screenshots/02_agentic_automation.jpg)

**传统自动化 vs. Agentic 自动化**:
- 传统: 从 A 到 B 写死逻辑，边缘情况越多代码越复杂（if-else 地狱）
- Agentic: 不需要绘制完整地图，只需展示选项，Agent 自主导航

**Regular Applications vs. AI Applications**:
| 维度 | 传统应用 | AI 应用 |
|------|---------|---------|
| 输入 | 强类型(整数/字符串) | 模糊输入(Fuzzy) |
| 转换 | 确定性数学运算 | 模糊转换(LLM决策) |
| 输出 | 可预测、可复现 | 模糊输出(形式多变) |

**生动比喻**: 传统编程像开一辆固定路线的公交车，AI Agent 像叫了一辆出租车——你只说目的地，司机自己选路线，虽然每次走的路可能不同，但都能到达。

**实际案例 - 销售线索处理**:
- 旧方式: 表单 -> 规则引擎(公司大小?地区?) -> 固定评分
- Agent方式: 表单 -> AI研究(搜索互联网) -> 智能比较 -> 动态评分 -> 生成谈话要点

![简历定制示例](screenshots/02_resume_example.jpg)

---

## 第3课: AI Agents 是什么

![Agent 核心组件：LLM + 工具](screenshots/03_agent_with_tools.jpg)

**Agent 的诞生过程**:
1. LLM 预测下一个 token
2. 通过与用户交互获得反馈来改进输出
3. 当 LLM 能自我提问并回答时——**Agent 诞生了**
4. 加上工具(Tools)使用能力 -> **完整的 Agent**

**生动比喻**: 普通 LLM 像一个闭卷考试的学生，只能凭记忆答题；Agent 则像一个可以查资料、打电话、上网搜索的研究员——不仅能思考，还能行动。

**多 Agent 系统的优势**:
- 每个 Agent 专注做一件事并做好（专家分工）
- 不同 Agent 可使用不同 LLM（Llama-3做研究，GPT-4做写作）
- 可使用自定义微调模型驱动特定 Agent

![crewAI 框架概览](screenshots/03_crewai_framework.jpg)

**crewAI 框架特点**:
- 简单的结构化概念（Agent, Task, Crew）
- 提供组装模式
- 内置丰富工具
- 支持自定义工具
- 提供生产部署平台

---

## 第4课: 实战 - 研究与写作 Crew

**第一个多Agent系统**: Planner + Writer + Editor

```python
# 核心三要素
from crewai import Agent, Task, Crew

# 创建 Agent: role, goal, backstory
planner = Agent(role="Content Planner", goal="...", backstory="...")
writer = Agent(role="Content Writer", goal="...", backstory="...")
editor = Agent(role="Editor", goal="...", backstory="...")

# 创建 Task: description, expected_output, agent
plan_task = Task(description="...", expected_output="...", agent=planner)

# 组装 Crew 并执行
crew = Crew(agents=[...], tasks=[...], verbose=2)
result = crew.kickoff(inputs={"topic": "Artificial Intelligence"})
```

**关键学习点**:
1. Agent 通过角色扮演表现更好
2. 应聚焦于目标和期望
3. 一个 Agent 可执行多个任务
4. 任务和 Agent 应保持粒度化
5. 任务可并行、串行或层级式执行
6. Crew 默认按任务顺序串行执行，前一个任务的输出作为后一个的输入

---

## 第5课: AI Agent 的六大关键要素

![六大要素](screenshots/05_six_elements.jpg)

### 1. 角色扮演 (Role Playing)
- 设置精确的 role/goal/backstory 直接影响输出质量
- 使用专业关键词（如 "FINRA approved" 而非简单 "financial analyst"）
- **类比**: 就像演员入戏越深表演越好，Agent 的"人设"越丰满，回答越专业

### 2. 聚焦 (Focus)
- 不要让一个 Agent 做所有事
- 工具不要太多（避免选择困难）
- 上下文不要太杂（信息过载导致幻觉）
- **类比**: 杂货店收银员 vs 米其林大厨——专注才能精通

### 3. 工具 (Tools)
- 允许 Agent 与外部世界交互
- 不要给太多工具，选择关键工具
- 小模型+太多工具 = 灾难

### 4. 合作 (Cooperation)
- Agent 间相互反馈产生更好结果
- 可以委托任务、提问
- QA Agent 模式非常有效

### 5. 护栏 (Guardrails)
- 防止幻觉
- 防止无限循环
- 防止重复使用同一工具
- crewAI 框架级别内置

### 6. 记忆 (Memory)

![记忆类型](screenshots/05_memory_types.jpg)

| 记忆类型 | 生命周期 | 作用 |
|---------|---------|------|
| 短期记忆 | 单次 Crew 执行期间 | Agent 间共享上下文 |
| 长期记忆 | 永久存储(本地数据库) | 从历史执行中学习改进 |
| 实体记忆 | 单次执行期间 | 记录讨论的实体(公司、人物等) |

**生动比喻**: 短期记忆像白板——会议结束就擦掉；长期记忆像笔记本——每次复盘后记下经验教训；实体记忆像通讯录——记住谈话中提到的人和公司。

---

## 第6课: 实战 - 客户支持自动化

构建了一个支持 Agent + QA Agent 的客户支持系统。

**核心设计模式**:
- Support Agent: 回答客户问题（使用 docs 搜索工具）
- QA Agent: 审核答案质量，可以委托回 Support Agent 修改

**实战亮点**:
- `allow_delegation=False` 限制 Agent 不能委托
- QA Agent 保持 `allow_delegation=True` 以便退回修改
- 使用 `memory=True` 启用三种记忆
- 工具: SerperDevTool, ScrapeWebsiteTool, WebsiteSearchTool

**工具分配策略**:
- Agent 级别: 工具适用于该 Agent 的所有任务
- Task 级别: 工具仅在执行特定任务时可用（覆盖 Agent 工具）

---

## 第7课: Agent 创建的心智框架

![经理人思维框架](screenshots/07_manager_framework.jpg)

**核心框架——像经理人一样思考**:

1. **明确目标**: 你试图完成什么？
2. **定义流程**: 如何达成目标？
3. **雇佣人才**: 你会雇谁来完成这个工作？
4. **设定角色**: 这些人的职位、目标、背景是什么？

**避免通用角色**:
| 差的示例 | 好的示例 |
|---------|---------|
| Researcher | HR Research Specialist |
| Writer | Senior Copywriter |
| Financial Analyst | FINRA Approved Analyst |

**生动比喻**: 创建 Agent 就像组建一支足球队——你不会让前锋去守门，每个位置都需要专业球员。关键词的选择就像球衣号码，让每个人立刻知道自己的角色。

---

## 第8课: 工具的三大关键特质

![工具三大品质](screenshots/08_tool_qualities.jpg)

### 1. 多功能性 (Versatile)
- 能处理 LLM 传来的各种输入格式
- crewAI 自动转换参数类型

### 2. 容错性 (Fault-Tolerant)
- 异常不应停止整个执行流程
- crewAI 将错误信息反馈给 Agent，让其自我修正
- **类比**: 好的助手不会因为一个电话没打通就罢工，而是尝试其他方式联系

### 3. 缓存 (Caching)
- 跨 Agent 缓存：相同工具+相同参数 = 直接返回缓存
- 避免重复 API 调用
- 节省时间和费率限制

---

## 第9课: 实战 - 客户外展 Campaign

**场景**: 给潜在客户创建个性化外展邮件

**Agent 设计**:
- Sales Rep Agent: 研究和分析潜在客户
- Lead Sales Rep Agent: 创建个性化沟通内容

**自定义工具示例**:
```python
from crewai_tools import BaseTool

class SentimentAnalysisTool(BaseTool):
    name: str = "Sentiment Analysis Tool"
    description: str = "Analyzes sentiment of text"
    
    def _run(self, text: str) -> str:
        return "positive"
```

**容错实战展示**: Agent 错误地试图用 FileReadTool 读取 URL -> crewAI 优雅失败 -> Agent 学习到不能这样做 -> 换用正确方式

---

## 第10课: 工具回顾

工具是连接 AI 应用(模糊世界)与外部系统(强类型世界)的桥梁。

**核心要点回顾**:
- 工具 = Agent 与外部世界的翻译层
- 三大品质: 多功能、容错、缓存
- 可使用 crewAI 内置工具 + LangChain 工具

---

## 第11课: 定义良好任务的关键要素

![任务核心要素](screenshots/11_task_elements.jpg)

**任务创建的两个必须**:
1. **清晰的描述** (Description): 你期望 Agent 做什么
2. **明确的预期输出** (Expected Output): 你期望得到什么结果

**生动比喻**: 就像给新来的实习生布置任务——你需要详细解释任务内容，并明确告诉他"交付物"长什么样。越清楚，结果越好。

**高级任务属性**:
| 属性 | 功能 |
|------|------|
| `context` | 指定依赖的前序任务输出 |
| `callback` | 任务完成后的回调函数 |
| `tools` | 覆盖 Agent 默认工具 |
| `human_input` | 执行前询问人类意见 |
| `async_execution` | 允许并行执行 |
| `output_json` | 输出为 JSON 格式 |
| `output_pydantic` | 输出为 Pydantic 对象 |
| `output_file` | 将结果写入文件 |

---

## 第12课: 实战 - 活动策划自动化

**三个 Agent**: Venue Coordinator + Logistics Manager + Marketing Agent

**新特性展示**:

**1. Pydantic 输出** - 将模糊输出转为强类型:
```python
class VenueDetails(BaseModel):
    name: str
    address: str
    capacity: int
    booking_status: str
```

**2. 并行执行**:
- 场地搜索任务先执行（串行）
- 物流任务 + 营销任务并行执行（`async_execution=True`）

**3. Human Input**:
- 物流任务设置 `human_input=True`
- Agent 完成前会暂停询问操作者意见

**4. 文件输出**:
- 场地信息 -> JSON 文件
- 营销策略 -> Markdown 文件

**生动比喻**: 就像策划婚礼——找场地必须先完成（不知道在哪办怎么安排餐饮？），但一旦场地确定，餐饮和请帖可以同时准备。

---

## 第13课: 任务回顾

**核心收获**:
- 心智框架帮助理解该创建什么 Agent 和 Task
- 描述和预期输出是任务的命脉
- 丰富的高级属性让系统更灵活
- 并行执行 + Pydantic 输出 = 生产级应用

---

## 第14-15课: 多 Agent 协作与金融分析

![协作流程](screenshots/14_collaboration_processes.jpg)

### 协作方式对比

| 方式 | 特点 | 适用场景 |
|------|------|---------|
| 顺序执行 (Sequential) | 任务依次传递，上下文逐渐衰减 | 流水线式工作 |
| 层级执行 (Hierarchical) | Manager Agent 统一调度 | 复杂决策场景 |
| 并行执行 (Parallel) | 独立任务同时运行 | 互不依赖的子任务 |

![层级流程](screenshots/14_hierarchical_process.jpg)

### 层级执行 (Hierarchical Process)

```python
from crewai import Process

crew = Crew(
    agents=[data_analyst, strategist, executor, risk_mgr],
    tasks=[...],
    process=Process.hierarchical,
    manager_llm=ChatOpenAI(model="gpt-4")
)
```

**Manager Agent 自动完成**:
- 记住初始目标
- 自动委派任务给合适的 Agent
- 审核结果，必要时要求改进

**金融分析实战**（4个 Agent）:
1. Data Analyst: 市场数据分析
2. Trading Strategy Developer: 制定交易策略
3. Execution Agent: 确定入场时机和价格
4. Risk Management Agent: 风险评估

**生动比喻**: 顺序执行像接力赛——一棒一棒传递；层级执行像交响乐团——指挥（Manager）协调各声部（Agent）何时演奏什么。

---

## 第16课: 实战 - 定制求职申请

**最复杂的 Crew**: 4 Agents + 4 Tasks + 并行 + 上下文依赖

**Agent 设计**:
1. Tech Job Researcher: 分析职位需求
2. Personal Profiler: 分析候选人技能
3. Resume Strategist: 重写简历
4. Interview Preparer: 准备面试材料

**执行流程**:
```
Research Task (async) ──┐
                        ├──> Resume Strategy Task ──> Interview Prep Task
Profile Task (async)  ──┘
```

**context 属性**: Resume Strategy Task 的 `context=[research_task, profile_task]` 确保等待两个并行任务都完成后再开始。

**输出文件**:
- `tailored_resume.md` - 针对职位定制的简历
- `interview_materials.md` - 面试准备材料

---

## 第17课: 下一步

**推荐资源**:
- crewAI 官方文档: docs.crewai.com
- crewAI Assistant (Custom GPT)
- crewAI+ 企业版（生产部署）
- YouTube 社区教程
- ChatDev 论文（Agent 协作理论基础）
- Discord 社区
- GitHub 开源代码

---

## 第18课: 总结

恭喜完成课程！你现在已经掌握了构建多 Agent 系统的完整知识体系，可以开始构建生产级的多智能体应用。

---

## 核心概念总结表

| 概念 | 定义 | 关键点 |
|------|------|--------|
| **Agent** | 具有角色、目标、背景的自治AI实体 | Role + Goal + Backstory |
| **Task** | Agent 需要执行的具体工作 | Description + Expected Output |
| **Crew** | Agent 和 Task 的组合体 | 定义协作方式和执行流程 |
| **Tools** | Agent 与外部世界交互的能力 | 多功能 + 容错 + 缓存 |
| **Process** | Agent 间的协作流程 | Sequential / Hierarchical / Parallel |
| **Memory** | Agent 的记忆能力 | 短期 + 长期 + 实体 |
| **Role Playing** | 让 Agent 扮演特定角色 | 精确关键词提升输出质量 |
| **Delegation** | Agent 间委托任务 | allow_delegation 控制 |
| **Guardrails** | 防止 Agent 偏离轨道 | 防幻觉、防循环、防超时 |
| **Kickoff** | 启动 Crew 执行 | 传入 inputs 字典插值变量 |

---

## 知识架构图

```
Multi AI Agent Systems with crewAI
|
├── 基础概念
|   ├── 什么是 AI Agent (LLM + 自我对话 + 工具)
|   ├── 什么是多 Agent 系统 (专家分工 + 协作)
|   ├── Agentic Automation vs 传统自动化
|   └── crewAI 框架 (开源 + 生产级)
|
├── 三大核心组件
|   ├── Agent (角色/目标/背景/委托/verbose)
|   ├── Task (描述/预期输出/工具/上下文/异步/输出格式)
|   └── Crew (Agent列表/Task列表/流程/记忆/verbose)
|
├── 六大关键要素
|   ├── 角色扮演 (Role Playing) — 专业关键词
|   ├── 聚焦 (Focus) — 单一职责
|   ├── 工具 (Tools) — 精选而非堆砌
|   ├── 合作 (Cooperation) — 委托与反馈
|   ├── 护栏 (Guardrails) — 防脱轨
|   └── 记忆 (Memory) — 短期/长期/实体
|
├── 工具系统
|   ├── 内置工具 (SerperDev/ScrapeWebsite/FileRead/DirectoryRead/MDXSearch)
|   ├── 自定义工具 (继承 BaseTool, 实现 _run)
|   ├── LangChain 工具兼容
|   └── 三大品质 (多功能/容错/缓存)
|
├── 协作流程 (Process)
|   ├── 顺序执行 (Sequential) — 默认模式
|   ├── 层级执行 (Hierarchical) — Manager 调度
|   └── 并行执行 (Async) — 独立任务并发
|
├── 高级特性
|   ├── 变量插值 (inputs dict -> {variable})
|   ├── Pydantic 输出 (模糊->强类型)
|   ├── 文件输出 (output_file)
|   ├── Human Input (人工审核)
|   ├── Context 依赖 (任务间数据流)
|   └── 跨 Agent 缓存
|
└── 实战项目
    ├── 研究与写作 Crew (Planner/Writer/Editor)
    ├── 客户支持自动化 (Support/QA + 工具 + 记忆)
    ├── 客户外展 Campaign (Sales/Lead + 自定义工具)
    ├── 活动策划 (Venue/Logistics/Marketing + 并行 + Pydantic)
    ├── 金融分析 (4 Agents + 层级流程)
    └── 简历定制 (4 Agents + 并行 + Context + 文件输出)
```

---

## 心智模型速查

**设计一个新 Crew 的步骤**:

1. 明确目标 -> "我要达成什么？"
2. 定义流程 -> "步骤是什么？"  
3. 确定角色 -> "我要雇谁？" -> 创建 Agent
4. 分配任务 -> "每人做什么？" -> 创建 Task
5. 选择工具 -> "他们需要什么工具？" -> 配置 Tools
6. 确定协作 -> "串行？并行？层级？" -> 设置 Process
7. 启动执行 -> `crew.kickoff(inputs={...})`

---

*笔记生成日期: 2026-05-20*
