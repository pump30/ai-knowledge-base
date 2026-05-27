# Query-Knowledge Skill Design

**Date**: 2026-05-27
**Status**: Approved
**Affects**: New file `.claude/skills/query-knowledge.md` (symlinked to `~/.claude/skills/query-knowledge/SKILL.md`)

## Goal

Create a skill that answers user questions by consulting the VKI knowledge base first, supplementing with training knowledge only when needed, and offering to save high-quality answers as `vki/articles/` via the same PR workflow `ingest-knowledge` uses.

## Non-Goals

- No automatic triggering on every user question. The skill is invoked explicitly via the `Skill` tool or `/query-knowledge` command.
- No semantic search / embeddings. Match candidates from `vki/index.md` text and follow `[[wiki-links]]`.
- No caching, indexing, or preprocessing.
- No language restriction on questions.
- No automatic merge of article PRs. The user merges manually.
- No modification to `CLAUDE.md`'s existing Query workflow (the skill formalizes it; CLAUDE.md continues to enforce "read `vki/index.md` first" globally).

## Scope

The skill takes a question (typically passed as args when invoked) and:

1. Consults the VKI knowledge base.
2. Synthesizes an answer.
3. Optionally saves it as an article via PR.

The user merges the article PR manually.

## Trigger

Explicit invocation only:

- `Skill` tool with `skill: "query-knowledge"` and `args: "<question>"`.
- `/query-knowledge <question>` slash command (Claude Code resolves it to the same `Skill` invocation).

The skill does NOT register keyword triggers ("查一下", "什么是…"). Default question handling is governed by `CLAUDE.md`'s top-priority banner ("read `vki/index.md` before answering"), independent of this skill.

## Flow

```
Phase 1 (检索):  read vki/index.md → list ≤5 candidate pages → report to user
Phase 2 (深读):  read candidate pages → follow Sources/Related links one level deep
Phase 3 (合成):  emit answer with two clearly separated sections
Phase 4 (询问保存): ask "save to vki/articles/?" — only if KB had relevant material
Phase 5 (保存为 article, 仅 yes 时): pre-ingest checks → article/<slug> branch → write article → commit → push → gh pr create
```

## Phase 1: Retrieve

1. Run `Read` on `vki/index.md`. If missing or unreadable → stop and report.
2. Scan the index's four sections (Concepts, Entities, Comparisons, Articles) for pages whose Title or Description matches the question's key terms.
3. Pick **at most 5** most relevant candidate page names.
4. Report to the user before moving on:

   ```
   📚 知识库检索结果（找到 N 个候选）：
     - [[page-1]] — <description>
     - [[page-2]] — <description>
     ...
   ```

5. If N == 0 → skip Phase 2 and Phase 4 (skip the save prompt). Go straight to Phase 3 with KB-empty path.

## Phase 2: Deep Read

1. `Read` each candidate page in full.
2. Within each page, look at `## Sources` and `## Related` lists. For each `[[wiki-link]]` referenced, decide whether reading it would materially improve the answer:
   - Read **at most 1 level deep** (no chains of 3+ pages).
   - Stop if the question is already answered by the candidate set.
3. Cap total pages read in this phase at **10** (5 candidates + 5 follow-ups).

## Phase 3: Synthesize

The answer has up to **two sections**, in this exact order:

### 来自知识库 (KB section)
- Main body of the answer.
- **Every concept/entity/comparison referenced must be a `[[wiki-link]]`** to the actual page name (no bare names).
- If multiple pages contributed, weave them coherently — don't dump page contents.

### 来自训练知识（KB 未覆盖部分） (External section)
- Appears **only if the KB content is incomplete** for answering the question.
- Plain prose, **no `[[wiki-links]]`** (because external knowledge isn't in the KB).
- Opening line: `以下内容不在知识库中，来自训练知识：`

### KB-empty case
If Phase 1 returned 0 candidates:

```
ℹ️ 知识库未覆盖此话题。

以下内容不在知识库中，来自训练知识：
<answer>
```

No "save as article" prompt in this case.

## Phase 4: Ask to Save

After emitting the answer, **only when at least one KB page was used**, append exactly:

```
📝 这个回答要不要保存为 vki/articles/<proposed-slug>.md？(yes/no)
```

`<proposed-slug>` is a kebab-case slug derived from the question topic.

If user says yes → Phase 5. Anything else → end.

## Phase 5: Save as Article (only on yes)

Mirrors the git flow in `ingest-knowledge` skill exactly. **Branch prefix differs** — `article/` instead of `ingest/` to keep PR types distinguishable.

### Pre-write checks
1. `git status --short` → if dirty, stop and ask user to commit/stash.
2. `git checkout main`.
3. `git pull origin main` → if conflict/failure, stop and report.
4. `git checkout -b article/YYYY-MM-DD-<slug>` (append `-2`, `-3`, … if name taken).

### Write the article
File path: `vki/articles/<slug>.md`. Template:

```markdown
# <Concise Title (derived from question)>

## Question
<verbatim user question>

## Answer
<the answer body — same content emitted in Phase 3, KB section first with [[wiki-links]], external section second if used>

## Sources Consulted
- [[page-name-1]]
- [[page-name-2]]
...

## External Knowledge Used
<one or two sentences describing which parts came from training knowledge; omit this section entirely if the answer was 100% KB-derived>

## Date
YYYY-MM-DD
```

### Update index
In `vki/index.md` `## Articles` table, add a row:

```
| [[<slug>]] | <one-line description> | YYYY-MM-DD |
```

If the table currently shows the placeholder `(暂无 - 等待 Query 产出)`, replace that row with the new article row.

### Commit + push + PR
1. Stage **only** the article file and `vki/index.md`. Never `git add -A`.
2. Commit with HEREDOC:

   ```
   Article: <Title>

   Question: <verbatim question>
   - New: vki/articles/<slug>.md
   - Updated: vki/index.md
   ```

3. `git push -u origin article/YYYY-MM-DD-<slug>`.
4. `gh pr create --base main --head <branch> --title "Article: <Title>" --body "<...>"`.

PR body:

```markdown
## Question
<verbatim>

## Answer summary
<2-3 sentence summary of the answer>

## Sources consulted
- [[page-1]]
- [[page-2]]

## VKI changes
- New: `vki/articles/<slug>.md`
- Updated: `vki/index.md` (added 1 article row)
```

5. Output the PR URL. Skill never merges.

## Phase 5 failure matrix

| Stage | Condition | Action |
|-------|-----------|--------|
| Pre-write | Working tree dirty | Stop; list dirty files; ask user to commit/stash |
| Pre-write | `git pull` conflict/failure | Stop; output stderr; do not auto-resolve |
| Pre-write | Branch name already exists | Append `-2`, `-3`, … until unique |
| Pre-write | Currently on existing `article/*` or `ingest/*` branch | Checkout main first |
| Save | Nothing staged after `git add` | Skip commit and PR; report "no article file written" |
| Save | `git push` fails | Keep local commit; report stderr |
| Save | `gh pr create` fails | Keep remote branch; output compare URL for manual PR |

## Output Examples

### Example A — KB has full coverage
User invokes with `args: "什么是 RAG？"`.

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

### Example B — KB partial, external supplements
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

### Example C — KB empty
```
ℹ️ 知识库未覆盖此话题。

以下内容不在知识库中，来自训练知识：
<answer>
```

(No save prompt.)

## Constraints Preserved from Existing Skills

- Article files created via this skill should not later be modified arbitrarily; if substantial new info comes in, ingest a new source instead.
- VKI conventions remain: Chinese-preferred prose, English technical terms kept as-is, all internal references via `[[page-name]]`.
- Sources files are write-once (Layer 1 rule unchanged).

## Out of Scope (YAGNI)

- Embedding/vector search.
- Chat history reuse across sessions.
- Multi-question batches.
- Auto-classifying which question deserves an article (always ask).
- Editing or deleting existing articles.
- Concurrent article PRs (one per invocation).

## Installation

After implementation:

1. File lives at `.claude/skills/query-knowledge.md` (git tracked, alongside `ingest-knowledge.md`).
2. Symlink: `~/.claude/skills/query-knowledge/SKILL.md` → `<repo>/.claude/skills/query-knowledge.md` (mirrors the `ingest-knowledge` setup so any `Skill` tool consumer can discover it).
