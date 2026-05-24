# LangChain for LLM Application Development - 完整课程笔记

> 课程来源：DeepLearning.AI  
> 讲师：Andrew Ng & Harrison Chase（LangChain 创始人）  
> 框架：LangChain（Python / JavaScript 开源框架）

---

## 第1课 Introduction（课程介绍）

### 核心要点

- 通过 Prompt 驱动 LLM，现在可以比以往更快地开发 AI 应用
- 但实际应用往往需要**多次调用 LLM**并**解析输出**，存在大量"胶水代码"
- **LangChain** 由 Harrison Chase 创建，是一个开源框架，极大简化了 LLM 应用的开发流程

![LangChain 框架概述](screenshots/01_langchain_overview.jpg)

### LangChain 的两大核心价值

| 价值 | 说明 |
|------|------|
| 组合性与模块化 | 提供大量独立组件，可单独使用也可组合使用 |
| 端到端用例（Chains） | 将模块化组件串联为完整应用，快速上手 |

### 课程涵盖内容

![课程主题概览](screenshots/01_course_topics.jpg)

1. **Models（模型）**：语言模型的抽象封装
2. **Prompts（提示）**：引导模型完成有用任务的输入设计
3. **Indexes（索引）**：数据摄取，让外部数据与模型结合
4. **Chains（链）**：端到端的工作流串联
5. **Agents（代理）**：以模型作为推理引擎的高级用例

> **生动比喻**：如果把 LLM 比作一位超级翻译官，那么 LangChain 就是帮你搭建整个翻译公司的脚手架——从接单（Prompt）、记录客户偏好（Memory）、到多部门协作（Chains）、再到外派员工去查资料（Agents），一站式搞定。

---

## 第2课 Models, Prompts and Parsers（模型、提示与解析器）

### 核心概念

- **Models**：LLM 的封装（如 ChatOpenAI）
- **Prompts**：构造传入模型的输入
- **Parsers**：将模型的文本输出解析为结构化格式（如 Python 字典）

![Prompt Template 代码示例](screenshots/02_prompt_template.jpg)

### 为什么需要 Prompt Template？

直接用 f-string 格式化 Prompt 虽然简单，但当应用变复杂时：
- Prompt 可能非常长且精细
- 需要**复用**同一模板处理不同输入
- LangChain 还提供**内置模板**（如摘要、QA、SQL 连接等）

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

chat = ChatOpenAI(temperature=0.0)

template_string = "Translate the text delimited by triple backticks \
into a style that is {style}. text: ```{text}```"

prompt_template = ChatPromptTemplate.from_template(template_string)
# 自动识别变量：style, text
```

> **生动比喻**：Prompt Template 就像一份合同模板——你只需要填入客户名称和金额（变量），不需要每次都重新写整份合同。

### Output Parser（输出解析器）

![输出解析与 ReAct 框架](screenshots/02_output_parser.jpg)

**问题**：LLM 返回的是字符串，即使看起来像 JSON，也无法直接当字典用。

**解决方案**：使用 `ResponseSchema` + `StructuredOutputParser`

```python
from langchain.output_parsers import ResponseSchema, StructuredOutputParser

gift_schema = ResponseSchema(name="gift", description="Was the item a gift?")
delivery_schema = ResponseSchema(name="delivery_days", description="How many days to deliver?")
price_schema = ResponseSchema(name="price_value", description="Price value perception")

output_parser = StructuredOutputParser.from_response_schemas(
    [gift_schema, delivery_schema, price_schema]
)
# parser 会自动生成格式指令（format_instructions）
# 嵌入 Prompt 后，LLM 输出的结果可被解析为 Python 字典
```

### ReAct 框架简介

- **Thought**（思考）：给 LLM 空间思考，往往能得到更准确的结论
- **Action**（行动）：指定要执行的动作
- **Observation**（观察）：从动作中学到的信息

Prompt 指令 + Parser 解析 = 完美的输入输出抽象

---

## 第3课 Memory（记忆）

### 核心问题

LLM 本身是**无状态的**（stateless）——每次 API 调用都是独立的。聊天机器人看起来"有记忆"，只是因为代码把历史对话作为上下文传给了 LLM。

![ConversationBufferMemory 示例](screenshots/03_memory_buffer.jpg)

### LangChain 提供的记忆类型

![多种记忆类型](screenshots/03_memory_types.jpg)

| 记忆类型 | 工作方式 | 适用场景 |
|----------|----------|----------|
| **ConversationBufferMemory** | 保存完整对话历史 | 短对话，需要完整上下文 |
| **ConversationBufferWindowMemory** | 只保留最近 k 轮对话 | 控制上下文长度，节省 token |
| **ConversationTokenBufferMemory** | 按 token 数限制记忆 | 精确控制成本 |
| **ConversationSummaryBufferMemory** | 用 LLM 生成对话摘要 | 长对话，保留关键信息 |
| **Vector Data Memory** | 用向量数据库存储嵌入 | 大规模信息检索 |
| **Entity Memory** | 记住特定人物/实体的信息 | 需要记住用户画像 |

> **生动比喻**：
> - **BufferMemory** = 录音机：一字不漏全部录下来
> - **WindowMemory** = 只看最后几页笔记的学生：之前的全忘了
> - **TokenBufferMemory** = 有字数限制的笔记本：写满了就从头开始覆盖
> - **SummaryMemory** = 秘书帮你写的会议纪要：细节丢了但大意都在

### 代码示例

```python
from langchain.memory import ConversationBufferMemory, ConversationSummaryBufferMemory

# 方式1：完整缓冲
memory = ConversationBufferMemory()

# 方式2：窗口缓冲（只记最近1轮）
memory = ConversationBufferWindowMemory(k=1)

# 方式3：摘要缓冲（超过token限制就生成摘要）
memory = ConversationSummaryBufferMemory(llm=llm, max_token_limit=100)
```

### 关键洞察

- 对话越长，token 消耗越大，成本越高
- 选择合适的记忆策略是**成本与质量的平衡**
- 实际生产中，常将完整对话额外存入数据库（用于审计和改进系统）

---

## 第4课 Chains（链）

### 核心概念

**Chain = LLM + Prompt**，是 LangChain 最重要的构建块。可以把多个 Chain 串联起来，对文本或数据执行一系列操作。

### Chain 类型一览

#### 1. LLMChain（基础链）

最简单但最强大的链，将一个 Prompt 和一个 LLM 组合在一起。

```python
from langchain.chat_models import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain.chains import LLMChain

llm = ChatOpenAI(temperature=0.9)
prompt = ChatPromptTemplate.from_template(
    "What is the best name for a company that makes {product}?"
)
chain = LLMChain(llm=llm, prompt=prompt)
chain.run("queen size sheet set")  # 输出: "Royal Beddings"
```

#### 2. SimpleSequentialChain（简单顺序链）

![简单顺序链示意图](screenshots/04_simple_sequential_chain.jpg)

- 每个子链只有**一个输入**和**一个输出**
- 前一个链的输出自动作为下一个链的输入
- 像流水线一样顺序执行

#### 3. SequentialChain（顺序链）

![顺序链示意图](screenshots/04_sequential_chain.jpg)

- 支持**多个输入**和**多个输出**
- 每一步可以接收来自之前任何步骤的变量
- 关键：**input_key 和 output_key 必须精确对齐**

```python
# 链1：翻译评论为英文 (input: review → output: english_review)
# 链2：生成一句话摘要 (input: english_review → output: summary)
# 链3：检测原始语言 (input: review → output: language)
# 链4：用原始语言写回复 (input: summary + language → output: followup_message)
```

#### 4. Router Chain（路由链）

![路由链概念](screenshots/04_router_chain.jpg)

- 根据输入内容**动态路由**到不同的子链
- 使用 LLM 自身来决定应该走哪条路径
- 需要为每个子链提供**名称**和**描述**（供路由器判断）
- 还需要一个 **Default Chain** 兜底

> **生动比喻**：
> - **LLMChain** = 一个员工完成一项任务
> - **SimpleSequentialChain** = 流水线：A做完交给B，B做完交给C
> - **SequentialChain** = 项目组：每个人可以参考多人的工作成果
> - **Router Chain** = 前台接待：根据你的问题把你引导到不同部门

---

## 第5课 Question and Answer（文档问答）

### 核心问题

LLM 的上下文窗口有限（只能处理几千个词），如何让它回答关于大量文档的问题？

### 解决方案：Embeddings + Vector Store

![Embeddings 概念](screenshots/05_embeddings.jpg)

**Embeddings（嵌入）**：将文本转化为数值向量表示
- 捕获文本的**语义含义**
- 内容相似的文本 → 向量相似
- 例如："我的猫" 和 "我的狗" 的向量接近，但和 "我的车" 的向量差异大

![向量存储工作流](screenshots/05_vector_store.jpg)

**Vector Store（向量存储）**工作流程：

1. **文档切分**：将大文档分成小块（chunks）
2. **向量化**：为每个 chunk 生成 embedding
3. **存储**：将向量存入向量数据库
4. **查询**：用户提问时，将问题也转成向量
5. **检索**：找到最相似的 n 个 chunk
6. **生成**：将检索到的内容和问题一起传给 LLM，生成最终答案

> **生动比喻**：想象你有一整个图书馆的书。你不可能把所有书都搬到桌上让教授看。所以你先给每本书写一个摘要标签（embedding），放在索引卡片盒里（vector store）。当有人问问题时，你先翻索引卡找到最相关的几本书（retrieval），再把这几本书拿给教授看（LLM），教授就能给出准确答案。

### 代码实现

```python
from langchain.document_loaders import CSVLoader
from langchain.vectorstores import DocArrayInMemorySearch
from langchain.indexes import VectorstoreIndexCreator

# 一行代码创建索引
loader = CSVLoader(file_path="outdoor_clothing.csv")
index = VectorstoreIndexCreator(
    vectorstore_cls=DocArrayInMemorySearch
).from_loaders([loader])

# 查询
response = index.query("Please list all shirts with sun protection")
```

### 四种文档问答方法

![QA 方法对比](screenshots/05_qa_methods.jpg)

| 方法 | 工作方式 | 优点 | 缺点 |
|------|----------|------|------|
| **Stuff** | 把所有文档塞入一个 prompt | 简单、便宜、效果好 | 文档太多时超出上下文限制 |
| **Map_Reduce** | 每个文档单独问答，再汇总 | 可处理任意数量文档，可并行 | 调用多、文档间无关联 |
| **Refine** | 逐个文档迭代优化答案 | 善于整合信息 | 慢、不能并行 |
| **Map_Rerank** | 每个文档打分，选最高分 | 可并行、快 | 依赖LLM打分准确性 |

---

## 第6课 Evaluation（评估）

### 核心挑战

LLM 应用的评估非常困难，因为：
- 输出是开放式的自然语言文本
- 没有唯一正确答案
- 传统的字符串匹配完全不适用

### 评估三步走

#### Step 1：创建测试数据集

**方法A - 手动创建**：查看文档，手写问题和期望答案

```python
examples = [
    {"query": "Does the Cozy Comfort Pullover Set have side pockets?", "answer": "Yes"},
    {"query": "What collection is this jacket from?", "answer": "The DownTek collection"}
]
```

**方法B - LLM 自动生成**：用 `QAGenerateChain` 从文档自动生成问答对

```python
from langchain.evaluation.qa import QAGenerateChain
example_gen_chain = QAGenerateChain.from_llm(ChatOpenAI())
new_examples = example_gen_chain.apply_and_parse([{"doc": doc} for doc in documents])
```

#### Step 2：调试（Debug）

![LangChain Debug 模式](screenshots/06_debug_mode.jpg)

```python
import langchain
langchain.debug = True  # 打印每个步骤的详细输入输出
```

Debug 模式可以让你看到：
- 每一步的输入和输出
- 检索到了哪些文档
- 实际发送给 LLM 的完整 prompt
- Token 使用量和模型信息

> **关键洞察**：当答案错误时，往往不是 LLM 本身的问题，而是**检索步骤**出了问题——传给 LLM 的文档不相关。

#### Step 3：自动评估

**用 LLM 来评估 LLM！**

```python
from langchain.evaluation.qa import QAEvalChain
eval_chain = QAEvalChain.from_llm(llm)
graded_outputs = eval_chain.evaluate(examples, predictions)
```

为什么必须用 LLM 做评估？
- "Yes" 和 "The Cozy Comfort Pullover Set does have side pockets" 语义相同
- 但字符串完全不同，正则/精确匹配无法判断
- LLM 能理解语义相似性

### LangChain 评估平台

![LangChain 评估平台 UI](screenshots/06_eval_platform.jpg)

- 可视化每次运行的输入输出
- 可以逐层深入查看链的每个步骤
- 方便积累测试数据集（flywheel 飞轮效应）

---

## 第7课 Agents（代理）

### 核心理念

LLM 不仅是知识库，更应该被看作**推理引擎**（Reasoning Engine）：
- 接收信息 → 思考推理 → 决定下一步行动 → 执行 → 观察结果 → 继续推理

![Agent 初始化与工具配置](screenshots/07_agent_tools.jpg)

### Agent 的工作方式

```python
from langchain.agents import load_tools, initialize_agent, AgentType
from langchain.chat_models import ChatOpenAI

llm = ChatOpenAI(temperature=0)
tools = load_tools(["ddg-search", "wikipedia"], llm=llm)

agent = initialize_agent(
    tools, llm,
    agent=AgentType.CHAT_ZERO_SHOT_REACT_DESCRIPTION,
    handle_parsing_errors=True
)
```

**CHAT_ZERO_SHOT_REACT_DESCRIPTION** 的含义：
- **Chat**：针对聊天模型优化
- **Zero-shot**：不需要示例就能使用工具
- **ReAct**：使用 ReAct 提示策略（思考-行动-观察循环）

### Agent 推理过程示例

![Agent 推理过程](screenshots/07_agent_reasoning.jpg)

```
用户: "谁赢了2022年世界杯？"

Agent 思考: 我的训练数据截止到2021年，需要搜索最新信息
Agent 行动: 使用 DuckDuckGo 搜索 "2022 World Cup winner"
Agent 观察: 搜索结果显示阿根廷相关信息...
Agent 思考: 需要更多确认信息
Agent 行动: 再次搜索确认
Agent 最终回答: "阿根廷赢得了2022年世界杯"
```

### 自定义工具（Custom Tools）

```python
from langchain.agents import tool
from datetime import date

@tool
def time(text: str) -> str:
    """Returns today's date. Use this for questions about the current date.
    The input should always be an empty string."""
    return str(date.today())

# 将自定义工具加入 agent
tools = load_tools(["ddg-search", "wikipedia"], llm=llm)
tools.append(time)
agent = initialize_agent(tools, llm, agent=AgentType.CHAT_ZERO_SHOT_REACT_DESCRIPTION)
```

**关键**：函数的 **docstring** 非常重要！Agent 根据它来决定何时、如何调用该工具。

> **生动比喻**：Agent 就像一位侦探。他有自己的推理能力（LLM），还有各种调查工具（搜索引擎、数据库、API）。面对一个案件（用户问题），他会思考需要什么线索，选择合适的工具去调查，拿到结果后继续推理，直到破案。

---

## 第8课 Conclusion（总结）

### 课程回顾

在这门短课程中，你学会了：
- 处理客户评论（翻译、提取信息、格式化输出）
- 构建文档问答系统（Embeddings + Vector Store + RetrievalQA）
- 使用 LLM 决定何时调用外部工具（Agents）

### 关键感悟

> "如果两周前有人问你，构建这些应用需要多少工作量？很多人会觉得需要数周甚至更长时间。但你看到了，用 LangChain 只需要相当少的代码就能高效实现。"

### LangChain 的更多可能

- 查询 CSV 文件
- 查询 SQL 数据库
- 与各种 API 交互
- 各种 Chain 的组合使用
- 社区贡献的丰富功能

---

## 核心概念总结表

| 概念 | 中文名 | 一句话说明 |
|------|--------|-----------|
| Model | 模型 | LLM 的封装，如 ChatOpenAI |
| Prompt Template | 提示模板 | 可复用的 Prompt 模板，含变量占位符 |
| Output Parser | 输出解析器 | 将 LLM 文本输出转为结构化数据 |
| Memory | 记忆 | 让无状态的 LLM 拥有对话上下文 |
| Chain | 链 | LLM + Prompt 的组合，可串联执行 |
| Sequential Chain | 顺序链 | 多个 Chain 按顺序执行 |
| Router Chain | 路由链 | 根据输入动态选择执行路径 |
| Embedding | 嵌入 | 文本的数值向量表示，捕获语义 |
| Vector Store | 向量存储 | 存储和检索文本嵌入的数据库 |
| Retriever | 检索器 | 根据查询找到相关文档的接口 |
| Agent | 代理 | 以 LLM 为推理引擎，自主决定使用工具 |
| Tool | 工具 | Agent 可以调用的外部功能/API |
| ReAct | 思考-行动 | 让 LLM 按"思考→行动→观察"循环推理 |

---

## 知识架构图

```
LangChain for LLM Application Development
│
├── 基础组件层
│   ├── Models（模型封装）
│   │   └── ChatOpenAI, temperature 参数
│   ├── Prompts（提示工程）
│   │   ├── ChatPromptTemplate
│   │   ├── 变量占位符 {style}, {text}
│   │   └── 内置模板（摘要、QA、SQL...）
│   └── Output Parsers（输出解析）
│       ├── ResponseSchema
│       └── StructuredOutputParser → Python Dict
│
├── 记忆层 Memory
│   ├── ConversationBufferMemory（完整缓冲）
│   ├── ConversationBufferWindowMemory（窗口缓冲）
│   ├── ConversationTokenBufferMemory（Token限制）
│   ├── ConversationSummaryBufferMemory（摘要缓冲）
│   ├── Vector Data Memory（向量记忆）
│   └── Entity Memory（实体记忆）
│
├── 链式执行层 Chains
│   ├── LLMChain（基础链）
│   ├── SimpleSequentialChain（简单顺序链）
│   ├── SequentialChain（多输入输出顺序链）
│   └── Router Chain（路由链）
│       ├── MultiPromptChain
│       ├── LLMRouterChain
│       └── Default Chain
│
├── 数据连接层 Indexes
│   ├── Document Loaders（文档加载器）
│   │   └── CSVLoader, PDFLoader...
│   ├── Text Splitters（文本分割器）
│   ├── Embeddings（嵌入模型）
│   │   └── OpenAIEmbeddings
│   ├── Vector Stores（向量存储）
│   │   └── DocArrayInMemorySearch, Chroma...
│   └── Retrievers（检索器）
│       └── similarity_search
│
├── 高级应用层
│   ├── RetrievalQA（检索问答）
│   │   ├── stuff（填充法）
│   │   ├── map_reduce（映射归约法）
│   │   ├── refine（迭代优化法）
│   │   └── map_rerank（映射重排法）
│   └── Agents（代理）
│       ├── 内置工具：DuckDuckGo, Wikipedia
│       ├── 自定义工具：@tool 装饰器
│       └── AgentType: CHAT_ZERO_SHOT_REACT_DESCRIPTION
│
└── 评估与调试层
    ├── langchain.debug = True
    ├── QAGenerateChain（自动生成测试集）
    ├── QAEvalChain（LLM 评估 LLM）
    └── LangChain Evaluation Platform（可视化平台）
```

---

## 学习建议

1. **动手实践**：每节课都有 Jupyter Notebook，务必亲自运行代码
2. **从简单开始**：先掌握 LLMChain，再学 SequentialChain，最后学 Agent
3. **关注 Prompt 设计**：好的 Prompt Template 是复用和协作的关键
4. **选对 Memory**：根据应用场景选择合适的记忆策略，平衡成本与效果
5. **重视评估**：建立评估飞轮，持续改进应用质量
6. **探索社区**：LangChain 生态极其活跃，社区有大量现成的 Chain 和 Tool

---

*笔记整理完毕。LangChain 的核心思想是"模块化 + 可组合"——每个组件独立可用，组合起来威力倍增。掌握了这些构建块，你就能快速搭建各种 LLM 应用。*
