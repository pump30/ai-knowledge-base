# Ingest-Knowledge Skill: Pre-Ingest Pull + PR Flow

**Date**: 2026-05-26
**Status**: Approved
**Affects**: `.claude/skills/ingest-knowledge.md`

## Goal

Extend the existing `ingest-knowledge` skill so every ingestion:

1. Starts from an up-to-date `main` branch.
2. Lands on its own ingestion branch.
3. Opens a Pull Request via `gh` CLI for the user to review and merge.

The user merges PRs manually. The skill never merges.

## Non-Goals

- No auto-merge, no auto-rebase on conflict, no tags, no changelog.
- No GPG signing changes.
- No modification to ingestion logic itself (Phase 1–4 stay as is).
- No changes to `CLAUDE.md` high-level workflow — this is an internal detail of the skill.

## Flow Overview

```
Pre-ingest:  git status clean → checkout main → pull → new branch ingest/YYYY-MM-DD-<slug>
Phase 1–4:   (unchanged) fetch → clean → write sources/ → update vki/
Phase 6:     stage specific files → commit → push branch → gh pr create
Phase 5:     report (now includes branch + PR URL)
```

Phase 6 runs before Phase 5 so the report can include the PR URL.

## Pre-Ingest (new, before Phase 1)

Inserted at the very top of the skill execution.

### Steps

1. **Clean workspace check**: run `git status --short`.
   - If output is non-empty → **stop** and tell user: "Working tree has uncommitted changes. Please commit or stash before ingesting." List the dirty files.
2. **Switch to main**: `git checkout main`.
3. **Pull latest**: `git pull origin main`.
   - If pull reports conflicts or fails → **stop** and report. Do not attempt to resolve.
4. **Create ingest branch**: `git checkout -b ingest/YYYY-MM-DD-<slug>`.
   - `YYYY-MM-DD` = today's date.
   - `<slug>` = the same kebab-case slug used in the source filename (derived from the content's main topic).
   - If branch already exists → append `-2`, `-3`, etc. until a unique name is found.

### Branch naming examples

- `ingest/2026-05-26-rag-evaluation-best-practices`
- `ingest/2026-05-26-langgraph-streaming-2` (if `-langgraph-streaming` already exists)

## Phase 1–4 (unchanged)

Existing logic for source detection, extraction, cleaning, structuring, writing `sources/`, and updating the VKI network runs unchanged.

## Phase 6: Open PR (new, before Phase 5 report)

### Steps

1. **Stage specific files only**: `git add` each file written or modified during this ingestion. Never use `git add -A` or `git add .`. The skill knows the exact paths because it just wrote them.
2. **Commit** with HEREDOC:

   ```
   Ingest: <source title>

   Source: <URL or local path>
   - New: vki/concepts/xxx.md, vki/entities/yyy.md
   - Updated: vki/concepts/zzz.md
   ```

   - First line ≤ 70 chars. Truncate `<source title>` if needed.
   - Omit `New:` or `Updated:` lines that are empty.
3. **Push branch**: `git push -u origin <branch-name>`.
4. **Create PR**:

   ```
   gh pr create --base main --head <branch-name> \
     --title "Ingest: <source title>" \
     --body "$(cat <<'EOF'
   Source: <URL or local path>

   ## VKI changes
   - New: vki/concepts/xxx.md, vki/entities/yyy.md
   - Updated: vki/concepts/zzz.md

   ## Notes
   <observations from the ingestion report, if any>
   EOF
   )"
   ```

5. Capture the PR URL from `gh pr create` stdout.

## Phase 5: Report (extended)

The existing report appends two lines:

```
✓ 已推送分支: ingest/2026-05-26-xxx
✓ PR 已创建: https://github.com/pump30/ai-knowledge-base/pull/N
```

On push failure:

```
✗ Push 失败: <stderr>。本地分支 ingest/... 和 commit <hash> 已保留。
```

On PR-create failure (push succeeded):

```
✓ 分支已推送: ingest/...
✗ gh pr create 失败: <stderr>。请手动开 PR: https://github.com/pump30/ai-knowledge-base/compare/main...ingest/...
```

## Failure Matrix

| Stage | Condition | Action |
|-------|-----------|--------|
| Pre-ingest | Working tree dirty | Stop, list dirty files, ask user to commit/stash |
| Pre-ingest | `git pull` conflict or failure | Stop, output stderr, do not auto-resolve |
| Pre-ingest | Branch name already taken | Append `-2`, `-3`, … |
| Pre-ingest | Currently on existing `ingest/*` branch | Checkout main first (existing branch left intact) |
| Phase 6 | Nothing staged after `git add` | Skip commit and PR, report "no new content to publish" |
| Phase 6 | `git push` fails (auth, network) | Keep local commit, report stderr |
| Phase 6 | `gh pr create` fails | Keep branch on remote, output compare URL |

## Constraints Preserved from Existing Skill

- Sources files are still write-once (Layer 1 rule).
- VKI pages still follow CLAUDE.md templates and `[[wiki-link]]` conventions.
- VKI language: Chinese-preferred, English technical terms kept as-is.

## Out of Scope (YAGNI)

- Auto-rebase / auto-resolve of any conflict.
- PR labels, reviewers, draft mode.
- Multiple ingestions batched into one PR.
- Status checks, CI integration.
- Rolling back a commit on push failure.
