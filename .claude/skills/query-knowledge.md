---
name: query-knowledge
description: 从 AI 知识库查询答案。优先使用 VKI 网络中的内容，KB 未覆盖时补充训练知识，并明确区分两者来源。可选保存为 vki/articles/ 文章并走 PR 流程。仅显式触发（Skill 工具或 /query-knowledge）。
---

# Query Knowledge

从 AI 知识库（VKI 网络）查询答案，并将高质量回答保存为 article。

## 触发条件

仅显式触发：

- 通过 `Skill` 工具调用，`skill: "query-knowledge"`，问题作为 `args` 传入
- `/query-knowledge <question>` slash 命令

**不**注册关键词触发。普通问答行为由 CLAUDE.md 顶部的最高优先级 banner（"任何用户提问，必须先读 vki/index.md"）保证，不与本 skill 抢答。

## 执行流程

### Phase 1: 检索

1. **读 index**：使用 `Read` 工具读取 `vki/index.md`。
   - 文件不存在或读不到 → 停下报告，不要继续。
2. **匹配候选**：扫描 index 的四个表格区段（Concepts / Entities / Comparisons / Articles），按问题关键词匹配 Title 或 Description 列。
3. **挑选**：取至多 **5 个**最相关的候选页面。
4. **报告候选**：输出给用户：

   ```
   📚 知识库检索结果（找到 N 个候选）：
     - [[page-1]] — <description>
     - [[page-2]] — <description>
     ...
   ```

5. **空命中**：候选数 N == 0 → 跳过 Phase 2 与 Phase 4（不询问保存），直接进入 Phase 3 走 KB-empty 路径。

### Phase 2: 深读

1. 对 Phase 1 选出的候选页面逐个 `Read`。
2. 在每个页面内查看 `## Sources` 与 `## Related` 列表，对其中的 `[[wiki-link]]`：
   - 仅当 **明显有助于回答** 时再 Read 一层（**最多深入一层**，不要链式 3+ 层跳跃）
   - 候选已能回答问题就停。
3. 本阶段总读取数上限 **10 个文件**（5 个候选 + 至多 5 个一阶扩展）。

### Phase 3: 合成

输出**至多两段**，按以下顺序：

#### 来自知识库

- 答案主体。
- **每个被引用的概念/实体/对比都必须用 `[[page-name]]` wiki-link 标注**——不要写裸名。
- 多页面信息要融合成连贯的论述，不要堆砌页面摘要。

#### 来自训练知识（KB 未覆盖部分）

- **仅当** KB 内容不足以完整回答时出现。
- 普通文字，**不要**用 `[[wiki-link]]`（外部知识不在 KB 内）。
- 开头一句必须是：`以下内容不在知识库中，来自训练知识：`

#### KB 完全空命中的情形

若 Phase 1 候选数为 0：

```
ℹ️ 知识库未覆盖此话题。

以下内容不在知识库中，来自训练知识：
<answer>
```

此时**不**进入 Phase 4，避免污染 KB。

### Phase 4: 询问保存

仅当 Phase 1 至少有 1 个 KB 候选被使用时，在答案末尾追加：

```
📝 这个回答要不要保存为 vki/articles/<proposed-slug>.md？(yes/no)
```

`<proposed-slug>` = 由问题主题派生的 kebab-case 短描述。

- 用户答 yes → 进入 Phase 5
- 其他回应 → 结束（不写 article、不动 git）

### Phase 5: 保存为 article（仅 yes 时执行）

git 流程与 `ingest-knowledge` 完全一致，只是分支前缀改为 `article/`。

#### Pre-write 检查
1. `git status --short`：非空 → 停下，列出 dirty 文件，提示用户先 commit/stash。
2. `git checkout main`。
3. `git pull origin main`：冲突或失败 → 停下，输出 stderr，**不要**自动 resolve 或 rebase。
4. `git checkout -b article/YYYY-MM-DD-<slug>`：
   - 分支名已占用 → 依次尝试 `-2`、`-3`、…，直到唯一。
   - 当前已在某个 `article/*` 或 `ingest/*` 分支上 → 先 `git checkout main` 再走步骤 3、4。

#### 写 article

文件路径：`vki/articles/<slug>.md`

模板（严格遵循）：

```markdown
# <Concise Title (从问题派生)>

## Question
<原问题，逐字保留>

## Answer
<Phase 3 输出的答案主体；KB 段在前，含 [[wiki-links]]；外部段在后，无链接>

## Sources Consulted
- [[page-name-1]]
- [[page-name-2]]

## External Knowledge Used
<一两句话描述哪部分来自训练知识；如果回答 100% 来自 KB，**整段省略**>

## Date
YYYY-MM-DD
```

#### 更新 index

在 `vki/index.md` 的 `## Articles` 表中追加一行：

```
| [[<slug>]] | <one-line description> | YYYY-MM-DD |
```

若该表当前是占位行 `(暂无 - 等待 Query 产出)`，**用新行替换占位行**（不要保留占位）。

#### Commit + push + PR

1. **只 add 本次产物**：`git add vki/articles/<slug>.md vki/index.md`。**禁止** `git add -A` 或 `git add .`。
2. Commit（HEREDOC，不要 `--amend`）：

   ```bash
   git commit -m "$(cat <<'EOF'
   Article: <Title>

   Question: <原问题>
   - New: vki/articles/<slug>.md
   - Updated: vki/index.md
   EOF
   )"
   ```

   - 第一行 ≤ 70 字符；超出则截断 Title。

3. 推送：`git push -u origin article/YYYY-MM-DD-<slug>`。失败 → 保留本地 commit，向用户报告 stderr。

4. 创建 PR：

   ```bash
   gh pr create --base main --head article/YYYY-MM-DD-<slug> \
     --title "Article: <Title>" \
     --body "$(cat <<'EOF'
   ## Question
   <原问题>

   ## Answer summary
   <2-3 句话总结答案>

   ## Sources consulted
   - [[page-1]]
   - [[page-2]]

   ## VKI changes
   - New: `vki/articles/<slug>.md`
   - Updated: `vki/index.md`（新增 1 条 article 记录）
   EOF
   )"
   ```

5. **捕获 PR URL**：取 `gh pr create` stdout 最后一行。

6. **不 merge**：用户手动 merge。Skill 永不调用 `gh pr merge` 或 `git merge`。

### Phase 5 失败处理

| 情况 | 行为 |
|------|------|
| Pre-write 工作区脏 | 停下，列出 dirty 文件，提示先 commit 或 stash |
| Pre-write `git pull` 冲突或失败 | 停下，输出 stderr，**不要**自动 resolve 或 rebase |
| Pre-write 分支名已占用 | 依次尝试 `-2`、`-3`、…，直到唯一 |
| Pre-write 当前在 `article/*` 或 `ingest/*` 分支 | 先 `git checkout main` 再继续 |
| Save 后无 staged 改动 | 跳过 commit 和 PR，报告"无 article 文件写入" |
| `git push` 失败（认证/网络） | 保留本地 commit，向用户报告 stderr |
| `gh pr create` 失败 | 远端分支已保留，输出 compare URL 让用户手动开 PR |

## 输出示例

### 示例 A — KB 全覆盖
用户调用：`args: "什么是 RAG？"`

```
📚 知识库检索结果（找到 3 个候选）：
  - [[rag-retrieval-augmented-generation]] — RAG 全流程：检索增强生成、Sentence Window、Auto-merging
  - [[rag-vs-fine-tuning]] — 外部检索 vs 权重修改的知识注入方式
  - [[embedding-models]] — Embedding 演进：Word2Vec→BERT→Dual Encoder

来自知识库：

[[rag-retrieval-augmented-generation]] 是把外部检索结果注入 LLM 上下文的技术…
对比 [[rag-vs-fine-tuning]]，RAG 的优势是…

📝 这个回答要不要保存为 vki/articles/what-is-rag.md？(yes/no)
```

### 示例 B — KB 部分覆盖，外部补充

```
📚 知识库检索结果（找到 1 个候选）：
  - [[transformer-architecture]] — Transformer 核心架构

来自知识库：
[[transformer-architecture]] 包含…

来自训练知识（KB 未覆盖部分）：
以下内容不在知识库中，来自训练知识：
关于 Mamba/SSM 这类替代架构…

📝 这个回答要不要保存为 vki/articles/transformer-vs-mamba.md？(yes/no)
```

### 示例 C — KB 完全未覆盖

```
ℹ️ 知识库未覆盖此话题。

以下内容不在知识库中，来自训练知识：
<answer>
```

（**不**询问保存。）

## 约束

- **不修改**已有的 sources 文件（Layer 1 write-once 规则）。
- **不删除** VKI 页面（包括 articles）；如需大幅修订内容，建议 ingest 新 source 而不是改 article。
- 新建 article 必须严格遵循上述模板。
- 答案中所有 KB 段引用必须用 `[[page-name]]` wiki-link，外部段不能用。
- 内容语言：中文为主，英文技术术语保留原文。
- Skill 永不 merge PR；用户手动 merge。
