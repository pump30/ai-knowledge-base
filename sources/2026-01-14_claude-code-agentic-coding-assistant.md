# Claude Code：高度自主的编程助手

> 课程来源：DeepLearning.AI & Anthropic  
> 讲师：Elie Schoppik（Anthropic 技术教育负责人）  
> 课程主题：Claude Code 的最佳实践与高效使用方法

---

## 第1课：课程介绍

### Claude Code 的定位
- AI 辅助编程的演进：提问 -> 自动补全 -> 自主编程助手
- Claude Code 是**高度自主（Highly Agentic）**的编码工具
- 可以持续工作数分钟，自主完成复杂任务
- 开发者可编排**多个 Claude 实例并行工作**

### 核心最佳实践
1. **提供清晰上下文**：指向相关文件
2. **清楚描述功能需求**
3. **扩展能力**：通过 MCP Server 和其他工具

### 课程三大实战项目
1. **RAG Chatbot**：前后端开发、重构、测试、GitHub 集成
2. **Jupyter Notebook**：电商数据分析、仪表盘创建
3. **Figma 设计稿**：通过 MCP Server 导入设计并构建前端

---

## 第2课：什么是 Claude Code

### 核心架构：模型 + 工具 + 环境

```
用户 -> Claude Code Harness（轻量外壳）
           ├── 模型（Opus / Sonnet）
           ├── 工具集（内置 + MCP 扩展）
           └── 环境（命令行，代码库本地）
```

### 内置工具列表

| 类别 | 工具 |
|------|------|
| 文件编辑 | Edit（编辑文件）、Write（写入新文件）、NotebookEdit |
| 文件读取 | Read（读取文件）、Glob（文件模式匹配） |
| 搜索 | Grep（内容搜索）、WebSearch |
| 执行 | Bash（执行 shell 命令） |
| 高级 | SubAgent（子代理处理困难任务） |

### 关键设计决策：不索引代码库

| 传统方案 | Claude Code 方案 |
|---------|-----------------|
| 语义嵌入整个代码库 | Agentic Search（按需搜索） |
| 代码发送到服务器 | 代码保持本地 |
| 需要预处理时间 | 即开即用 |
| 安全顾虑 | 安全性更好 |

### CLAUDE.md 记忆系统
- Markdown 文件，自动加载到上下文
- 存储：配置、风格指南、代码库结构说明
- 对话历史存储在本地，可清除或恢复

### 可扩展性
- 通过 **MCP Server** 添加额外工具
- 示例：Figma Server、Playwright Server、数据库 Server

> **生动比喻**：Claude Code 就像一个极其聪明的新同事——你不需要把整个代码库打印给他看，只需要告诉他去哪里找文件（工具），他会自己翻阅、理解、然后动手改代码。CLAUDE.md 就是你给他的"入职手册"。

---

## 第4课：环境搭建与代码库理解

### CLAUDE.md 的三种作用域

| 位置 | 作用域 | 用途 |
|------|--------|------|
| `~/.claude/CLAUDE.md` | 全局（所有项目） | 个人偏好、通用规范 |
| 项目根目录 `CLAUDE.md` | 项目级（团队共享） | 项目架构、编码规范 |
| 子目录 `CLAUDE.md` | 目录级 | 特定模块说明 |

### 代码库理解最佳实践
- 首先让 Claude Code **解释代码库结构**
- 使用 `/init` 命令自动生成 CLAUDE.md
- 让 Claude 绘制架构图，验证理解是否正确

---

## 第5课：添加功能

### Plan 模式
- 在编写代码前，先让 Claude **制定计划**
- `shift + tab` 切换 Plan 模式
- Claude 先分析需求，列出步骤，确认后再执行

### MCP Playwright 集成
- 连接 Playwright MCP Server
- Claude 可以自动打开浏览器、截图验证 UI
- 实现"编码-预览-修正"循环

> **生动比喻**：Plan 模式就像让建筑师先画图纸再施工——避免"边建边拆"的浪费。Playwright 集成则让 Claude 自己当"验收员"，不用你来回切换浏览器检查。

---

## 第6课：测试、错误调试与代码重构

### 测试工作流
1. 让 Claude 编写测试用例
2. 运行测试，发现失败
3. Claude 自动分析错误并修复
4. 重复直到所有测试通过

### 子代理（SubAgent）重构
- 对于复杂重构任务，Claude 可以启动**子代理**
- 子代理处理特定子任务，主代理协调整体
- 降低单次上下文负担，提高质量

---

## 第7课：同时添加多个功能

### Git Worktrees（工作树）
- 创建**隔离的工作目录**，每个分支独立
- 多个 Claude 实例可以**并行工作**在不同 worktree
- 避免分支切换冲突

### 自定义命令（Custom Commands）
- 在 `.claude/commands/` 目录创建 Markdown 文件
- 定义可重复使用的任务模板
- 通过 `/` 斜杠命令调用

### 并行开发流程
```
main
├── worktree-1: Claude 实例 A 开发功能 1
├── worktree-2: Claude 实例 B 开发功能 2
└── worktree-3: Claude 实例 C 修复 bug
```

> **生动比喻**：Git Worktrees + 多实例 Claude 就像一个建筑工地有多个工队同时施工——水电队、装修队、外墙队各干各的，互不干扰，最后合并验收。

---

## 第8课：GitHub 集成与 Hooks

### GitHub 集成
- Claude Code 可以直接操作 GitHub
- 创建 PR、审查代码、修复 Issue
- 使用 `gh` CLI 工具

### Hooks（钩子）
- 在特定事件触发时自动执行操作
- 配置在 `settings.json` 中
- 示例：提交前自动运行 lint、测试完成后通知

---

## 第9课：重构 Jupyter Notebook 与创建仪表盘

### Notebook 重构
- Claude Code 可以读取和编辑 `.ipynb` 文件
- 移除冗余代码、优化分析逻辑
- 提取关键洞察

### 创建 Streamlit 仪表盘
- 从 Notebook 数据分析 -> Web 仪表盘
- Claude 自动生成 Streamlit 应用
- 交互式可视化

---

## 第10课：基于 Figma 设计稿创建 Web 应用

### Figma MCP Server
- 连接 Figma 设计文件
- Claude 读取设计规范（颜色、布局、组件）
- 自动生成对应的前端代码

### 端到端流程
1. Figma 设计稿 -> MCP Server 提取设计信息
2. Claude Code 生成 HTML/CSS/JS
3. Playwright MCP 截图验证
4. 迭代修正直到匹配设计

---

## 第11课：总结

### Claude Code 核心能力回顾
- 代码库发现与理解
- 功能开发与重构
- 测试与调试
- 并行多任务开发
- GitHub 工作流集成
- 数据分析与可视化
- 设计稿实现

---

## 总结表

| 课程主题 | 关键技术 | 实用建议 |
|---------|---------|---------|
| 架构理解 | 模型 + 工具 + 环境 | 不需索引代码库，本地安全 |
| 记忆系统 | CLAUDE.md（全局/项目/目录） | 用 /init 自动生成 |
| 功能开发 | Plan 模式 + Playwright 验证 | 先计划后执行 |
| 测试调试 | 自动写测试 + 修复循环 | 让 Claude 自己跑测试 |
| 并行开发 | Git Worktrees + 多实例 | 每个功能一个 worktree |
| GitHub | PR/Issue 操作 + Hooks | 自动化工作流 |
| 数据分析 | Notebook 编辑 + Streamlit | 从分析到仪表盘一步到位 |
| 设计实现 | Figma MCP + Playwright MCP | 设计稿自动变代码 |

---

## 知识树

```
Claude Code — 高度自主编程助手
├── 核心架构
│   ├── 轻量 Harness（不索引代码库）
│   ├── Agentic Search（按需搜索文件）
│   ├── 内置工具（Read/Edit/Grep/Glob/Bash）
│   └── MCP 扩展（Playwright/Figma/自定义）
├── 记忆与上下文
│   ├── CLAUDE.md（三级作用域：全局/项目/目录）
│   ├── 对话历史（本地存储，可清除/恢复）
│   └── /init 命令（自动生成项目文档）
├── 工作模式
│   ├── Plan 模式（先规划后执行）
│   ├── 子代理（SubAgent 处理复杂子任务）
│   ├── 自定义命令（.claude/commands/）
│   └── Hooks（事件触发自动操作）
├── 并行开发
│   ├── Git Worktrees（隔离工作目录）
│   ├── 多实例并行（每个 worktree 一个 Claude）
│   └── 最终合并
├── 集成生态
│   ├── GitHub（PR/Issue/代码审查）
│   ├── VS Code（可视化变更）
│   ├── Playwright MCP（浏览器自动化验证）
│   └── Figma MCP（设计稿导入）
└── 实战场景
    ├── RAG Chatbot 全栈开发
    ├── Jupyter Notebook 重构 + 仪表盘
    └── Figma 设计 -> Web 应用
```
