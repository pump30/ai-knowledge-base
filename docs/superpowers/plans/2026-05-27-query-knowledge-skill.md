# Query-Knowledge Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create the `query-knowledge` skill that answers user questions from the VKI knowledge base, falls back to training knowledge for gaps, and optionally saves answers as `vki/articles/` via the same PR workflow `ingest-knowledge` uses.

**Architecture:** Single Markdown instruction file (`.claude/skills/query-knowledge.md`) that the LLM follows at runtime, mirroring the structure of the existing `ingest-knowledge` skill. A user-level symlink at `~/.claude/skills/query-knowledge/SKILL.md` makes the skill discoverable by Claude Code globally.

**Tech Stack:** Markdown (skill instructions). Bash (`git`, `gh` CLI for the article PR flow). No code, no tests — verification is manual file inspection plus environment checks.

---

## File Structure

- **Create**: `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md` — the skill instructions, written in Chinese-with-English-tech-terms voice matching `ingest-knowledge.md`. Single file, ~150 lines, self-contained.
- **Create**: `/Users/I572881/.claude/skills/query-knowledge/SKILL.md` — symlink → the file above (per spec's Installation section, mirrors how `ingest-knowledge` is wired up).

No edits to any other repo file. CLAUDE.md is intentionally untouched (per spec Non-Goals).

---

### Task 1: Write the query-knowledge skill file

**Files:**
- Create: `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md`

- [ ] **Step 1: Confirm sibling skill exists for style reference**

Run the `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/ingest-knowledge.md`.

Expected: file readable; first line is `---` (frontmatter open); lines 2-3 are `name:` and `description:`. The skill uses Chinese prose with English technical terms, numbered Phase headings, and tables for failure handling. Match this voice and structure.

- [ ] **Step 2: Create the skill file**

Use the `Write` tool with `file_path` = `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md` and `content` = the full text below (copy verbatim, no edits):

````markdown
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
````

- [ ] **Step 3: Verify file written correctly**

Run the `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md`.

Expected:
- First three lines:

  ```
  ---
  name: query-knowledge
  description: 从 AI 知识库查询答案。…
  ```

- File contains all five Phase headings: `### Phase 1: 检索`, `### Phase 2: 深读`, `### Phase 3: 合成`, `### Phase 4: 询问保存`, `### Phase 5: 保存为 article（仅 yes 时执行）`.
- File contains the `### Phase 5 失败处理` table with 7 rows.
- File contains all three example sections (A, B, C).
- File ends with the `## 约束` section.

If anything is missing → re-run Step 2 (the Write tool overwrites cleanly).

- [ ] **Step 4: Commit**

```bash
git -C /Users/I572881/workspace/ai-knowledge-base add .claude/skills/query-knowledge.md
git -C /Users/I572881/workspace/ai-knowledge-base commit -m "Add query-knowledge skill"
```

Expected: one commit, one file added (`.claude/skills/query-knowledge.md`).

---

### Task 2: Install user-level symlink

**Files:**
- Create: `/Users/I572881/.claude/skills/query-knowledge/SKILL.md` (as a symlink)

This mirrors how `ingest-knowledge` is wired up, so any Claude Code session at any working directory can discover the skill via the user-level skills folder.

- [ ] **Step 1: Confirm parent exists**

Run:
```bash
ls -la /Users/I572881/.claude/skills/ | head -5
```

Expected: directory exists and contains other skill subdirectories (e.g., `ingest-knowledge`, various `impeccable:*`).

- [ ] **Step 2: Create the skill subdirectory**

Run:
```bash
mkdir -p /Users/I572881/.claude/skills/query-knowledge
```

Expected: no output. Directory either already exists or is created.

- [ ] **Step 3: Create the symlink**

Run:
```bash
ln -s /Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md /Users/I572881/.claude/skills/query-knowledge/SKILL.md
```

Expected: no output. If `ln` fails because the target already exists, run `rm /Users/I572881/.claude/skills/query-knowledge/SKILL.md` first, then re-run the `ln` command.

- [ ] **Step 4: Verify the symlink resolves**

Run:
```bash
ls -la /Users/I572881/.claude/skills/query-knowledge/SKILL.md
head -3 /Users/I572881/.claude/skills/query-knowledge/SKILL.md
```

Expected:
- `ls -la` shows `lrwxr-xr-x ... SKILL.md -> /Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md`
- `head -3` outputs:
  ```
  ---
  name: query-knowledge
  description: 从 AI 知识库查询答案。…
  ```

If the head output is empty or shows a different file → the symlink target is wrong. Remove and re-create.

- [ ] **Step 5: No commit needed**

The symlink lives outside the repo (in `~/.claude/`). Nothing to commit for this task.

---

### Task 3: Manual dry-run verification

**Files:**
- Read-only checks across the skill file, the symlink, the existing ingest skill, and the environment.

- [ ] **Step 1: Re-read the skill file end-to-end**

Run the `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/query-knowledge.md` (no `offset`/`limit`).

Verify in order:

1. Frontmatter present at the top: `---` / `name: query-knowledge` / `description: ...` / `---`.
2. `## 触发条件` section says "仅显式触发" and references both `Skill` tool invocation and `/query-knowledge` slash command.
3. Five `### Phase` headings appear in order: 检索, 深读, 合成, 询问保存, 保存为 article（仅 yes 时执行）.
4. The `### Phase 5 失败处理` table has 7 rows (Pre-write 工作区脏, `git pull` 冲突, 分支名已占用, 当前在 `article/*` 或 `ingest/*` 分支, Save 无 staged, `git push` 失败, `gh pr create` 失败).
5. Three examples present and labeled (A, B, C).
6. `## 约束` section is the last top-level section.
7. No `TBD`, `<placeholder>` that doesn't make sense, no duplicate phase headings.

- [ ] **Step 2: Confirm parity with ingest-knowledge skill on git/PR conventions**

Run the `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/ingest-knowledge.md` and compare side-by-side that:

1. Both skills use the same Pre-Ingest/Pre-write step structure: `git status --short` → `git checkout main` → `git pull origin main` → `git checkout -b <prefix>/YYYY-MM-DD-<slug>`.
2. Both skills forbid `git add -A` and require staging specific files.
3. Both skills use HEREDOC for commit messages and require ≤ 70 char first line.
4. Both skills use `gh pr create --base main --head <branch>` and capture the PR URL from stdout.
5. Both skills explicitly state they never merge.
6. Failure handling row counts and intent are aligned (query has 7; ingest had 6, plus one extra row in this skill for `ingest/*` branch awareness — confirm this is intentional).

If any divergence (other than branch prefix `article/` vs `ingest/` and the explicit `article/*-or-ingest/*` checkout-main row) → fix the query skill to match ingest's wording.

- [ ] **Step 3: Verify environment prerequisites**

Run:
```bash
git -C /Users/I572881/workspace/ai-knowledge-base remote -v
gh auth status 2>&1 | head -10
```

Expected:
- `origin` points to `https://github.com/pump30/ai-knowledge-base.git`.
- `gh auth status` shows logged in to `github.com` with `repo` scope (token shown as `gho_***`).

If either check fails → surface the gap to the user. Don't proceed.

- [ ] **Step 4: Confirm clean repo state and commit history**

Run:
```bash
git -C /Users/I572881/workspace/ai-knowledge-base status --short
git -C /Users/I572881/workspace/ai-knowledge-base log --oneline -3
```

Expected:
- `git status --short` is empty (no uncommitted changes).
- The most recent commit is `Add query-knowledge skill` from Task 1 Step 4.
- The commit before that is the spec commit `Add design spec for query-knowledge skill`.

If status is non-empty → there are stray changes. Investigate before claiming the skill is ready.

- [ ] **Step 5: No commit needed for verification**

This task is read-only. If any fix was applied during verification, that fix gets its own focused commit (e.g., `Fix typo in query-knowledge skill`).

---

## Self-Review Notes

- **Spec coverage**:
  - Goal / Non-Goals → reflected in skill body's `## 触发条件`, `## 约束`, and Phase headings.
  - Trigger (explicit only) → Task 1 file body, `## 触发条件`.
  - Phase 1 / 2 / 3 / 4 / 5 → all present in Task 1 file body.
  - Article template → present in Phase 5 of skill body.
  - Failure matrix (7 rows) → present in skill body's `### Phase 5 失败处理` table; verified in Task 3 Step 1.
  - Output examples (A, B, C) → present in skill body; verified in Task 3 Step 1.
  - Installation: symlink from `~/.claude/skills/query-knowledge/SKILL.md` → Task 2.
  - Constraints preserved (Layer 1, wiki-link convention, language) → present in skill body's `## 约束`.
- **Placeholders**: every `<placeholder>` in the skill body is a runtime substitution token (slug, title, question, stderr) — these are correct, not failures. Searched plan for `TBD` / `TODO` — none.
- **Type/name consistency**: `<slug>` consistently kebab-case; branch prefix `article/` consistent across Phase 5 steps and failure table; commit-message and PR-body templates align (same `Question` / `New:` / `Updated:` keys).
- **No tests**: this is a doc-only skill, just like `ingest-knowledge`. Verification is reading + env check (Task 3).
