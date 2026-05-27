# AI Agent 评估 (Evaluating AI Agents)

> **课程来源**: DeepLearning.AI x Arize AI  
> **讲师**: John Gilhuly (Developer Relations Lead), Aman Khan (Director of Product) @ Arize AI  
> **工具**: Arize Phoenix, OpenTelemetry, OpenAI API  

---

## 第1课: 课程介绍 (Introduction)

### 核心要点

本课程教你如何通过**评估驱动开发 (Evaluation-Driven Development)** 来系统化地改进 AI Agent。课程围绕一个**数据分析 Agent** 展开,该 Agent 可以连接数据库、执行分析并生成可视化图表。

**你将学到的核心能力:**
- 为 Agent 添加**可观测性 (Observability)**,看到每一步的执行过程
- 对 Agent 的各个组件进行**逐一评估**
- 评估 Agent 的**执行路径 (Trajectory)** 是否高效
- 将评估结构化为**实验 (Experiments)** 以系统化迭代

> **生动比喻**: 评估驱动开发就像给赛车装上仪表盘 -- 你不再是"蒙眼开车随意调整",而是看着速度表、油温表、胎压表来精确调校每一个部件。

---

## 第2课: LLM 时代的评估 (Evaluation in the Time of LLMs)

### 核心要点

#### 两层评估体系

| 评估层次 | 关注点 | 典型基准 |
|---------|--------|---------|
| **模型评估 (Model Eval)** | LLM 本身在特定任务上的表现 | MMLU, HumanEval |
| **系统评估 (System Eval)** | 整个应用(含 prompt、工具、记忆)的表现 | 自定义数据集 |

#### 传统软件测试 vs LLM 系统测试

> **生动比喻**: 传统软件测试像**火车在轨道上运行** -- 起点终点明确,检查轨道和车厢是否正常即可。LLM 系统测试像**在繁忙城市里开车** -- 环境多变,系统非确定性,同一个输入可能得到不同输出。

#### Agent 带来的额外复杂性

Agent = **推理 (Reasoning)** + **路由 (Routing)** + **行动 (Action)**

**常见评估维度:**
- 幻觉检测 (Hallucination)
- 检索相关性 (Retrieval Relevance)
- 问答准确性 (QA Accuracy)
- 毒性检测 (Toxicity)
- 整体性能 (Overall Performance)

**关键洞察**: 即使微小的 prompt 改动也可能产生意想不到的**涟漪效应**,因此需要维护一组代表性测试用例,每次调整后重新运行评估以捕捉回归。

---

## 第3课: 分解 Agent 结构 (Decomposing Agents)

### 核心要点

#### Agent 的三大组件

1. **路由器 (Router)** -- Agent 的"大脑",决定调用哪个技能
   - 可以是 LLM + Function Calling(灵活但不稳定)
   - 也可以是 NLP 分类器或规则代码(稳定但能力有限)
   
2. **技能 (Skills)** -- Agent 的"四肢",执行具体任务
   - 每个技能由多个步骤组成(LLM调用、API调用、代码执行)
   - 例: RAG 技能 = 嵌入 + 向量检索 + LLM生成

3. **记忆与状态 (Memory & State)** -- Agent 的"记忆",存储上下文
   - 检索到的上下文
   - 配置变量
   - 历史执行步骤日志

#### 课程示例 Agent: 数据分析助手

该 Agent 包含三个工具:
- **Lookup Sales Data**: 生成SQL -> 执行SQL -> 返回结果
- **Data Analysis**: 单次LLM调用进行数据分析
- **Data Visualization**: 生成图表配置 -> 生成Python代码 (两步法)

> **生动比喻**: 数据可视化工具采用**两步法**,就像你让实习生先画草稿再正式作图 -- 先确定图表类型/坐标轴/标题,再生成代码,比一步到位更可靠。

---

## 第4课: 实验1 - 构建你的 Agent (Lab 1)

### 核心要点

本实验使用 Python 从零构建数据分析 Agent:

**技术栈**: OpenAI API + Pandas + DuckDB + Pydantic

**关键实现细节:**
1. **SQL 生成工具**: Prompt 模板 + GPT-4o-mini 生成 SQL,DuckDB 执行
2. **数据分析工具**: 简单的 LLM 调用,传入数据和问题
3. **可视化工具**: 使用 Pydantic + Structured Output 确保输出格式正确
4. **路由器**: While 循环 + OpenAI Function Calling,遍历工具调用直到完成
5. **工具描述**: JSON 格式定义每个工具的名称、描述和参数 -- 这是路由正确性的关键

> **实践经验**: 工具描述写得好不好,直接决定路由器能否正确选择工具。这是构建 Agent 时最需要反复调试的部分。

---

## 第5课: 追踪 Agent (Tracing Agents)

### 核心要点

#### 可观测性的基本构建块

| 概念 | 定义 | 类比 |
|------|------|------|
| **Trace** | 应用的一次完整运行(从输入到输出) | 一段完整的旅程 |
| **Span** | 单个步骤(LLM调用/工具调用/逻辑步骤) | 旅程中的一站 |
| **OpenTelemetry** | 应用可观测性的标准框架 | 通用的"监控语言" |

#### Phoenix UI 中的追踪视图

**Span 类型与颜色:**
- 橙色: LLM 调用
- 蓝色: Chain (逻辑步骤)
- 黄色: Tool (工具调用)

#### 插桩 (Instrumentation) 方法

两种方式添加追踪:
1. **自动插桩**: `OpenAIInstrumentor().instrument()` -- 自动追踪所有 OpenAI 调用
2. **手动插桩**: 使用 `with tracer.start_as_current_span()` 或 `@tracer.tool` 装饰器

> **生动比喻**: 给 Agent 加可观测性就像给手术室装摄像头 -- 你可以回看每一步操作,精确定位哪里出了问题,而不是靠猜测和 print 语句。

---

## 第6课: 实验2 - 追踪你的 Agent (Lab 2)

### 核心要点

**实现步骤:**
1. 导入 Phoenix + OpenTelemetry + OpenInference 库
2. 通过 `register()` 方法连接 Phoenix 实例
3. 使用 `OpenAIInstrumentor` 自动追踪 OpenAI 调用
4. 从**外向内**添加手动 Span:
   - 最外层: `start_main_span` (Agent Run, kind=AGENT)
   - 中间层: Router Call (kind=CHAIN)
   - 内层: 各工具调用 (kind=TOOL)
5. 使用**装饰器** `@tracer.tool` / `@tracer.chain` 简化插桩

**最佳实践**: 从最外层开始追踪,逐步深入 -- 这确保你先有全局视图,再细化到每个组件。

---

## 第7课: 添加路由器和技能评估 (Adding Router and Skill Evaluations)

### 核心要点

#### 三种评估技术

| 技术 | 适用场景 | 准确度 | 可扩展性 |
|------|---------|--------|---------|
| **代码评估** | 格式检查、正则匹配、Ground Truth对比 | 100% | 高 |
| **LLM-as-Judge** | 开放性指标(相关性、清晰度等) | <100% | 高 |
| **人工标注** | 任何场景(最灵活) | 100% | 低 |

#### LLM-as-Judge 工作流程

1. 获取应用的输入/输出
2. 构造评估 Prompt
3. 发送给独立的"法官 LLM"
4. 获得分类标签(如: correct/incorrect)

**关键原则:**
- 只用**顶级模型**(GPT-4, Claude 3.5 Sonnet)做 Judge
- 永远使用**离散分类标签**(correct/incorrect),而非模糊评分(1-100)
- LLM Judge 永远不会 100% 准确,但可以通过调优来逼近

> **生动比喻**: 让 LLM 打1-100分就像让人类评委给两道菜分别打83分和79分 -- 差异太主观。但让它说"好/不好"就像判断菜是生的还是熟的,简单明确得多。

#### 路由器评估的两个维度
1. **Function Calling 选择**: 是否选对了工具?
2. **参数提取**: 是否正确提取了参数?(如: 订单号 vs 物流号)

#### 技能评估示例
- **数据库查询**: SQL 生成正确性(代码评估 或 LLM Judge)
- **数据分析**: 分析清晰度 + 实体正确性(LLM Judge)
- **代码生成**: 代码是否可运行(代码评估)

---

## 第8课: 实验3 - 添加路由器和技能评估 (Lab 3)

### 核心要点

**评估流程 (Phoenix 模式):**
1. 从 Phoenix 导出相关 Span
2. 用 LLM Judge 或 代码评估 给每行添加标签
3. 上传标签回 Phoenix 进行可视化

**路由器评估实现:**
```python
# 使用 Phoenix 内置的 tool_calling_prompt_template
# 导出 LLM spans,过滤有 tool_call 的
# 用 llm_classify() 运行 LLM Judge
# 上传结果到 Phoenix
```

**代码评估实现 (可视化工具):**
```python
def code_is_runnable(generated_code):
    try:
        exec(generated_code)
        return True
    except:
        return False
```

**重要技巧:**
- 使用 `suppress_tracing` 避免评估调用本身被追踪
- 评估模板中的变量名要与导出数据的列名匹配
- 用 `score` 列(0/1)辅助数值化可视化

---

## 第9课: 添加轨迹评估 (Adding Trajectory Evaluations)

### 核心要点

#### 什么是轨迹 (Trajectory)?

轨迹 = Agent 为响应查询而经过的**路由 -> 工具 -> 路由 -> ... -> 用户**的完整路径。

#### 为什么轨迹很重要?

即使输出正确,**效率也至关重要**:
- 更少的步骤 = 更少的 LLM 调用 = 更低成本
- 更少的步骤 = 更低延迟 = 更好的用户体验
- 更少的步骤 = 更少的变量 = 更高可靠性

#### 收敛性评分 (Convergence Score)

**计算方法:**
1. 对一组相似查询运行 Agent
2. 记录每次运行的步骤数
3. 找到最短路径(最优路径)
4. 计算: `收敛分 = 最优路径的占比`

**公式**: 收敛分 = 1 意味着 Agent 100% 的时间都在走最优路径

**注意事项:**
- 如果所有运行都多了一个不必要的步骤,收敛评估**检测不出来**
- 只对**成功完成**的运行计算收敛性(排除错误中断的运行)

> **生动比喻**: 收敛性评估就像出租车公司检查司机是否走了最短路线 -- 如果10次从A到B,有3次绕远路,你的"收敛分"就是0.7。

---

## 第10课: 实验4 - 添加轨迹评估 (Lab 4)

### 核心要点

**Phoenix Experiments 工作流:**
1. **创建数据集**: 一组相似的测试查询
2. **定义任务**: 运行 Agent 并记录步骤数
3. **运行实验**: `run_experiment(dataset, task)`
4. **评估结果**: `evaluate_experiment(experiment, evaluators)`

**收敛评估实现:**
```python
# 1. 计算最优路径长度
optimal_path = min(all_path_lengths)

# 2. 对每个输出评分
def evaluate_path_length(output):
    return 1 if output['path_length'] == optimal_path else 0
```

---

## 第11课: 为评估添加结构 (Adding Structure to Your Evaluations)

### 核心要点

#### 评估驱动开发 (Evaluation-Driven Development)

**四步循环:**
1. **策划测试数据集** -- 代表性 > 穷举性,1-2个示例/类型即可
2. **运行实验** -- 改变模型/prompt/逻辑,产生不同版本
3. **运行评估器** -- 对每个版本打分
4. **比较与迭代** -- Apple-to-Apple 对比,选择最优版本

#### 实验仪表盘

理想状态: 每一行是 Agent 的一次运行,每一列是一个评估指标,形成全景视图。

**数据集来源:**
- 手动构造(初期)
- 从生产环境的真实数据中提取(后期)
- 尽可能包含 Expected Output(解锁更多评估方式)

**可以实验的变量:**
- Prompt 措辞
- 工具定义/描述
- 路由逻辑
- 技能实现
- 使用的 LLM 模型

> **生动比喻**: 评估驱动开发就像**A/B Testing的升级版** -- 不是单一指标的对比,而是用一整套"体检报告"来全面衡量每次改动的效果。

---

## 第12课: 实验5 - 为评估添加结构 (Lab 5)

### 核心要点

本实验将所有评估器组合成一个**大型综合实验**:

**5个评估器同时运行:**
1. `evaluate_tool_calling` -- 路由器 Function Calling (LLM Judge)
2. `evaluate_sql_result` -- SQL 结果正确性 (代码对比 Ground Truth)
3. `evaluate_clarity` -- 响应清晰度 (LLM Judge)
4. `evaluate_entity_correctness` -- 实体正确性 (LLM Judge)
5. `evaluate_code_runnable` -- 代码可运行性 (代码执行测试)

**实验结构:**
```python
run_experiment(
    dataset=overall_dataset,
    task=run_agent_task,
    evaluators=[eval1, eval2, eval3, eval4, eval5]
)
```

**迭代改进示例:**
- 发现 SQL 生成得分低 -> 修改 SQL 生成 Prompt (如: 添加 "Think before you respond")
- 重新运行实验为 V2 -> 对比 V1 和 V2 的各项得分
- 也可以使用 Phoenix **Playground** 直接在 UI 中快速迭代 Prompt

---

## 第13课: 改进你的 LLM-as-Judge (Improving Your LLM-as-a-Judge)

### 核心要点

#### 为什么要评估"评估器"本身?

- LLM Judge 永远不是 100% 准确的
- 可以通过**实验来评估和改进 Judge 本身**

#### 改进方法

**核心思路**: 用同一组数据同时运行:
- 代码评估(100% 准确,但需要 Ground Truth)
- LLM Judge(可扩展,但不完美)

然后对比两者结果,衡量 LLM Judge 的准确率。

**可以实验的方向:**
1. 修改 Judge Prompt 的措辞
2. 添加 **Few-shot 示例**(之前的正确判断)
3. 更换 Judge 模型
4. 使用**语义相似度**替代精确字符串匹配来评估

> **生动比喻**: 这就像**给法官考试** -- 你有标准答案(代码评估),然后看这个"法官"(LLM Judge)的判决有多接近标准答案。如果准确率太低,就换一个"法官"或给他更好的"判案指南"(prompt)。

---

## 第14课: 监控生产环境中的 Agent (Monitoring Agents)

### 核心要点

#### 从开发到生产的四步曲

1. **选择正确架构** -- 匹配用例的 Agent 框架
2. **确定评估指标** -- 准确率、延迟、收敛性
3. **构建评估结构** -- Prompt + 工具 + 数据
4. **用数据迭代** -- 分析结果、调整、重新测试

#### 生产环境的特殊挑战

- **新的失败模式**: 用户问出开发时从未见过的问题
- **更多外部依赖**: 额外的 API 调用带来更多出错机会
- **A/B 测试副作用**: 不同模型策略可能引入意外回归

#### 生产监控策略

1. **持续收集用户反馈** -- 附加人工标注
2. **维护 Golden Dataset** -- 捕获关键用例和已知失败模式
3. **CI/CD 集成** -- 每次推送变更都跑一遍实验
4. **自我改进循环**: 生产数据 -> 加入测试集 -> 重新实验 -> 改进 Agent

> **生动比喻**: 生产监控就像汽车的**行车记录仪** -- 开发阶段是在试车场调校,上路后你需要持续监控,遇到新路况(新查询)随时调整。

---

## 第15课: 总结 (Conclusion)

课程核心技能总结:
- 追踪 (Trace)
- 评估 (Evaluate)
- 改进 (Improve)

这些技能可以应用于**任何 Agent 框架**,不限于本课程使用的代码实现。

---

## 核心概念总结表

| 概念 | 定义 | 工具/方法 |
|------|------|----------|
| **Trace** | 应用一次完整运行的记录 | OpenTelemetry + Phoenix |
| **Span** | 运行中的单个步骤 | `tracer.start_as_current_span()` |
| **LLM-as-Judge** | 用独立LLM评估输出质量 | `llm_classify()` + Prompt Template |
| **代码评估** | 用代码逻辑检查输出 | 正则/执行/Ground Truth对比 |
| **收敛性** | Agent路径效率的度量 | min(steps) / actual_steps |
| **实验** | 数据集 + 任务 + 评估器的组合 | `run_experiment()` |
| **评估驱动开发** | 用评估结果指导Agent改进方向 | 循环: 数据集->实验->评估->迭代 |
| **可观测性** | 对应用每一层的完整可见性 | Phoenix UI + 插桩 |
| **插桩** | 标记哪些代码块需要追踪 | 装饰器 / with语句 |
| **Phoenix** | Arize的开源追踪和评估平台 | Projects / Traces / Experiments |

---

## 知识架构图

```
AI Agent 评估体系
|
|-- 1. 可观测性 (Observability)
|   |-- OpenTelemetry 标准
|   |-- Traces (完整运行)
|   |   |-- Spans (单步)
|   |       |-- LLM Span (模型调用)
|   |       |-- Tool Span (工具调用)
|   |       |-- Chain Span (逻辑步骤)
|   |-- 插桩方法
|   |   |-- 自动插桩 (OpenAIInstrumentor)
|   |   |-- 手动插桩 (装饰器 / with语句)
|   |-- Arize Phoenix (可视化平台)
|
|-- 2. 评估技术 (Evaluation Techniques)
|   |-- 代码评估 (Code-based)
|   |   |-- 正则匹配
|   |   |-- JSON可解析性
|   |   |-- 代码可执行性
|   |   |-- Ground Truth对比
|   |-- LLM-as-Judge
|   |   |-- Prompt Template 设计
|   |   |-- 离散分类标签 (correct/incorrect)
|   |   |-- 顶级模型 (GPT-4, Claude)
|   |-- 人工标注 (Human Annotations)
|       |-- 标注队列
|       |-- 用户反馈 (thumbs up/down)
|
|-- 3. 评估目标 (What to Evaluate)
|   |-- 路由器 (Router)
|   |   |-- Function Calling 选择正确性
|   |   |-- 参数提取准确性
|   |-- 技能 (Skills)
|   |   |-- 输出质量 (清晰度/相关性/正确性)
|   |   |-- 中间步骤正确性
|   |-- 轨迹 (Trajectory)
|       |-- 收敛性评分 (Convergence Score)
|       |-- 路径效率
|
|-- 4. 评估驱动开发 (Eval-Driven Development)
|   |-- 策划测试数据集
|   |-- 运行实验 (Experiments)
|   |   |-- 数据集 + 任务 + 评估器
|   |   |-- 多版本对比 (V1 vs V2 vs ...)
|   |-- 迭代改进
|   |   |-- 修改 Prompt
|   |   |-- 修改工具描述
|   |   |-- 更换模型
|   |   |-- 调整逻辑
|   |-- Phoenix Playground (快速迭代)
|
|-- 5. 生产监控 (Production Monitoring)
|   |-- 持续追踪与评估
|   |-- 用户反馈收集
|   |-- Golden Dataset 维护
|   |-- CI/CD 实验集成
|   |-- 自我改进循环
|
|-- 6. 改进评估器本身
    |-- 衡量 LLM Judge 准确率
    |-- Few-shot 示例优化
    |-- 语义相似度替代精确匹配
    |-- Judge Prompt 迭代
```

---

## 实用技巧速查

1. **评估选择决策树:**
   - 能用代码精确判断? -> 代码评估
   - 需要理解语义但不需100%准确? -> LLM-as-Judge
   - 需要100%准确且是开放性问题? -> 人工标注

2. **LLM Judge 最佳实践:**
   - 永远用离散标签,不用连续分数
   - 提供完整的工具定义上下文
   - 抑制评估调用的追踪 (`suppress_tracing`)
   - 使用 GPT-4 或同级模型

3. **实验设计原则:**
   - 数据集: 代表性 > 穷举性
   - 每次只改一个变量
   - 多次运行取平均(减少随机性影响)
   - 从生产数据中持续补充测试用例

4. **收敛性评估:**
   - 只用成功运行的数据
   - 对相似查询使用统一的步骤计算方式
   - 分数为1 = 最优,接近0 = 路径混乱
