# Unit 1.2: SAP Application Foundation Overview

## 元信息
- 来源: https://share.synthesia.io/80b6d286-622f-44f4-afd1-11a6445d2eaf
- 类型: 视频（Synthesia AI 生成培训课程）
- 作者: SAP（SAP Application Foundation 培训团队）
- 原始日期: 2026-08-03
- 摄入日期: 2026-08-04

## 内容

### 课程定位

本 Unit 属于 "SAP Application Foundation Code-Based Agents Training Course" 的第 1.2 节。前一节（Unit 1.1）介绍了 AI Agents、LLM、A2A 协议的基础概念。

### SAP Application Foundation 是什么

SAP Application Foundation 是一个专为构建 **code-based AI agents** 设计的综合平台。它提供从开发工具和 SDK 到部署基础设施和监控的全部能力。

**关键定位：Code-Based**
- 与 Low-code/No-code 方案不同，Application Foundation 面向需要完全控制 agent 行为的开发者
- 开发者用 Python 编写代码，定义逻辑
- 通过强大的 SDK 集成 SAP 服务

### 平台核心支柱

#### 1. Infrastructure（基础设施）

- 提供 SAP BTP 上的 managed infrastructure
- **Kyma Runtime**（QMAR）：基于 Kubernetes 的运行环境
  - Agent 被打包为 Docker 容器部署到 Kyma
  - 基于需求自动扩缩容
  - 无需手动管理服务器、配置网络或搭建 K8s 集群
- 完成 onboarding 后即获得 ready-to-use 的基础设施：
  - 配置好的 BTP sub-account
  - Kyma cluster 访问权限
  - 团队成员配置好的访问权限
- 支持 **Multi-region deployment**：
  - 可同时部署到 EU、US、Canada 等多个区域
  - 满足数据驻留要求（data residency）

#### 2. AI Services（AI 服务）—— SAP AI Core

- SAP AI Core：SAP 托管的 AI/ML 工作负载服务
- 提供强大的 LLM 访问能力：
  - 理解自然语言
  - 生成回复
  - 推理任务
- 安全性：
  - 使用 **OAuth 2.0** 认证
  - 仅授权 agent 可访问 AI 能力
  - 凭据安全管理 + 速率限制
- 合规性：
  - 审计追踪（audit trails）
  - 数据隔离（data isolation）
  - 满足各类监管框架
- 多区域配置：
  - 欧洲运营 → EU region endpoints
  - 美国运营 → US East region endpoints
  - 确保数据留在对应区域的数据中心

#### 3. Development Tools（开发工具）

**IDE：Visual Studio Code**
- 推荐的 agent 开发 IDE
- 扩展生态与其他工具无缝集成

**SAP Cloud SDK for Python**（开源库）
- 提供 BTP 服务的预构建集成
- 核心模块：
  - **Audit Log**：记录安全和合规事件
  - **Destination Service**：管理服务端点和凭据
  - **Object Store**：存储和检索二进制数据
  - **Telemetry**：代码可观测性插桩

**A2A Agent Template**
- 创建新 agent 时的起点模板
- 包含：正确的项目结构、预配置的 CI/CD pipelines、与核心服务的集成
- 不需要从零开始

### Skills 概念（开发辅助）

**核心理解：Skills 不是部署到 agent 中的能力，而是开发时辅助 Cline 的工具。**

- Skills 是预构建的能力，扩展 AI 编码助手（Cline）在开发过程中的能力
- 让 Cline 更懂 Application Foundation

**预置 Skills 清单：**
- **SAP Agent Bootstrap**：脚手架完整 agent 项目（文件 + 配置）
- **SAP Agent Instrumentation**：为 agent 添加可观测性
- **SAP Agent Run Local**：本地测试 agent

### 课程总结

本单元覆盖了：
1. BTP 基础设施（运行 agents）
2. SAP AI Core（提供智能）
3. 开发工具生态（加速开发）

下一模块（Module 2）将进入实操：完成 onboarding 流程、搭建开发环境。
