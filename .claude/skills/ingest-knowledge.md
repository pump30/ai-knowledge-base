---
name: ingest-knowledge
description: 从 URL 或本地文件自动摄入知识到 AI 知识库。支持 Bilibili、YouTube、小宇宙播客、网页、PDF、电子书、本地文件。用户贴 URL 或说"学这个"/"ingest"即触发。
---

# Ingest Knowledge

从多种来源自动提取知识并摄入到 AI 知识库。

## 触发条件

当以下任一条件满足时触发此 skill：
- 用户贴了一个 URL（视频、文章、播客等）
- 用户说"学这个"、"ingest"、"摄入"、"加入知识库"
- 用户给了一个本地文件路径（.pdf、.md、.txt、.epub）

## Pre-Ingest（执行前的准备）

在 Phase 1 之前必须先完成以下步骤。任何一步失败或检测到不应自动处理的状态时，**停下并向用户报告**，不要继续。

1. **检查工作区干净**：运行 `git status --short`。
   - 输出非空 → 停下，列出未提交的文件，提示用户："工作区有未提交改动，请先 commit 或 stash 再摄入。"
2. **切回 main**：运行 `git checkout main`。
   - 失败（例如分支不存在、checkout 冲突）→ 停下报告。
3. **拉取最新**：运行 `git pull origin main`。
   - 报告冲突或失败 → 停下，输出 stderr，不要自动 resolve、不要 rebase。
4. **创建摄入分支**：运行 `git checkout -b ingest/YYYY-MM-DD-<slug>`。
   - `YYYY-MM-DD` 用今天的日期。
   - `<slug>` = 内容主题的 kebab-case 短描述（与即将写入的 source 文件名 slug 保持一致）。
   - 若分支名已存在 → 依次尝试 `-2`、`-3`、…，直到找到未占用的名字。
   - 若当前已在某个 `ingest/*` 分支上，先 `git checkout main` 再走步骤 3、4，避免分支套娃。

### 分支命名示例

- `ingest/2026-05-26-rag-evaluation-best-practices`
- `ingest/2026-05-26-langgraph-streaming-2`（若 `-langgraph-streaming` 已存在）

## 执行流程

### Phase 1: 识别来源类型并提取内容

根据输入识别类型，按对应策略提取：

**Bilibili 视频** (`bilibili.com`, `b23.tv`):
1. 使用 bbdown-bilibili-downloader skill 下载视频
2. 优先提取字幕文件（.srt/.ass）
3. 若无字幕，使用 FunASR 对音频进行语音转录
4. 提取视频标题、UP主、发布日期

**YouTube 视频** (`youtube.com`, `youtu.be`):
1. 运行 `yt-dlp --write-auto-sub --sub-lang zh,en --skip-download --print title --print uploader --print upload_date <URL>` 获取字幕和元信息
2. 若无字幕，运行 `yt-dlp -x --audio-format wav <URL>` 下载音频，然后用 Whisper 转录：`whisper audio.wav --language zh --output_format txt`
3. 提取视频标题、频道名、发布日期

**小宇宙播客** (`xiaoyuzhoufm.com`):
1. 使用 xiaoyuzhou-podcast skill（自带 FunASR 转录）
2. 获取完整转录文本和节目信息

**网页/博客** (其他 http/https URL):
1. 使用 WebFetch 工具抓取页面内容
2. 提取正文、标题、作者、发布日期
3. 去除导航栏、侧边栏、广告等无关内容

**PDF 文件** (`.pdf` 后缀或本地路径):
1. 使用 Read 工具读取 PDF 内容（大文件分页读取）
2. 提取标题、作者信息

**电子书** (用户提到书名+下载意图):
1. 使用 zlibrary skill 搜索并获取电子书
2. 解析内容（epub 用 pandoc 转 markdown，pdf 用 Read）

**本地 markdown/txt** (本地文件路径):
1. 使用 Read 工具直接读取文件内容

### Phase 2: 清洗与结构化

对提取的原始内容进行处理：

1. **去噪**：
   - 移除广告、赞助内容、无关片段
   - 移除视频转录中的重复口头语（"嗯"、"那个"、"然后"、"就是说"）
   - 移除时间戳标记（保留内容本身）

2. **结构化**：
   - 识别内容的主要章节/话题
   - 用 markdown 标题组织层次
   - 保留所有核心知识点和技术细节
   - 保留原文中的代码示例

3. **语言**：保持原始语言，技术术语保留英文

### Phase 3: 写入 sources

使用 Write 工具创建 source 文件：

**文件路径**: `sources/YYYY-MM-DD_descriptive-title.md`
- YYYY-MM-DD 使用今天的日期
- descriptive-title 用英文短横线连接，描述内容主题

**文件格式**:

```markdown
# 标题

## 元信息
- 来源: URL 或文件路径
- 类型: 视频/文章/播客/电子书/文件
- 作者: 作者名
- 原始日期: YYYY-MM-DD（内容发布日期，未知则留空）
- 摄入日期: YYYY-MM-DD（今天）

## 内容
（整理后的结构化知识内容）
```

### Phase 4: VKI Ingest（知识网络更新）

写入 sources 后，执行 CLAUDE.md 中定义的 Ingest 流程：

1. **识别概念与实体**：从内容中提取：
   - 概念（抽象方法、模式、理论）
   - 实体（工具、框架、模型、人物、组织）
   - 对比关系（X vs Y）

2. **匹配现有 VKI 页面**（先读取 `vki/index.md`）：
   - 已有页面 → 在对应 section 追加新信息，在 Sources 添加 `[[source-file-name]]`
   - 无页面但是核心概念 → 创建新页面（使用 CLAUDE.md 中定义的模板）
   - 无页面且只出现一次 → 在报告中标记为"观察"，暂不创建

3. **处理比较关系**：
   - 若内容包含明确的 X vs Y 对比 → 创建或更新 `vki/comparisons/` 页面

4. **更新索引**：更新 `vki/index.md`，添加新页面条目

5. **交叉引用**：确保新页面与已有页面用 `[[page-name]]` 互相链接

### Phase 6: Open PR（提交并发起 Pull Request）

执行顺序：**Phase 6 在 Phase 5 之前**，这样 Phase 5 的报告可以包含 PR URL。

1. **只 add 本次产物**：对 Phase 3 写入的 source 文件和 Phase 4 创建/更新的 VKI 文件，逐个 `git add <path>`。**禁止** `git add -A` 或 `git add .`，避免误提交无关文件。
2. **Commit**（用 HEREDOC，不要用 `--amend`）：

   ```bash
   git commit -m "$(cat <<'EOF'
   Ingest: <source title>

   Source: <URL or local path>
   - New: vki/concepts/xxx.md, vki/entities/yyy.md
   - Updated: vki/concepts/zzz.md
   EOF
   )"
   ```

   - 第一行 ≤ 70 字符；超出则截断 `<source title>`。
   - `New:` 或 `Updated:` 行为空时省略该行。

3. **推送分支**：`git push -u origin <branch-name>`。失败 → 保留本地 commit，向用户报告 stderr。

4. **创建 PR**（push 成功后）：

   ```bash
   gh pr create --base main --head <branch-name> \
     --title "Ingest: <source title>" \
     --body "$(cat <<'EOF'
   Source: <URL or local path>

   ## VKI changes
   - New: vki/concepts/xxx.md, vki/entities/yyy.md
   - Updated: vki/concepts/zzz.md

   ## Notes
   <Phase 5 报告中的 observations，无则省略整段>
   EOF
   )"
   ```

5. **捕获 PR URL**：从 `gh pr create` 的 stdout 取最后一行（即 PR URL）。

6. **不 merge**：用户手动 merge。Skill 永不调用 `gh pr merge` 或 `git merge`。

### Phase 6 失败处理

| 情况 | 行为 |
|------|------|
| `git add` 后无 staged 改动 | 跳过 commit 和 PR，Phase 5 报告写"无新内容可发布" |
| `git push` 失败（认证/网络） | 保留本地 commit 和分支；Phase 5 写 push 失败信息（见下） |
| `gh pr create` 失败 | 远端分支已存在；Phase 5 输出 compare URL 让用户手动开 PR |

### Phase 5: 输出报告

完成后输出摘要：

```
✓ 摄入完成: [标题]
  - 来源: [URL/路径]
  - 新建: concept/xxx, entity/yyy
  - 更新: concept/zzz（新增了关于...的信息）
  - 发现关联: [[a]] ↔ [[b]]
  - 观察: "某概念" 出现1次，暂不创建页面
✓ 已推送分支: ingest/YYYY-MM-DD-<slug>
✓ PR 已创建: https://github.com/pump30/ai-knowledge-base/pull/N
```

**Push 失败时**，最后两行替换为：

```
✗ Push 失败: <stderr>。本地分支 ingest/... 和 commit <hash> 已保留。
```

**`gh pr create` 失败（push 成功）时**，最后两行替换为：

```
✓ 分支已推送: ingest/...
✗ gh pr create 失败: <stderr>。请手动开 PR: https://github.com/pump30/ai-knowledge-base/compare/main...ingest/...
```

**无新内容可发布时**（Phase 6 step 1 检测到 nothing to stage），最后两行替换为：

```
ℹ 无新内容可发布（VKI 与 sources 均无变更）。
```

## 错误处理

| 情况 | 处理方式 |
|------|----------|
| 视频无字幕且转录失败 | 告知用户，尝试从视频描述/评论区提取信息作为补充 |
| 网页反爬/需登录 | 告知用户，请他们手动复制内容粘贴过来 |
| 内容太短（<100字有效内容） | 仍写入 sources，但 VKI 部分只更新已有页面，不创建新页面 |
| 内容与已有知识高度重复 | 只补充增量信息到已有页面，不重复创建 |
| URL 无法识别类型 | 默认按网页处理，使用 WebFetch 抓取 |
| 内容与 AI/LLM 主题无关 | 提醒用户这是 AI 知识库，询问是否仍要摄入 |

## 约束

- **不修改**已有的 sources 文件（Layer 1 规则：创建后不变）
- **不删除** VKI 页面（只创建和更新）
- 新建 VKI 页面必须严格遵循 CLAUDE.md 中定义的模板格式
- 所有 VKI 页面使用 `[[page-name]]` wiki-link 语法交叉引用
- VKI 内容语言：中文为主，英文技术术语保留原文
