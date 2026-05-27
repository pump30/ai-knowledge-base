# Functions, Tools and Agents with LangChain - 学习笔记

> **课程来源**: DeepLearning.AI  
> **讲师**: Harrison Chase (LangChain 联合创始人兼CEO) & Andrew Ng  
> **核心主题**: OpenAI Function Calling、LangChain 表达式语言 (LCEL)、工具使用与智能代理构建

---

## 第一课: 课程介绍 (Introduction)

### 核心要点

LLM 最初被设计为为人类生成文本，但现在一些 LLM 已经被训练为可以输出**格式化数据**（如 JSON），从而使 LLM 能够决定何时调用其他代码作为子程序。

**课程涵盖两大变化**:
1. **LangChain 表达式语言 (LCEL)** - 一种全新的语法，使构建链和序列更加简单透明
2. **OpenAI Function Calling** - 利用函数调用能力实现标签、数据提取、工具使用和对话代理

> **生动比喻**: 想象 LLM 是一个非常聪明但被关在房间里的人。Function Calling 就像给了他一部电话和一本通讯录 -- 他可以决定什么时候打电话给谁（调用哪个函数），告诉对方需要什么信息（传递参数），然后利用得到的回答来帮助你。

---

## 第二课: OpenAI Function Calling

### 核心概念

OpenAI 对最新模型进行了微调，使其能够接受一个额外的 `functions` 参数，模型会判断是否需要调用某个函数。

#### 函数定义结构 (JSON Schema)

```json
{
  "name": "get_current_weather",
  "description": "获取指定位置的当前天气",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {"type": "string", "description": "城市名称"},
      "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
    },
    "required": ["location"]
  }
}
```

#### 三种调用模式 (function_call 参数)

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `"auto"` (默认) | 模型自行决定是否调用函数 | 通用场景 |
| `"none"` | 强制不调用任何函数 | 只需要文本回复时 |
| `{"name": "函数名"}` | 强制调用指定函数 | 数据提取、标签等场景 |

#### 关键要点

1. **OpenAI 不会直接执行函数** -- 它只告诉你调用哪个函数以及用什么参数，执行需要你自己完成
2. **函数描述占用 token** -- 函数定义和描述会计入 token 使用限额
3. **可以将结果回传** -- 通过 `role: "function"` 的消息将函数执行结果传回模型，获得自然语言最终回答

> **生动比喻**: Function Calling 就像你问秘书"波士顿今天天气怎么样？"，秘书不会自己去看天气，而是写了一张便条告诉你："请查询 get_current_weather 函数，参数是 location='Boston'"。你查完后把结果告诉秘书，秘书再用自然语言告诉你："波士顿今天72华氏度，晴天有风。"

---

## 第三课: LangChain 表达式语言 (LCEL)

### 核心概念

LCEL 是 LangChain 中的新语法，通过 **Runnable 协议** 定义了统一的组件接口。

#### Runnable 接口的标准方法

| 方法 | 功能 | 说明 |
|------|------|------|
| `invoke` | 单输入调用 | 同步执行 |
| `stream` | 流式输出 | 逐步返回结果 |
| `batch` | 批量调用 | 传入列表，自动并行 |
| `ainvoke/astream/abatch` | 异步版本 | 对应的异步方法 |

#### LCEL 四大优势

1. **开箱即用的异步/批量/流式支持** -- 无需额外编码
2. **Fallback 回退机制** -- 可为单个组件或整个链附加备用方案
3. **自动并行化** -- LLM 调用耗时长，LCEL 自动并行执行
4. **内置日志记录** -- 所有步骤的输入输出自动记录，支持 LangSmith

#### 链的组合语法

```python
# 使用 pipe (|) 操作符组合组件
chain = prompt | model | output_parser

# 调用链
result = chain.invoke({"topic": "bears"})
```

#### 高级功能

- **RunnableMap**: 并行执行多个操作，将结果组合为字典
- **bind()**: 为 Runnable 绑定固定参数（如绑定 functions 到模型）
- **with_fallbacks()**: 当主链失败时自动切换到备用链

> **生动比喻**: LCEL 就像乐高积木系统。每块积木（Runnable）都有标准的接口（凸起和凹槽），你可以把 Prompt 积木、Model 积木、Parser 积木自由拼接。无论拼成什么形状，每块积木都能正常工作，因为接口是统一的。

---

## 第四课: 在 LangChain 中使用 OpenAI Function Calling

### 核心概念

用 **Pydantic** 简化 OpenAI 函数定义的创建过程，再与 LCEL 结合使用。

#### Pydantic 是什么？

Pydantic 是 Python 数据验证库，有以下优势:
- **类型验证**: 自动检查数据类型是否正确
- **Schema 导出**: 轻松导出为 JSON Schema
- **嵌套支持**: 支持复杂的嵌套数据结构

#### 从 Pydantic 到 OpenAI Function

```python
from langchain.utils.openai_functions import convert_pydantic_to_openai_function

class WeatherSearch(BaseModel):
    """搜索指定机场的天气"""  # docstring -> 函数描述
    airport_code: str = Field(description="机场代码")  # Field -> 参数描述

# 一键转换
weather_function = convert_pydantic_to_openai_function(WeatherSearch)
```

**映射关系**:
- 类名 -> 函数 name
- docstring -> 函数 description (必须提供!)
- 属性名 -> 参数 properties
- Field(description=...) -> 参数描述
- 类型标注 -> 参数类型

#### 与 LCEL 结合

```python
# 绑定函数到模型
model_with_function = model.bind(functions=[weather_function])

# 构建完整链
chain = prompt | model_with_function
```

可以绑定多个函数，让模型根据输入自动选择合适的函数调用。

> **生动比喻**: 手写 JSON Schema 就像用汇编语言编程 -- 繁琐且容易出错。Pydantic 就像高级编程语言，你只需要写清楚"我要什么类型的数据"，编译器（convert_pydantic_to_openai_function）会自动帮你翻译成 JSON Schema 这种"机器语言"。

---

## 第五课: 标签与数据提取 (Tagging and Extraction)

### 核心概念

利用 OpenAI Function Calling 从非结构化文本中提取结构化数据。

#### 标签 (Tagging) vs 提取 (Extraction)

| 维度 | 标签 (Tagging) | 提取 (Extraction) |
|------|---------------|-------------------|
| 目的 | 对文本整体进行分类/标注 | 从文本中提取具体实体信息 |
| 输出 | 单个结构化对象 | 实体列表 |
| 示例 | 情感分析、语言识别 | 提取人名、论文标题 |

#### 标签示例

```python
class Tagging(BaseModel):
    """标记文本的特定信息"""
    sentiment: str = Field(description="情感：pos/neg/neutral")
    language: str = Field(description="语言，ISO 639-1 代码")
```

#### 提取示例

```python
class Person(BaseModel):
    """一个人的信息"""
    name: str = Field(description="人名")
    age: Optional[int] = Field(description="年龄")

class Information(BaseModel):
    """要提取的信息"""
    people: List[Person]  # 提取多个人的信息
```

#### 关键技巧

1. **强制调用函数**: 使用 `function_call={"name": "函数名"}` 确保模型总是返回结构化数据
2. **Prompt Engineering**: 在系统消息中明确指示 "如果信息未提供，不要猜测"
3. **输出解析器**:
   - `JsonOutputFunctionsParser` -- 解析出 JSON 结果
   - `JsonKeyOutputFunctionsParser` -- 只提取特定 key 的值

#### 处理长文本

对于超过 token 限制的文章:
1. 使用 `RecursiveCharacterTextSplitter` 分割文本
2. 对每个片段并行执行提取链 (`.map()`)
3. 使用 `flatten` 函数合并所有结果

> **生动比喻**: 标签就像给一封信贴邮票 -- 你看完整封信后，给它贴一个"紧急"或"普通"的标签。提取就像从一大堆信件中找出所有提到的人名和地址 -- 你需要逐字逐句地扫描，把相关信息一条条抽出来。

---

## 第六课: 工具与路由 (Tools and Routing)

### 核心概念

LangChain 中"工具"(Tool) 包含两个核心组件:
1. **函数描述** -- 告诉 LLM 工具的用途和输入格式
2. **实际可执行的函数** -- 真正执行操作的代码

#### 创建自定义工具

```python
from langchain.agents import tool

@tool
def get_current_temperature(latitude: float, longitude: float) -> str:
    """获取指定经纬度的当前温度"""
    # 调用 OpenMeteo API
    ...
    return f"当前温度为 {temp} 摄氏度"
```

使用 Pydantic 定义更精确的输入 Schema:
```python
class OpenMeteoInput(BaseModel):
    latitude: float = Field(description="纬度")
    longitude: float = Field(description="经度")

@tool(args_schema=OpenMeteoInput)
def get_current_temperature(latitude: float, longitude: float) -> str:
    """获取指定经纬度的当前温度"""
    ...
```

#### OpenAPI Spec 转换

可以直接将 OpenAPI 规范文件转换为 LangChain 工具:
```python
from langchain.chains.openai_functions import openapi_spec_to_openai_fn
functions, callables = openapi_spec_to_openai_fn(spec)
```

#### 路由 (Routing) 机制

路由是指使用 LLM 来决定执行哪条路径:

```python
from langchain.agents.output_parsers import OpenAIFunctionsAgentOutputParser

chain = prompt | model | OpenAIFunctionsAgentOutputParser()
result = chain.invoke({"input": "SF 现在天气怎么样？"})

# result 可能是:
# - AgentAction: 需要调用工具 (result.tool, result.tool_input)
# - AgentFinish: 直接回复 (result.return_values)
```

#### 完整路由函数

```python
def route(result):
    if isinstance(result, AgentFinish):
        return result.return_values["output"]
    else:
        tool = {"search_wikipedia": search_wikipedia, 
                "get_current_temperature": get_current_temperature}[result.tool]
        return tool.run(result.tool_input)

chain = prompt | model | OpenAIFunctionsAgentOutputParser() | route
```

> **生动比喻**: 路由就像医院的导诊台。病人（用户输入）来了之后，导诊护士（LLM）根据症状描述决定你应该去哪个科室（选择哪个工具）。如果你只是来问路（简单问题），护士直接回答你（AgentFinish）；如果你需要看病（需要工具），护士会告诉你去几楼几号诊室（AgentAction）。

---

## 第七课: 对话代理 (Conversational Agent)

### 核心概念

将工具使用与**聊天记忆**结合，构建类似 ChatGPT 的对话系统。

#### 什么是 Agent？

Agent = 语言模型 + 代码执行循环

```
用户输入 -> LLM 推理 -> 选择工具 -> 执行工具 -> 观察结果 
    ^                                              |
    |______________ 继续循环直到满足停止条件 _________|
```

**停止条件**:
- LLM 决定不再调用工具（AgentFinish）
- 达到最大迭代次数
- 其他硬编码规则

#### Agent Scratchpad (代理草稿本)

代理需要一个地方来记录"中间步骤"（已调用的工具和结果）:

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有用的助手"),
    MessagesPlaceholder(variable_name="chat_history"),  # 对话历史
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),  # 中间步骤
])
```

#### 自定义 Agent 循环

```python
def run_agent(user_input):
    intermediate_steps = []
    while True:
        result = agent_chain.invoke({
            "input": user_input,
            "intermediate_steps": intermediate_steps
        })
        if isinstance(result, AgentFinish):
            return result.return_values["output"]
        # 执行工具
        tool = tool_map[result.tool]
        observation = tool.run(result.tool_input)
        intermediate_steps.append((result, observation))
```

#### AgentExecutor (LangChain 内置)

```python
from langchain.agents import AgentExecutor

agent_executor = AgentExecutor(
    agent=agent_chain, 
    tools=tools, 
    verbose=True,
    memory=memory  # 添加记忆
)
```

AgentExecutor 相比自定义循环的优势:
- **错误处理**: LLM 输出无效 JSON 时自动重试
- **工具错误处理**: 工具执行失败时传回错误让 LLM 修正
- **内置超时和最大迭代限制**

#### 添加对话记忆

```python
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(
    return_messages=True,  # 以消息列表形式返回
    memory_key="chat_history"  # 对应 prompt 中的变量名
)

agent_executor = AgentExecutor(
    agent=agent_chain, tools=tools, 
    verbose=True, memory=memory
)
```

> **生动比喻**: Agent 就像一个项目经理。他有一个待办清单（scratchpad），接到任务后，他会思考"我需要先查天气数据，再查维基百科"，然后一步步执行，每完成一步就在清单上记录结果。等所有必要信息都收集完毕，他才会给出最终汇报。记忆（Memory）就像项目经理的笔记本，记录了之前所有的对话，这样他就不会忘记你之前说过什么。

---

## 第八课: 课程总结 (Conclusion)

本课程涵盖了两大核心进展:
1. **OpenAI Function Calling** -- LLM 输出结构化数据的能力
2. **LangChain Expression Language (LCEL)** -- 组合 LLM 组件的新语法

应用方向:
- 结构化数据提取（Tagging & Extraction）
- 工具使用与工具选择（Tool Usage & Selection）
- 对话代理（Conversational Agent）

---

## 核心概念总结表

| 概念 | 定义 | 核心作用 |
|------|------|----------|
| Function Calling | OpenAI 模型输出函数调用建议的能力 | 让 LLM 与外部系统交互 |
| LCEL | LangChain 表达式语言 | 用 pipe 语法组合 AI 组件 |
| Pydantic | Python 数据验证库 | 简化函数定义的创建 |
| Runnable | LCEL 中的标准组件接口 | 统一的 invoke/stream/batch |
| Tagging | 对文本整体打标签 | 情感分析、分类 |
| Extraction | 从文本提取结构化实体 | 信息抽取、知识提取 |
| Tool | 包含描述和可执行代码的工具 | 让 Agent 能执行实际操作 |
| Routing | LLM 决定调用哪个工具 | 智能路径选择 |
| Agent | LLM + 循环执行逻辑 | 自主推理和行动 |
| AgentExecutor | LangChain 的 Agent 运行器 | 错误处理、记忆管理 |
| Memory | 对话历史存储 | 多轮对话上下文保持 |
| Agent Scratchpad | 代理中间步骤记录 | 工具调用历史追踪 |

---

## 知识架构图

```
Functions, Tools and Agents with LangChain
|
|-- [基础能力]
|   |-- OpenAI Function Calling
|   |   |-- 函数定义 (JSON Schema)
|   |   |-- 调用模式 (auto/none/force)
|   |   |-- 结果回传 (role: function)
|   |   +-- Token 消耗注意事项
|   |
|   +-- LangChain Expression Language (LCEL)
|       |-- Runnable 协议 (invoke/stream/batch)
|       |-- Pipe 操作符 (|) 组合链
|       |-- RunnableMap (并行执行)
|       |-- bind() (绑定参数)
|       +-- with_fallbacks() (回退机制)
|
|-- [工具层]
|   |-- Pydantic 数据建模
|   |   |-- BaseModel 类定义
|   |   |-- Field 描述
|   |   +-- convert_pydantic_to_openai_function()
|   |
|   |-- Tool 创建
|   |   |-- @tool 装饰器
|   |   |-- args_schema 参数
|   |   +-- format_tool_to_openai_function()
|   |
|   +-- OpenAPI Spec 转换
|       +-- openapi_spec_to_openai_fn()
|
|-- [应用层]
|   |-- Tagging (标签)
|   |   |-- 情感分析
|   |   |-- 语言识别
|   |   +-- 关键词提取
|   |
|   |-- Extraction (提取)
|   |   |-- 实体提取
|   |   |-- 长文本分割 + 并行提取
|   |   +-- 结果合并 (flatten)
|   |
|   +-- Routing (路由)
|       |-- AgentAction (需要工具)
|       |-- AgentFinish (直接回复)
|       +-- route() 函数分发
|
+-- [代理层]
    |-- Agent 循环
    |   |-- 推理 -> 行动 -> 观察 -> 重复
    |   |-- Agent Scratchpad
    |   +-- 停止条件
    |
    |-- AgentExecutor
    |   |-- 错误处理
    |   |-- 超时控制
    |   +-- 最大迭代限制
    |
    +-- Conversational Agent
        |-- Chat History (对话历史)
        |-- ConversationBufferMemory
        +-- Panel UI 展示
```

---

## 实践建议

1. **从简单开始**: 先掌握单个函数调用，再逐步构建复杂链
2. **重视描述质量**: 函数描述和参数描述就是给 LLM 的 prompt，写得越清晰效果越好
3. **善用 Pydantic**: 比手写 JSON Schema 更安全、更易维护
4. **注意 Token 预算**: 函数定义会占用 token，函数越多越要注意总量
5. **增量式 Prompt 优化**: 从简单 prompt 开始，根据错误结果逐步添加约束条件
6. **错误处理不可少**: 生产环境务必使用 AgentExecutor 而非自定义简单循环
