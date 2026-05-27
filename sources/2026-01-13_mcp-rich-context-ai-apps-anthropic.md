# MCP：使用 Anthropic 构建富上下文 AI 应用

> 课程来源：DeepLearning.AI & Anthropic  
> 讲师：Elie Schoppik（Anthropic 技术教育负责人）  
> 课程主题：Model Context Protocol（模型上下文协议）的核心概念与实践

---

## 第1课：课程介绍

### 什么是 MCP
- **Model Context Protocol（模型上下文协议）**：一个开放协议，标准化 LLM 应用访问外部工具和数据资源的方式
- 基于**客户端-服务器架构**
- 2024年11月由 Anthropic 发布，生态系统快速增长
- **模型无关（Model Agnostic）**——可与任何 LLM 配合使用

### 起源
- 源自 Anthropic 内部项目——扩展 Claude Desktop 与本地文件系统和外部系统的交互能力
- 发现该协议对许多 AI 应用普遍适用后开源发布

### 课程内容
1. MCP 客户端-服务器架构详解
2. 构建并测试 MCP Server
3. 构建 MCP Client 和 Chatbot
4. 连接第三方参考服务器
5. 配置 Claude Desktop
6. 部署远程 MCP Server

> **生动比喻**：MCP 就像 USB 接口——在 USB 出现前，每种设备都有自己独特的接口；有了 USB，"一次构建，到处使用"。MCP 是 AI 世界的"USB 标准"。

---

## 第2课：为什么需要 MCP

### 核心问题
- **模型只有上下文好，才能表现好**——再智能的模型，没有外部数据连接就受限
- 没有 MCP 的世界：每个 AI 应用为每个数据源重复编写集成代码（M x N 问题）

### MCP 的价值主张
- **一次构建，到处使用**（Build once, use everywhere）
- 类比 **LSP（Language Server Protocol）**：微软2016年开发，标准化 IDE 与语言工具的交互
- 类比 **REST**：标准化 Web 应用与后端的通信

### 各角色受益

| 角色 | 收益 |
|------|------|
| 应用开发者 | 用极少代码连接 MCP Server |
| API 开发者 | 构建一次 Server，被所有应用采用 |
| AI 用户 | 提供 URL 即可获取数据访问 |
| 企业 | 关注点分离，独立集成供多团队共用 |

### 实战演示
- Claude Desktop 连接 GitHub MCP Server + Asana MCP Server
- 用自然语言：读取 GitHub issue -> 自动在 Asana 创建并分配任务

> **生动比喻**：没有 MCP 之前，让 AI 连接不同数据源就像给每个电器都配一个专用转接头；有了 MCP，就像所有电器都用标准插座——即插即用。

---

## 第3课：MCP 架构

### 核心架构

```
Host（宿主应用，如 Claude Desktop）
├── MCP Client 1 ──── 1:1 连接 ──── MCP Server A（GitHub）
├── MCP Client 2 ──── 1:1 连接 ──── MCP Server B（Google Drive）
└── MCP Client 3 ──── 1:1 连接 ──── MCP Server C（SQLite）
```

- **Host**：LLM 应用，管理多个 Client（如 Claude Desktop、Cursor、Windsurf）
- **Client**：与单个 Server 保持 1:1 连接
- **Server**：轻量级程序，暴露特定功能

### 三大原语（Primitives）

| 原语 | 类比 | 说明 | 控制方 |
|------|------|------|--------|
| **Tools（工具）** | POST 请求 | 可被调用的函数，用于检索/修改数据 | 模型控制 |
| **Resources（资源）** | GET 请求 | 只读数据/上下文，服务器暴露给客户端 | 应用控制 |
| **Prompts（提示模板）** | 预设表单 | 预定义的提示，减轻用户 prompt engineering 负担 | 用户控制 |

### Python SDK 代码示例
```python
# 工具：装饰器声明
@mcp.tool()
def query_database(sql: str) -> str:
    ...

# 资源：通过 URI 暴露
@mcp.resource("docs://documents")
def list_documents() -> str:
    ...

# 提示模板：预定义 prompt
@mcp.prompt()
def summarize_data(topic: str) -> list[Message]:
    ...
```

### 传输机制（Transport）

| 传输方式 | 适用场景 | 特点 |
|---------|---------|------|
| **Standard IO** | 本地服务器 | Client 启动 Server 为子进程 |
| **HTTP + SSE** | 远程服务器（有状态） | 保持连接，支持服务器推送事件 |
| **Streamable HTTP** | 远程服务器（新版） | 支持有状态和无状态，未来推荐 |

### 通信流程
1. **初始化**：Client 发请求 -> Server 回响应 -> 确认通知
2. **消息交换**：双向请求和通知
3. **终止**：关闭连接

> **生动比喻**：三大原语就像餐厅服务——Tools 是"点菜"（主动发起请求改变状态），Resources 是"自助沙拉台"（数据摆在那里你随时取），Prompts 是"推荐套餐"（厨师帮你搭配好，不用自己研究菜单）。

---

## 第4课：Chatbot 示例

### 工具定义与使用
- 使用 Anthropic API 的 tool_use 功能
- 工具定义包含：name、description、input_schema
- LLM 决定何时调用工具，返回结构化工具调用请求

---

## 第5课：创建 MCP Server

### FastMCP 框架
- 使用 Python SDK 的高级 API
- 用装饰器声明工具、资源、提示模板
- 使用 **MCP Inspector** 调试和测试服务器

### 开发流程
1. 创建 FastMCP 实例
2. 用 `@mcp.tool()` 装饰器定义工具
3. 用 `@mcp.resource()` 装饰器定义资源
4. 启动服务器，用 Inspector 测试

---

## 第6课：创建 MCP Client

### Client 实现要点
- 建立与 Server 的连接（通过 StdIO transport）
- 发现可用工具并转换为 LLM tool schema
- 处理 LLM 的工具调用请求，转发给 Server 执行
- 将结果返回 LLM 继续对话

---

## 第7课：连接参考服务器

### 多服务器连接
- 一个 Host 可以连接多个 MCP Server
- 通过 JSON 配置文件管理服务器列表
- 每个服务器独立管理自己的工具和资源

### 配置示例
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
    }
  }
}
```

---

## 第8课：添加 Prompt 和 Resource 功能

### Resources 实现
- 直接资源：固定 URI 返回数据
- 模板资源：URI 中含动态参数（如 `docs://users/{id}`）
- 客户端通过 `@` 符号引用资源

### Prompts 实现
- 服务器提供预定义的高质量 prompt
- 用户只需填入动态数据
- 减轻用户 prompt engineering 负担

---

## 第9课：配置 Claude Desktop

### 配置方式
- 编辑 `claude_desktop_config.json` 文件
- 指定 MCP Server 的启动命令和参数
- Claude Desktop 作为 Host 自动管理 Client 连接

### MCP 兼容客户端
- Claude Desktop、Claude AI
- Cursor、Windsurf、VS Code
- 其他社区开发的 MCP 兼容应用

---

## 第10课：创建和部署远程服务器

### 远程部署
- 使用 HTTP + SSE transport
- 部署到云平台（如 Cloudflare Workers、Modal）
- 远程服务器可被多个应用共享

### 与本地服务器的区别
- 本地：StdIO transport，Client 启动 Server 子进程
- 远程：HTTP transport，Server 独立运行，Client 连接 URL

---

## 第11课：总结与未来展望

### 已学内容回顾
- Host / Client / Server 架构
- Tools / Resources / Prompts 三大原语
- 构建 Server 和 Client
- 远程部署

### 进阶特性

| 特性 | 说明 |
|------|------|
| **Authentication（认证）** | OAuth 2.1，2025年3月规范更新加入 |
| **Roots（根目录）** | Client 建议 Server 操作的文件路径范围 |
| **Sampling（采样）** | Server 反向请求 LLM 推理（方向反转） |

### 未来路线图
- **统一注册中心（Registry API）**：发现、验证、版本管理 MCP Server
- **动态发现**：Agent 自动搜索并连接所需 Server
- **多 Agent 架构**：Client 可以是 Server，Server 可以是 Client（递归/组合式）
- **命名冲突解决**：多 Server 场景下工具名去重
- **Streamable HTTP** 全面推广

### 多 Agent 架构愿景
```
User + LLM
    ↓
Agent（同时是 MCP Client + Server）
    ├── 分析 Agent（MCP Server）
    ├── 编码 Agent（MCP Server）
    └── 研究 Agent（MCP Server）
         └── 连接其他 Server...
```

> **生动比喻**：MCP 的 Sampling 就像老板（Client）派助理（Server）去调查问题，助理可以自己请教专家（反向调用 LLM）后再汇报结果，而不是把所有原始资料都搬回老板桌上。

---

## 总结表

| 课程主题 | 关键技术 | 实用建议 |
|---------|---------|---------|
| 为什么 MCP | 标准化集成，解决 M x N 问题 | 构建一次 Server，到处复用 |
| 架构 | Host > Client > Server（1:1） | 理解三层职责分离 |
| 原语 | Tools / Resources / Prompts | Tools 给模型，Resources 给应用，Prompts 给用户 |
| Server 开发 | FastMCP + Python SDK + Inspector | 用装饰器快速声明能力 |
| Client 开发 | 工具发现 + 调用转发 | 转换 Server 工具为 LLM schema |
| 多服务器 | JSON 配置管理 | 每个数据源一个 Server |
| 远程部署 | HTTP + SSE / Streamable HTTP | 生产环境用 OAuth 2.1 认证 |
| 未来 | Registry + 动态发现 + Multi-Agent | 关注协议规范更新 |

---

## 知识树

```
MCP（Model Context Protocol）
├── 核心架构
│   ├── Host（宿主应用：Claude Desktop / Cursor / Windsurf）
│   ├── Client（1:1 连接管理）
│   └── Server（暴露能力的轻量程序）
├── 三大原语（Server 暴露）
│   ├── Tools（工具）—— 模型控制，类似 POST
│   ├── Resources（资源）—— 应用控制，类似 GET
│   └── Prompts（提示模板）—— 用户控制，预定义 prompt
├── Client 原语
│   ├── Roots（建议操作路径范围）
│   └── Sampling（Server 反向请求推理）
├── 传输层（Transport）
│   ├── Standard IO（本地，子进程通信）
│   ├── HTTP + SSE（远程，有状态）
│   └── Streamable HTTP（远程，支持有状态+无状态）
├── 开发实践
│   ├── FastMCP Python SDK（装饰器声明）
│   ├── MCP Inspector（调试测试）
│   ├── JSON 配置文件（多服务器管理）
│   └── 远程部署（云平台 + OAuth 2.1）
├── 通信流程
│   ├── 初始化（握手）
│   ├── 消息交换（双向请求/通知）
│   └── 终止（断开连接）
└── 未来方向
    ├── Registry API（注册、发现、验证、版本管理）
    ├── 动态发现（.well-known/mcp.json）
    ├── 多 Agent 架构（Client 即 Server，递归组合）
    └── 认证增强（OAuth 2.1 + 授权控制）
```
