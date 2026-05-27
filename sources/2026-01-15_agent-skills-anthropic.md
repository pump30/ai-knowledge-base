# Agent Skills with Anthropic - 课程笔记

> **课程来源**: DeepLearning.AI x Anthropic 合作课程  
> **讲师**: Elie Schoppik (回归讲师), Andrew Ng (主持)  
> **核心主题**: Agent Skills (代理技能) - 一种开放标准,用于扩展AI代理的能力

---

## 第一课: 课程介绍 (Introduction)

### 核心概念

Skills (技能) 是**指令文件夹**,用于扩展你的代理(Agent)的能力,为其注入专业知识。

**关键要点:**
- Skills 是一种**开放标准** (Open Standard),可跨多个平台使用(Claude Code、Codex、Gemini CLI 等)
- 每个 Skill 必须包含一个 `SKILL.md` 文件,其中有技能的名称、描述和主要指令
- Skills 采用**渐进式披露** (Progressive Disclosure):名称和描述始终在上下文窗口中,其余内容仅在触发时加载
- 代理需要基本工具:文件系统访问(读写文件)+ Bash 工具(执行代码)

### 生动比喻

> 把 Skills 想象成一本**食谱书**:书名和目录(名称+描述)始终摆在厨房台面上,但只有当你决定做某道菜时,才翻开对应的具体页面(加载详细指令)。这样既不占用台面空间(上下文窗口),又能随时找到需要的信息。

---

## 第三课: 为什么使用Skills - 上篇 (Why Use Skills - Part I)

### 从提示到技能的演变

本课通过一个**营销活动分析**场景,展示了从手动提示到打包为 Skill 的完整过程。

**问题**: 每周都需要重复相同的分析流程 - 数据质量检查、漏斗分析、效率分析、预算重新分配。

**解决方案**: 将重复性工作流打包成 Skill。

### SKILL.md 文件结构

```yaml
---
name: analyzing-marketing-campaign
description: 执行每周营销活动绩效分析...
---
```

**文件组织规范:**
- 文件夹命名: 小写字母 + 短横线连接 (如 `analyzing-marketing-campaign`)
- 不使用保留关键词 (如 claude, anthropic)
- SKILL.md 放在文件夹顶层
- 引用文件放在 `references/` 子文件夹

### Skill 文件夹结构示例

```
analyzing-marketing-campaign/
  SKILL.md                              # 主指令文件
  references/
    budget_reallocation_rules.md        # 仅在用户提问时加载
```

### 在 Claude AI 中上传使用

1. 将 Skill 文件夹打包为 ZIP
2. Settings -> Capabilities -> Skills -> Add
3. 上传后即可在新对话中自动触发

---

## 第四课: 为什么使用Skills - 下篇 (Why Use Skills - Part II)

### Skills 作为开放标准

Skills 已经成为跨平台的开放标准,支持:
- Claude Code, Claude AI, Claude Desktop
- Codex, Gemini CLI, Open Code 等

### Agent 架构的演进

| 阶段 | 特点 | 问题 |
|------|------|------|
| 单一目的代理 | 每个领域一个专门代理 | 难以扩展和维护 |
| 简单脚手架代理 | 基础工具(Bash+文件系统) | 缺乏领域专业知识 |
| **Skills加持的代理** | 简单脚手架 + 按需加载知识 | **最佳平衡** |

### Skills 的三大价值

1. **领域专业知识** (Domain Expertise): Claude 知道如何做数据分析,但不知道**你公司**如何做
2. **可重复工作流** (Repeatable Workflow): 在非确定性系统中创造可预测输出
3. **新能力** (New Capabilities): 教会代理原本不会的事情(如生成PPT、PDF)

### 渐进式披露 (Progressive Disclosure)

> **核心理念**: 上下文窗口是"公共资源" (Public Good),加入越多数据,token 消耗越快,回答质量可能下降。

**三层加载机制:**
1. **始终在上下文中**: 名称 + 描述
2. **触发时加载**: SKILL.md 主体内容
3. **按需加载**: 引用文件、脚本、资产

---

## 第五课: Skills vs Tools, MCP 和 Subagents

### 生态系统全景

### 类比: 建书架

| 组件 | 类比 | 作用 |
|------|------|------|
| **Tools (工具)** | 锤子、锯子、钉子 | 底层能力,访问系统的方式 |
| **Skills (技能)** | 如何建造书架的说明书 | 专业知识,告诉代理如何组合工具 |
| **MCP (模型上下文协议)** | 五金店 (提供工具和材料) | 连接外部系统和数据 |
| **Subagents (子代理)** | 分工合作的助手 | 隔离上下文,并行执行任务 |

### Skills + MCP 协作模式

- **MCP**: 带来外部数据和工具(如 BigQuery、Google Drive)
- **Skills**: 教代理如何使用这些数据,创建可重复工作流

### Skills + Subagents 协作模式

- 主代理作为协调器 (Orchestrator)
- 子代理拥有隔离的上下文窗口 + 细粒度权限
- 每个子代理可以使用特定的 Skills

### 总结对比

| 特性 | Prompts | Skills | Subagents | MCP |
|------|---------|--------|-----------|-----|
| 持久性 | 单次对话 | 跨对话 | 跨会话 | 实时连接 |
| 加载方式 | 立即 | 渐进式 | 独立上下文 | 按需 |
| 最佳用途 | 原子交互 | 可预测工作流 | 并行/隔离任务 | 外部数据 |

---

## 第六课: 探索预构建Skills (Exploring Pre-Built Skills)

### Anthropic 预构建 Skills 仓库

仓库地址: `github.com/anthropic/skills`

**两大类别:**
1. **文档技能** (Document Skills) - 始终内置于 Claude AI:
   - Excel 处理
   - PowerPoint 创建
   - Word 文档
   - PDF 处理

2. **示例技能** (Example Skills) - 可切换开关:
   - **Skill Creator** (默认开启) - 能自动创建新 Skills

### Skill Creator: 元技能

这是一个**用来创建 Skills 的 Skill** - 包含:
- `scripts/init_skill.py` - 初始化技能结构
- `scripts/package_skill.py` - 打包为 ZIP
- `scripts/validate_skill.py` - 验证技能格式

### 实战: 组合多个 Skills + MCP

**完整工作流演示:**
1. 用 Skill Creator 修改营销分析 Skill (CSV -> BigQuery)
2. 用 Skill Creator 创建品牌指南 Skill (logos + colors + fonts)
3. 组合: 营销分析 Skill + 品牌指南 Skill + 内置 PowerPoint Skill + BigQuery MCP
4. 输出: 带品牌风格的营销数据驱动 PPT

---

## 第七课: 创建自定义Skills (Creating Custom Skills)

### 最佳实践总结

#### 命名与描述

| 要素 | 规则 |
|------|------|
| 名称 | 小写字母 + 数字 + 连字符;使用 verb+ing 形式 |
| 描述 | 说明做什么 + 何时使用 + 触发关键词 |
| 可选字段 | license, compatibility, metadata |

#### SKILL.md 正文最佳实践

- **步骤化指令**: 明确的顺序步骤
- **边缘情况处理**: 指定何时可跳过步骤
- **保持简洁**: 正文不超过 500 行
- **路径格式**: 始终使用正斜杠 `/` (即使在 Windows 上)
- **自由度控制**: 低自由度 = 严格步骤;高自由度 = 创意输出

#### 标准目录结构

```
my-skill/
  SKILL.md              # 必需 - 主指令
  scripts/              # 可选 - 可执行代码
  references/           # 可选 - 参考文档
  assets/               # 可选 - 模板、图片、数据
```

### 两个实战案例

**案例1: 生成练习题 (generating-practice-questions)**
- 输入: 课程讲义 (PDF/LaTeX)
- 工作流: 提取学习目标 -> 按类型生成题目(判断题 -> 编程题 -> 应用题)
- 输出模板放在 `assets/` 中按需加载

**案例2: 分析时间序列 (analyzing-time-series)**
- 输入: CSV 时间序列数据
- 工作流: 运行 `diagnose.py` -> 生成可视化 -> 输出摘要
- 关键: Python 脚本在 `scripts/` 中,按确定性顺序执行

### 评估 Skills

使用 Skill Creator 来评估你的 Skills:
- 自动检查最佳实践
- 输出评分 + 改进建议 (如 9/10, 10/10)
- 类似于为软件写单元测试

---

## 第八课: Skills 与 Claude API

### 核心概念: Code Execution Tool + Files API

**不同环境的差异:**

| 环境 | 文件系统 | 代码执行 | 联网 |
|------|----------|----------|------|
| Claude AI/Desktop | 自动提供 | 自动提供 | 有 |
| Claude API | 需手动配置 | Code Execution Tool | 无 |
| Claude Code | 直接访问本地 | 直接访问 | 有 |

### Code Execution Tool

- 提供沙盒容器执行代码
- 有 RAM/磁盘/CPU 限制
- **API 中无网络连接** (Claude AI/Desktop 中有)
- 预装常用库

### Files API

- 上传/下载文件
- 文件有唯一 ID
- 与 Code Execution Tool 配合使用

### API 中使用 Skills 的代码模式

```python
# 1. 上传 Skill 目录
skill = client.skills.create(files_from_dir("my-skill/"), betas=[...])

# 2. 上传输入文件
file = client.files.create(open("data.csv"), betas=[...])

# 3. 发送消息 (带 Skills + Code Execution)
response = client.messages.create(
    model="claude-sonnet-...",
    container={"skills": [{"skill_id": skill.id}]},
    tools=[{"type": "code_execution"}],
    messages=[...],
    betas=["skills", "code-execution", "files-api"]
)

# 4. 下载生成的文件
client.files.download(file_id, betas=[...])
```

---

## 第九课: Skills 与 Claude Code

### Claude Code 中的 Skills 配置

**Skills 存放位置:**
```
project/
  .claude/
    skills/
      adding-cli-command/
        SKILL.md
      generating-cli-tests/
        SKILL.md
      reviewing-cli-command/
        SKILL.md
```

- **项目级**: `.claude/skills/` (项目目录)
- **用户级**: `~/.claude/skills/` (主目录)

### 实战: CLI 应用开发工作流

**项目**: Python CLI 任务管理应用 (使用 Typer + Rich)

**三个协作 Skills:**
1. `adding-cli-command` - 添加新命令的编码规范
2. `generating-cli-tests` - pytest 测试生成规范
3. `reviewing-cli-command` - 代码审查清单

### Subagents + Skills 结合

**关键发现**: 子代理**不会继承**父代理的 Skills,需显式指定!

```yaml
# .claude/agents/code-reviewer.yaml
name: code-reviewer
description: Reviews code for quality, security...
tools: [Bash, Glob, Grep, Read]
skills: [reviewing-cli-command]
```

**子代理中 Skills 的特殊行为:**
- 子代理启动时**完整加载** SKILL.md (不是仅名称+描述)
- 引用文件的渐进式加载仍然正常工作

### 完整工作流

1. 主代理使用 `adding-cli-command` Skill 编写 `edit.py`
2. 派遣 `code-reviewer` 子代理审查代码
3. 派遣 `test-generator-runner` 子代理生成测试
4. 主代理根据反馈修复问题

---

## 第十课: Skills 与 Claude Agent SDK

### 构建研究代理

**架构设计:**
- 主代理 (Orchestrator): 协调研究并合成结果
- 子代理 1: Documentation Researcher (WebSearch + WebFetch)
- 子代理 2: Repository Analyzer (Bash + WebSearch + Read)
- 子代理 3: Web Researcher (WebSearch + WebFetch)

### Skill: `learning-a-tool`

这个 Skill 引导主代理:
1. **研究阶段**: 并行派遣三个子代理
2. **组织阶段**: 按渐进式学习路径组织内容
3. **输出阶段**: 创建标准化学习指南

引用文件 `references/progressive_learning.md` 定义学习等级:
- Level 1: Overview & Motivation
- Level 2: Installation & Core Concepts
- Level 3: Practical Patterns
- Level 4: Where to Go Next

### Agent SDK 代码实现

```python
from claude_agent_sdk import Agent, AgentDefinition

agent = Agent(
    system_prompt=main_agent_prompt,
    allowed_tools=["Write", "Bash", "WebSearch", "WebFetch", 
                   "Task", "Skill", "mcp_notion_*"],
    agents={
        "docs_researcher": AgentDefinition(...),
        "repo_analyzer": AgentDefinition(...),
        "web_researcher": AgentDefinition(...)
    },
    mcp_servers={"notion": {...}},
    setting_sources=["user", "project"]  # Skills 来源
)
```

**关键配置:**
- `allowed_tools` 必须包含所有子代理需要的工具
- 添加 `"Skill"` 工具以启用 Skills 功能
- 添加 `"Task"` 工具以启用子代理派遣
- `setting_sources` 指定 Skills 搜索路径

### MCP 集成 (Notion)

研究完成后,使用 Notion MCP Server 将结果写入 Notion 页面,实现团队共享。

---

## 第十一课: 结论 (Conclusion)

### 创建 Skills 的最佳建议

1. **从简单开始**: 先用基础 Markdown 指令,后续再扩展
2. **遵循渐进式披露**: 不要把所有内容塞进 SKILL.md
3. **监控实际使用**: 观察代理如何使用你的 Skill,基于观察迭代
4. **描述要充分**: 确保代理知道何时使用该 Skill
5. **利用 Skill Creator**: 用它来检查最佳实践和自动生成结构

---

## 核心概念总结表

| 概念 | 定义 | 关键特征 |
|------|------|----------|
| **Skills** | 包含指令的文件夹,扩展代理能力 | 开放标准、渐进式披露、可移植 |
| **SKILL.md** | 技能的核心指令文件 | 必须包含 YAML Frontmatter (name + description) |
| **Progressive Disclosure** | 按需逐步加载信息 | 保护上下文窗口,避免污染 |
| **Skill Creator** | 自动创建 Skills 的元技能 | 包含 init/package/validate 脚本 |
| **Code Execution Tool** | API 中的沙盒代码执行环境 | 无网络、有资源限制、预装库 |
| **Files API** | 上传/下载文件的 API | 文件有唯一 ID,配合容器使用 |
| **Agent SDK** | 构建自定义代理的 SDK | 与 Claude Code 使用相同底层 |
| **MCP** | 模型上下文协议 | 连接外部数据源和工具 |
| **Subagents** | 子代理 | 隔离上下文、并行执行、受限权限 |
| **Open Standard** | Skills 的开放规范 | 跨平台兼容 (Claude/Codex/Gemini等) |

---

## 知识架构图

```
Agent Skills 知识体系
├── 基础概念
│   ├── 什么是 Skills (指令文件夹)
│   ├── 开放标准规范
│   ├── SKILL.md 结构 (YAML Frontmatter + Markdown Body)
│   └── 渐进式披露机制 (名称/描述 -> SKILL.md -> 引用文件)
│
├── Skills 在生态中的定位
│   ├── vs Tools (工具是底层能力, Skills是组合说明)
│   ├── vs MCP (MCP提供外部数据, Skills教如何使用)
│   ├── vs Subagents (子代理并行执行, Skills提供知识)
│   └── 可组合性 (多个Skills + MCP + Subagents)
│
├── 创建最佳实践
│   ├── 命名规范 (lowercase-with-hyphens, verb+ing)
│   ├── 描述编写 (做什么 + 何时触发 + 关键词)
│   ├── 正文结构 (步骤化、<500行、处理边缘情况)
│   ├── 目录组织 (scripts/ + references/ + assets/)
│   └── 评估方法 (Skill Creator评分 + 单元测试思维)
│
├── 跨平台使用
│   ├── Claude AI / Desktop (内置Skills, 拖拽上传ZIP)
│   ├── Claude API (Code Execution Tool + Files API + betas)
│   ├── Claude Code (.claude/skills/ 目录, /skills命令)
│   └── Agent SDK (Skill工具 + setting_sources配置)
│
├── 高级模式
│   ├── Skills + MCP (BigQuery数据 -> 分析技能 -> PPT技能)
│   ├── Skills + Subagents (每个子代理配备专属Skills)
│   ├── Skill Creator (元技能, 自动创建新Skills)
│   └── 多Skills协作 (分析 + 品牌 + 文档生成)
│
└── 实战项目
    ├── 营销活动分析 (CSV/BigQuery -> 漏斗/效率分析)
    ├── 练习题生成 (讲义 -> 多类型题目)
    ├── 时间序列诊断 (CSV -> Python脚本 -> 可视化)
    ├── CLI开发工作流 (编码/测试/审查三Skills)
    └── 研究代理 (SDK + 三子代理 + Notion MCP)
```

---

## 学习建议

1. **入门**: 先在 Claude AI 中手动创建一个简单 Skill,体会从提示到打包的过程
2. **进阶**: 使用 Skill Creator 自动化创建,学习最佳实践
3. **实战**: 在 Claude Code 中为你的项目添加项目级 Skills
4. **高级**: 使用 Agent SDK 构建带 Skills 的多代理系统

> **记住**: Skills 的核心价值是将你的专业知识转化为可移植、可重复、可共享的标准化资产。就像软件工程中的"基础设施即代码",Skills 实现了"专业知识即代码"。
