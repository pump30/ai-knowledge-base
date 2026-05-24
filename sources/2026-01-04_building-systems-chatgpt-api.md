# Building Systems with the ChatGPT API 学习笔记

> 课程来源：DeepLearning.AI  
> 讲师：Andrew Ng & Isa Fulford (OpenAI)

---

## 第1课：Introduction（引言）

### 核心概念
- 构建复杂应用需要**多步骤处理**，而非单次LLM调用
- 端到端客户服务助手示例流程：
  1. 评估输入（检查有害内容）
  2. 处理输入（分类查询类型）
  3. 检索相关信息
  4. 生成回复
  5. 检查输出（确保准确性和安全性）

### 关键要点
- 应用通常需要多个对用户不可见的内部步骤
- 系统需要持续改进和评估

![系统概览](screenshots/01_system_overview.jpg)

---

## 第2课：Language Models, the Chat Format and Tokens（语言模型、聊天格式与Token）

### LLM工作原理
- **监督学习**是训练LLM的核心模块
- LLM通过反复预测下一个**token**（而非"词"）来生成文本
- 训练流程：大量文本数据 -> 预测下一个词 -> Base LLM

### Base LLM vs Instruction-tuned LLM
- Base LLM：单纯预测下一个词
- Instruction-tuned LLM：经过指令微调 + RLHF，能遵循指令
- 从Base到Instruction-tuned只需数天（相比Base训练需数月）

### Token机制
- 英语中 1 token 约等于 4 个字符或 3/4 个单词
- "lollipop" 被分为 "l"、"oll"、"ipop" 三个token
- 这解释了为什么LLM难以逐字母反转单词
- **技巧**：用破折号分隔字母（如 l-o-l-l-i-p-o-p）让模型正确处理

### Chat Format（聊天格式）
- **System Message**：设定助手整体行为和角色
- **User Message**：用户的具体请求
- **Assistant Message**：模型的回复
- GPT-3.5 Turbo 限制约4000 tokens（输入+输出）

### 开发效率对比
| | 传统ML | 基于Prompting |
|---|---|---|
| 数据准备 | 数周-数月 | 无需 |
| 模型训练 | 数天-数月 | 无需 |
| 部署 | 数天-数周 | 即时API调用 |
| 总耗时 | 6个月-1年 | 分钟-小时 |

### 生动比喻
> Token就像**乐高积木**——模型不是一个字母一个字母地看文字，而是把常见的字母组合拼成一块"积木"来处理。"lollipop"变成了三块积木"l|oll|ipop"，所以模型看不到中间具体有哪些字母。

![Base vs Instruction-tuned](screenshots/02_base_vs_instruction.jpg)
![Chat格式](screenshots/02_chat_format.jpg)
![Token机制](screenshots/02_tokens.jpg)
![监督学习](screenshots/02_supervised_learning.jpg)
![Prompting vs 传统方法](screenshots/02_prompting_vs_traditional.jpg)

---

## 第3课：Classification（分类）

### 核心思想
- 对用户输入进行**分类**，然后根据分类结果选择不同的处理指令
- 定义固定类别 + 硬编码各类别的处理逻辑

### 实战案例
- 主类别：Billing、Technical Support、Account Management、General Inquiry
- 次类别：Unsubscribe、Upgrade、Close Account等
- 输出格式：JSON（primary + secondary）
- 示例："I want to delete my profile" -> Account Management / Close Account

### 生动比喻
> 分类就像医院的**分诊台**——患者进门先分诊，感冒去内科，骨折去骨科。不同类别的问题走不同的处理流程，而不是所有问题用同一套指令。

![分类示例](screenshots/03_classification.jpg)

---

## 第4课：Moderation（内容审核）

### OpenAI Moderation API
- 免费使用，用于检测有害内容
- 分类类别：hate、self-harm、sexual、violence等
- 返回各类别的flagged状态和score分数
- 可同时用于**输入审核**和**输出审核**

### 防范Prompt Injection（提示词注入）
**策略1：使用分隔符 + 系统指令**
- 将用户输入放在分隔符内
- 在系统消息中明确指令，并移除用户消息中的分隔符

**策略2：额外的分类检测**
- 用另一个prompt判断用户是否在尝试注入
- 输出Y/N（是否为注入尝试）
- 使用few-shot示例提升分类准确性

### 生动比喻
> Moderation API就像餐厅的**安检门**——进门要查包（检查输入），出门也要查包（检查输出），确保没有违禁品进出。Prompt Injection就像有人试图**冒充工作人员混入后厨**，我们需要多重身份验证来阻止。

![Moderation API](screenshots/04_moderation_api.jpg)
![Prompt Injection防御](screenshots/04_prompt_injection.jpg)

---

## 第5课：Chain of Thought Reasoning（思维链推理）

### 核心概念
- 让模型在回答前**分步骤推理**，减少错误
- **Inner Monologue（内心独白）**：隐藏模型的推理过程，只展示最终结果给用户

### 实战案例：产品咨询助手
步骤设计：
1. 判断用户是否在问特定产品
2. 确认产品是否在可用列表中
3. 列出用户做的假设
4. 验证假设是否正确
5. 礼貌地纠正错误假设

### 使用分隔符截取最终回复
- 模型输出中用分隔符标记各步骤
- 代码只提取最后一个分隔符之后的内容展示给用户

### 生动比喻
> Chain of Thought就像**数学考试要求写出解题步骤**——直接写答案容易出错，但一步步写出推导过程，最终答案的正确率大大提高。Inner Monologue就像**考官在心里打分**——学生看不到打分过程，只看到最终得分。

![Chain of Thought](screenshots/05_chain_of_thought.jpg)

---

## 第6课：Chaining Prompts（链式提示）

### 核心思想
- 将复杂任务拆分为多个简单子任务，每个子任务用一个prompt处理
- 前一步的输出作为后一步的输入

### 与Chain of Thought的对比
| Chain of Thought | Chaining Prompts |
|---|---|
| 一个prompt内分步骤 | 多个prompt串联 |
| 像一次做完整顿饭 | 像分阶段烹饪 |
| 像一个复杂函数 | 像模块化代码 |

### Chaining Prompts的优势
- 每步更简单，准确率更高
- 可以在某步加入人工或代码逻辑
- 更容易调试和维护
- 减少不必要的token消耗（只在需要时加载信息）
- 更灵活——可以根据上一步结果决定下一步走哪条路

### 实战案例
1. 提取产品和类别
2. 查找产品详细信息
3. 用详细信息生成回复

### 生动比喻
> Chaining Prompts就像**工厂流水线**——每个工位只负责一道工序（切割、焊接、喷漆），每道工序简单可控。而Chain of Thought就像让一个人**从头到尾手工打造**整件产品。流水线更适合复杂系统。

---

## 第7课：Check Outputs（输出检查）

### 核心要点
- 生成输出后、展示给用户前，需要**质量检查**
- 两种检查方式：
  1. **Moderation API**：检查输出是否包含有害内容
  2. **Model自查**：用另一个prompt评估输出质量（如是否基于提供的产品信息）

### 自查Prompt示例
- 让模型判断回复是否充分回答了用户问题
- 让模型判断回复中的信息是否与提供的产品信息一致

### 生动比喻
> 输出检查就像文章发表前的**编辑审稿**——先让Moderation API做"政治正确检查"，再让模型做"事实核查"，双重保险确保用户看到的内容既安全又准确。

![输出检查](screenshots/07_check_outputs.jpg)

---

## 第8课：Evaluation（端到端系统评估）

### 端到端流程整合
1. Moderation API检查输入
2. 提取产品列表
3. 查找产品信息
4. 用模型生成回复
5. Moderation API检查输出
6. 返回给用户

### 生动比喻
> 端到端系统就像一家**高端餐厅的服务流程**——客人点菜（输入）-> 前台确认没有不合理要求（审核）-> 分配给对应厨师（分类）-> 厨师查食材库存（检索）-> 烹饪（生成）-> 品控试吃（输出检查）-> 上菜（展示）。

![评估流程](screenshots/08_evaluation_process.jpg)

---

## 第9课：Evaluation Part I（评估方法一）

### 评估思路
- 从开发集（少量样本）开始评估
- 逐步扩展到更大的测试集
- 建立可重复的自动化评估流程

### 评估层次递进
1. **手动检查**：少量样本，肉眼看结果
2. **规则评估**：自动检查分类是否正确、产品列表是否匹配
3. **LLM评估**：用另一个LLM来评判输出质量
4. **大规模评估**：数百个测试用例 + 自动化pipeline

### 生动比喻
> 评估LLM系统就像**餐厅质量管理**——先让主厨自己尝几道菜（手动检查），然后制定标准食谱对照（规则评估），再请美食评论家打分（LLM评估），最后上线后收集顾客评价（大规模评估）。

![评估开发流程](screenshots/09_eval_development.jpg)

---

## 第10课：Evaluation Part II（评估方法二：无标准答案时）

### 问题
- 当LLM输出是开放式文本（没有唯一正确答案）时如何评估？

### 解决方案：用LLM评估LLM
- 设计评估prompt，让模型以rubric（评分标准）给输出打分
- 评估维度：准确性、相关性、完整性、格式等
- 对比模型输出与"理想回答"的差异

### 生动比喻
> 就像**作文考试**——没有唯一正确答案，但有评分标准。让一个"评委LLM"按照rubric给"选手LLM"的作文打分，虽然不完美，但比没有评估好得多。

![Rubric评估](screenshots/10_rubric_eval.jpg)

---

## 第11课：Summary（总结）

### 课程回顾
- LLM工作原理（tokenizer机制）
- 输入评估（分类 + Moderation + Prompt Injection防御）
- 输入处理（Chain of Thought + Chaining Prompts）
- 输出检查（Moderation + 自查）
- 系统评估（规则 + LLM评估 + 持续改进）
- 负责任地构建安全、准确的AI系统

---

## 总结表格

| 课时 | 主题 | 核心技能 |
|------|------|----------|
| 01 | Introduction | 多步骤系统架构概览 |
| 02 | Language Models & Tokens | LLM原理/Token/Chat格式 |
| 03 | Classification | 查询分类路由 |
| 04 | Moderation | 内容审核/Prompt Injection防御 |
| 05 | Chain of Thought | 分步推理/内心独白 |
| 06 | Chaining Prompts | 多prompt串联/流水线 |
| 07 | Check Outputs | 输出质量与安全检查 |
| 08 | Evaluation (End-to-End) | 端到端系统整合 |
| 09 | Evaluation Part I | 分层评估方法论 |
| 10 | Evaluation Part II | 用LLM评估LLM（Rubric） |
| 11 | Summary | 课程总结 |

---

## 知识树 (ASCII)

```
Building Systems with the ChatGPT API
├── LLM基础
│   ├── 监督学习 → 预测下一个Token
│   ├── Base LLM vs Instruction-tuned LLM
│   ├── Token机制（分词器）
│   └── Chat Format（System/User/Assistant）
├── 输入评估 (Evaluate Inputs)
│   ├── 分类 (Classification)
│   │   ├── 主类别 + 次类别
│   │   └── JSON结构化输出 → 路由到不同处理逻辑
│   ├── 内容审核 (Moderation API)
│   │   ├── 暴力/仇恨/自残/色情检测
│   │   └── 自定义阈值策略
│   └── Prompt Injection防御
│       ├── 分隔符隔离用户输入
│       └── 分类检测注入意图
├── 输入处理 (Process Inputs)
│   ├── Chain of Thought Reasoning
│   │   ├── 分步推理
│   │   └── Inner Monologue（隐藏推理过程）
│   └── Chaining Prompts
│       ├── 任务拆分为子任务
│       ├── 信息按需加载
│       └── 条件分支处理
├── 输出检查 (Check Outputs)
│   ├── Moderation API（有害内容）
│   └── Model自查（事实一致性）
└── 系统评估 (Evaluation)
    ├── 手动检查（少量样本）
    ├── 规则评估（自动化断言）
    ├── LLM评估（Rubric评分）
    └── 持续改进（扩展测试集）
```
