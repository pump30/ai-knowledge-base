# Ingest-Knowledge PR Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Edit `.claude/skills/ingest-knowledge.md` to add a Pre-Ingest stage (clean check + pull main + branch) and a Phase 6 (commit + push + open PR), per the approved spec at `docs/superpowers/specs/2026-05-26-ingest-knowledge-pr-flow-design.md`.

**Architecture:** Pure documentation/skill edit. No code, no tests. The skill file is a Markdown instruction document the LLM follows at runtime. Verification is a manual dry-run after the edits.

**Tech Stack:** Markdown, Bash (`git`, `gh` CLI). Target file: `.claude/skills/ingest-knowledge.md`.

---

## File Structure

- **Modify**: `.claude/skills/ingest-knowledge.md` — insert Pre-Ingest section before existing Phase 1, insert Phase 6 between existing Phase 4 and Phase 5, extend Phase 5 report template, and extend the error-handling table.

No new files. No deletes.

---

### Task 1: Insert Pre-Ingest section

**Files:**
- Modify: `.claude/skills/ingest-knowledge.md` — insert a new `## Pre-Ingest` section between the `## 触发条件` section and the `## 执行流程` heading.

- [ ] **Step 1: Read the file to locate the insertion point**

Run: `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/ingest-knowledge.md`.

Expected: confirm the file ends `## 触发条件 ... .epub）` block at lines ~10–15 and `## 执行流程` heading at line ~17.

- [ ] **Step 2: Insert the Pre-Ingest section**

Use `Edit` tool. `old_string` is the existing line `## 执行流程` (must match exactly with surrounding context to be unique). Replace with the new Pre-Ingest section followed by `## 执行流程`.

`old_string`:
```
- 用户给了一个本地文件路径（.pdf、.md、.txt、.epub）

## 执行流程
```

`new_string`:
```
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
```

- [ ] **Step 3: Commit**

```bash
git add .claude/skills/ingest-knowledge.md
git commit -m "Add Pre-Ingest stage to ingest-knowledge skill"
```

---

### Task 2: Insert Phase 6 between Phase 4 and Phase 5

**Files:**
- Modify: `.claude/skills/ingest-knowledge.md` — insert a new `### Phase 6: Open PR` section before the existing `### Phase 5: 输出报告` heading.

- [ ] **Step 1: Insert Phase 6 ahead of Phase 5**

Use `Edit` tool. Match the exact line `### Phase 5: 输出报告` and replace with the new Phase 6 followed by Phase 5.

`old_string`:
```
5. **交叉引用**：确保新页面与已有页面用 `[[page-name]]` 互相链接

### Phase 5: 输出报告
```

`new_string`:
```
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/ingest-knowledge.md
git commit -m "Add Phase 6: Open PR to ingest-knowledge skill"
```

---

### Task 3: Extend Phase 5 report template

**Files:**
- Modify: `.claude/skills/ingest-knowledge.md` — replace the existing Phase 5 report block with one that includes branch and PR URL output, plus failure variants.

- [ ] **Step 1: Replace the existing report code block**

Use `Edit` tool. Match the existing Phase 5 report block exactly.

`old_string`:
```
完成后输出摘要：

```
✓ 摄入完成: [标题]
  - 来源: [URL/路径]
  - 新建: concept/xxx, entity/yyy
  - 更新: concept/zzz（新增了关于...的信息）
  - 发现关联: [[a]] ↔ [[b]]
  - 观察: "某概念" 出现1次，暂不创建页面
```
```

`new_string`:
```
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
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/ingest-knowledge.md
git commit -m "Extend Phase 5 report with branch/PR/error variants"
```

---

### Task 4: Extend the existing error-handling table

**Files:**
- Modify: `.claude/skills/ingest-knowledge.md` — append rows to the existing `## 错误处理` table for Pre-Ingest and Phase 6 cases.

- [ ] **Step 1: Append rows to the error table**

Use `Edit` tool. Match the last existing row + the closing `## 约束` heading to keep the match unique.

`old_string`:
```
| 内容与 AI/LLM 主题无关 | 提醒用户这是 AI 知识库，询问是否仍要摄入 |

## 约束
```

`new_string`:
```
| 内容与 AI/LLM 主题无关 | 提醒用户这是 AI 知识库，询问是否仍要摄入 |
| Pre-Ingest 工作区脏 | 停下，列出 dirty 文件，提示先 commit 或 stash |
| Pre-Ingest `git pull` 冲突或失败 | 停下，输出 stderr，**不要**自动 resolve 或 rebase |
| Pre-Ingest 分支名已占用 | 依次尝试 `-2`、`-3`、…，直到唯一 |
| Phase 6 无 staged 改动 | 跳过 commit 和 PR；Phase 5 写"无新内容可发布" |
| Phase 6 `git push` 失败 | 保留本地 commit；Phase 5 写 push 失败信息 |
| Phase 6 `gh pr create` 失败 | 远端分支保留；Phase 5 输出 compare URL 让用户手动开 PR |

## 约束
```

- [ ] **Step 2: Commit**

```bash
git add .claude/skills/ingest-knowledge.md
git commit -m "Extend error table with Pre-Ingest and Phase 6 cases"
```

---

### Task 5: Manual dry-run verification

**Files:**
- Read-only check on `.claude/skills/ingest-knowledge.md`.

- [ ] **Step 1: Read the full file end-to-end**

Run: `Read` tool on `/Users/I572881/workspace/ai-knowledge-base/.claude/skills/ingest-knowledge.md`.

Verify in order:

1. The `## Pre-Ingest` section appears between `## 触发条件` and `## 执行流程`.
2. `### Phase 6: Open PR` appears after `### Phase 4` and before `### Phase 5: 输出报告`.
3. The Phase 5 report block contains the new `已推送分支` / `PR 已创建` lines and the three failure variants.
4. The `## 错误处理` table has 6 new rows at the bottom (3 Pre-Ingest + 3 Phase 6).
5. No leftover spec markers, no `TBD`, no duplicate sections.

- [ ] **Step 2: Verify environment prerequisites**

Run:
```bash
git remote -v
gh auth status
```

Expected:
- `origin` points to `https://github.com/pump30/ai-knowledge-base.git`.
- `gh auth status` shows logged in to `github.com` with `repo` scope.

If either check fails, do not claim the skill is ready — surface the gap to the user.

- [ ] **Step 3: Final commit (only if Steps 1–2 surfaced no fixes)**

If no fixes needed, no commit. If fixes were made, commit them with a focused message (e.g. `Fix typo in ingest-knowledge Phase 6`).

---

## Self-Review Notes

- **Spec coverage**: every spec section maps to a task. Pre-Ingest steps → Task 1. Phase 6 steps + failure matrix Phase-6 rows → Tasks 2 & 4. Phase 5 report extensions → Task 3. Pre-Ingest failure rows → Task 4. Constraints preserved (write-once sources, Chinese language) require no edit since the existing skill already documents them.
- **Placeholders**: every code/text block above is concrete and ready to paste.
- **Type/name consistency**: branch naming (`ingest/YYYY-MM-DD-<slug>`), heading names (`Pre-Ingest`, `Phase 6: Open PR`, `Phase 5: 输出报告`), and command syntax (`gh pr create --base main --head ...`) are identical across tasks.
- **No tests**: this is a skill-doc edit; verification is manual reading + env check (Task 5).
