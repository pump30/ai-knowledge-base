# Ingest Knowledge Skill 设计文档

## 概述

创建一个 Claude Code skill（`.claude/skills/ingest-knowledge.md`），用于从多种来源自动摄入知识到 AI 知识库。用户只需贴一个 URL 或文件路径，skill 自动完成：提取内容 → 写入 sources → 生成/更新 VKI 页面 → 更新 index。

## 触发条件

- 用户贴了一个 URL（任意平台）
- 用户提到"学这个"、"ingest"、"摄入"
- 用户给了一个本地文件路径

## 路由规则

| 输入类型 | 识别方式 | 提取策略 |
|----------|----------|----------|
| Bilibili 视频 | `bilibili.com` / `b23.tv` | bbdown 下载 → 字幕优先，无字幕则 FunASR 转录 |
| YouTube 视频 | `youtube.com` / `youtu.be` | yt-dlp 提取字幕，无字幕则 Whisper 转录 |
| 小宇宙播客 | `xiaoyuzhoufm.com` | xiaoyuzhou-podcast skill（自带 FunASR） |
| 网页/博客 | 其他 http(s) URL | WebFetch 抓取 → 提取正文 |
| PDF 文件 | `.pdf` 后缀或本地路径 | Read 工具直接读取 |
| 电子书 | 书名+下载意图 | zlibrary skill 获取 → 解析 |
| 本地 markdown/txt | 本地路径 | 直接读取 |

## 内容处理

### 清洗

- 去除广告、无关片段
- 去除视频转录中的重复口头语（"嗯"、"那个"等）
- 保留核心知识点

### 结构化写入 sources

文件命名：`YYYY-MM-DD_descriptive-title.md`

文件格式：

```markdown
# 标题

## 元信息
- 来源: [URL]
- 类型: 视频/文章/播客/电子书
- 作者: xxx
- 原始日期: YYYY-MM-DD
- 摄入日期: YYYY-MM-DD

## 内容
（整理后的知识内容）
```

## VKI 自动生成

写入 sources 后，自动执行 CLAUDE.md 定义的 Ingest 流程：

1. 从内容中提取关键概念、工具、人物、组织
2. 匹配现有 VKI 页面：
   - 已存在 → 追加新信息，添加 source 引用
   - 不存在但重要 → 创建新页面
   - 不存在且只出现一次 → 标记观察，暂不创建（除非是核心概念）
3. 如果内容涉及 X vs Y 对比，创建 comparison 页面
4. 更新 `vki/index.md`
5. 输出摘要报告

### 报告格式

```
✓ 摄入完成: [标题]
  - 新建: concept/xxx, entity/yyy
  - 更新: concept/zzz（新增了关于...的信息）
  - 发现关联: [[a]] ↔ [[b]]
  - 观察: "某概念" 出现1次，暂不创建页面
```

## 错误处理

| 情况 | 处理方式 |
|------|----------|
| 视频无字幕且无法转录 | 提示用户，尝试用视频描述/评论区补充 |
| 网页反爬/需登录 | 提示用户手动复制内容或提供 cookies |
| 内容太短（<100字） | 仍然写入 sources，但 VKI 部分只更新已有页面，不创建新页面 |
| 内容与已有知识高度重复 | 只补充增量信息，不重复创建 |
| URL 无法识别类型 | 默认按网页处理，WebFetch 抓取 |

## 约束

- 不修改已有的 sources 文件（遵守 Layer 1 规则）
- 不删除 VKI 页面（只创建和更新）
- 不处理与 AI/LLM 无关的内容（提醒用户这是 AI 知识库）

## 技术依赖

- bbdown（Bilibili 下载）
- yt-dlp（YouTube 下载）
- Whisper / FunASR（语音转录）
- 已有 skills：xiaoyuzhou-podcast、zlibrary
- Claude Code 内置工具：WebFetch、Read、Write、Glob、Grep

## 文件结构

Skill 安装位置：`.claude/skills/ingest-knowledge.md`

纯 markdown 指令文件，无额外代码。Claude Code 按指令调用已有工具完成任务。
