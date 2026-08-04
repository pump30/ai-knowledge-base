# Night Shift — Day/Night Task Orchestration Agent

**Status**: draft (awaiting user review)
**Date**: 2026-06-04
**Owner**: I572881

---

## 1. Problem Statement

User has many work tasks across multiple projects (UI5 apps, MTA, AI tools, infra). Some tasks need interactive judgement during the day (review UI screenshots, decide direction, read mail, do SAP-internal manual steps); others are purely mechanical / well-specified and can be done unattended (refactors, test runs, doc generation, batch fixes).

Today the user does both during the day, which wastes daytime on tasks that don't need a human and also under-uses idle hours overnight.

**Goal**: a personal task orchestration system where the user spends the day **planning and reviewing** with Claude Code, and the system spends the **night executing** specced-out tasks autonomously, delivering whole-topic results the next morning.

---

## 2. Core Principle: Shifts ("Day" vs "Night")

The whole system is organised around the metaphor of a **work-shift roster**.

| | **Day shift** | **Night shift** |
|---|---|---|
| Who works | User + Claude Code interactive sessions | Orchestrator-Claude (long-running) + worker subprocesses |
| What runs | brainstorm / edit spec / review last night's results / merge or discard worktrees / answer agent's open questions | code edits, tests, doc gen, refactors, research — anything fully specifiable |
| Triggers | user-initiated (skill, kanban click) | cron at start of night window; only if there are `shift=night, status=ready` topics |
| LLM allowed? | yes — by user | yes — by orchestrator |
| Daemon allowed to call LLM during day? | **NO** | n/a |

**Hard constraint**: during the day window, the daemon serves only the kanban and accepts skill writes. It does **not** call any LLM, **not** spawn any worker, **not** modify any worktree. All autonomous execution happens strictly in the night window.

### 2.1 Shift attribute on every task

Each task carries `shift: day | night | either` in its spec frontmatter. The plan-night-task skill asks during brainstorming:

> "Can this be done fully unattended? If you can write success criteria + boundaries + rollback now, it's `night`. If you need to look at it / decide mid-flight / answer a question, it's `day`. Default when unsure: `day`."

`either` means user is OK either way; scheduler treats it as night by default to free up the day.

### 2.2 Night window

- Default: **22:00–07:00 local time**.
- Configurable in `config.json` (`shift.night.start`, `shift.night.end`).
- Cron at `start` triggers orchestrator if any night-shift topics are ready.
- At `end - 30min`, orchestrator gets soft signal: stop spawning new workers, finish current, persist state.
- Anything not delivered by end → status `carryover`, prioritised next night.

---

## 3. Architecture (chosen: Orchestrator-as-Claude)

```
                  Day                                         Night
  ┌──────────────────────────────────┐         ┌──────────────────────────────────┐
  │ User in Claude Code              │         │ cron 22:00                       │
  │   /plan-night-task skill         │         │   ↓                              │
  │     ↓ brainstorm                 │         │ ns-daemon checks state.db        │
  │     ↓ writes spec.md             │         │   ↓ has night-shift ready?       │
  │     ↓ status=ready                │         │   ↓ yes                          │
  │                                  │         │ spawn orchestrator-claude        │
  │ Kanban (read-only mostly)        │         │   (long-running `claude -p`,     │
  │   review yesterday's runs        │         │    given list of tonight's       │
  │   merge / discard worktrees      │         │    topics + tools)               │
  │   re-queue failed                │         │   ↓                              │
  │                                  │         │ For each topic:                  │
  │ ns-daemon serves kanban          │         │   orchestrator decides plan,     │
  │   NO LLM calls                   │         │   spawns worker-claude in        │
  │   NO worker spawns                │         │   git worktree, reads logs,     │
  │   NO worktree mutations          │         │   re-spawns until done or       │
  │                                  │         │   budget exhausted                │
  └──────────────────────────────────┘         └──────────────────────────────────┘
```

### 3.1 Orchestrator-Claude

A long-running `claude -p --dangerously-skip-permissions` invocation, given:

- System prompt: "You are the night-shift orchestrator. You will be given a list of topics with specs. For each topic, decide a plan, spawn workers, read their results, decide what to do next. Use the deliver_topic tool when done. Respect budgets. Never ask the user."
- **Tools** (custom CLI scripts, exposed via Claude Code's built-in shell):
  - `ns spawn-worker <topic-id> "<instructions>"` → spawns worker-claude in topic's worktree, returns run id, blocks until exit
  - `ns read-run <run-id>` → returns last N lines of log + git diff stat
  - `ns read-spec <topic-id>` → returns spec.md
  - `ns update-status <topic-id> <status>` → SQLite state change
  - `ns deliver <topic-id> <commit|pr|diff-only>` → finalise: commit / open PR / leave diff
  - `ns budget <topic-id>` → returns remaining tokens / runs
  - `ns next-topic` → returns next ready topic (orchestrator chooses among ready ones)

### 3.2 Worker-Claude

Per-spawn one-shot `claude -p --dangerously-skip-permissions`, started inside the topic's git worktree, given the instruction string from orchestrator + the spec.md. Runs to completion, exits, log captured.

### 3.3 Watchdog

Separate Node process. Heartbeat: orchestrator must call `ns heartbeat` every N minutes. If silent, watchdog SIGTERMs orchestrator and marks affected topic `failed: heartbeat-lost`. Also enforces hard token budget per topic.

### 3.4 ns-daemon (web)

- Day mode: Express server on `localhost:3737`, kanban + SSE event stream, accepts skill writes via REST. **No LLM calls.**
- Night mode: same process, additionally cron-triggers orchestrator at window start.

---

## 4. Data Model

### 4.1 Markdown (human-readable, git-tracked)

Per-topic dir `data/topics/<id>/`:

```
spec.md            — what the user wants (skill writes; user can hand-edit)
orchestrator.md    — orchestrator's working memory across nights (orchestrator writes)
runs/
  run-001.log      — full transcript of a worker (or orchestrator) run
  run-001.diff     — git diff snapshot at end of run
  run-001.meta.json — exit code, tokens, started/ended, summary
```

`spec.md` frontmatter (canonical fields):

```yaml
---
id: 2026-06-04-give-night-shift-rbac
title: 给 night-shift 项目本身加 RBAC
shift: night                # day | night | either
status: ready               # draft | ready | running | review | done | failed | carryover
project_path: /Users/I572881/workspace/night-shift
priority: normal            # high | normal | low
budget_tokens: 200000
budget_runs: 8
deliver_as: pr              # commit | pr | diff-only
created: 2026-06-04T11:30
boundaries:
  - 只改 src/ 和 test/
  - 不能 git push
  - 不能改 schema
success_criteria:
  - npm test 全过
  - npm run lint 干净
  - 至少一个 commit
---

## 背景
...

## 目标
...

## 已知风险
...
```

### 4.2 SQLite (machine-written, kanban-queried, not git-tracked)

`data/state.db`:

```sql
CREATE TABLE topics (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  shift TEXT NOT NULL,                  -- day | night | either
  status TEXT NOT NULL,                 -- draft | ready | running | review | done | failed | carryover
  project_path TEXT,
  priority TEXT DEFAULT 'normal',
  budget_tokens INTEGER,
  budget_runs INTEGER,
  tokens_used INTEGER DEFAULT 0,
  runs_count INTEGER DEFAULT 0,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  delivered_at INTEGER,
  worktree_path TEXT
);

CREATE TABLE runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  topic_id TEXT NOT NULL REFERENCES topics(id),
  run_index INTEGER NOT NULL,
  kind TEXT NOT NULL,                   -- orchestrator | worker
  started_at INTEGER NOT NULL,
  ended_at INTEGER,
  exit_code INTEGER,
  tokens_used INTEGER,
  log_path TEXT,
  diff_path TEXT,
  summary TEXT
);

CREATE TABLE events (                   -- SSE source
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  topic_id TEXT,
  ts INTEGER NOT NULL,
  type TEXT NOT NULL,
  data TEXT                              -- JSON
);

CREATE TABLE shift_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT NOT NULL,
  started_at INTEGER,
  ended_at INTEGER,
  topics_attempted INTEGER,
  topics_delivered INTEGER,
  topics_carryover INTEGER,
  total_tokens INTEGER
);
```

SQLite is the source of truth for status; spec.md frontmatter `status:` is mirrored on every change for git-history readability.

---

## 5. Day-Side Interaction: the `plan-night-task` Skill

A Claude Code skill the user invokes by typing `/plan-night-task` (or auto-triggered when user says "记一下我要做 X").

Skill workflow:

1. **Capture** the rough idea from the user (one sentence is fine).
2. **Brainstorm**: drives a conversation to fill in:
   - title, project_path
   - success criteria (concrete, observable)
   - boundaries (what's off-limits)
   - rollback condition (when should the worker abort?)
   - estimated budget (tokens / max worker spawns)
3. **Shift decision**: at the end, asks the canonical question
   *"Can this run fully unattended? If yes → night. If you need to look mid-flight → day. Default: day."*
4. **Write** `data/topics/<slug>/spec.md` with frontmatter + body.
5. **Insert** SQLite row with `status=ready`.
6. **Confirm**: shows the user the kanban URL + topic id.

The skill lives in `night-shift/skill/plan-night-task/`, symlinked into `~/.claude/skills/`.

---

## 6. Night-Side Execution Loop

### 6.1 Cron entry

```
0 22 * * *  cd ~/workspace/night-shift && node src/start-night.js
```

`start-night.js`:
1. Query SQLite: list topics with `shift IN (night, either) AND status = ready`.
2. If empty: log "nothing tonight", exit.
3. Insert `shift_log` row.
4. Spawn orchestrator-claude with the list of topic ids and a system prompt.
5. Start watchdog.
6. Wait for orchestrator to exit OR window-end signal.
7. On exit: close shift_log row, mark non-delivered topics `carryover`.

### 6.2 Orchestrator loop (its own decision, not coded)

Orchestrator is given system prompt + topic list, then it autonomously calls tools. Pseudo-flow it tends to follow:

```
while topics remain and budget allows:
  pick next topic (own discretion: priority, ease, dependencies)
  read spec
  decide plan (1 worker or multiple sequential workers)
  loop:
    spawn worker with instructions
    read run log + diff
    decide: deliver / spawn another worker / abort / re-plan
  call deliver_topic
```

### 6.3 Worker invocation

```bash
cd <worktree_path>
claude -p --dangerously-skip-permissions \
       --max-turns 50 \
       --output-format stream-json \
       "<orchestrator's instruction>

The full topic spec is in spec.md. Respect boundaries listed there. When done, exit; do not ask questions."
```

stdout/stderr → `runs/run-NNN.log`, `git diff` after exit → `runs/run-NNN.diff`.

### 6.4 Worktree lifecycle

- Created on first worker spawn for a topic: `git worktree add .worktrees/<topic-id> -b nightshift/<topic-id>`
- Reused for subsequent worker runs of the same topic.
- On `deliver`:
  - `commit`: commit pending changes, leave worktree (user merges manually next day).
  - `pr`: commit, push branch, open PR via `gh`.
  - `diff-only`: leave uncommitted, save diff to `delivered.diff`, user reviews and decides.
- Worktree never auto-removed; user prunes during day-shift review.

---

## 7. Kanban UI

Single-page Express app (`web/server.js` + `web/public/index.html`), no build step.

### 7.1 Layout

Two-column, dark theme:

- **Left: Day Shift** — topics where the user has work to do today. Includes:
  - `status=review` (last night's deliverables waiting for user)
  - `status=ready, shift=day` (need user to do them)
  - `status=failed` (need user to look)
  - `status=draft` (skill capture in progress)
- **Right: Night Shift** — topics tagged for night, by status:
  - `status=running` (currently being worked on; only during night window)
  - `status=ready` (queued for next night)
  - `status=carryover` (deferred from previous night)
  - `status=done` (recent deliveries; auto-archive after 7 days)

### 7.2 Per-card actions

- Click → drawer shows spec.md, orchestrator.md, runs list, latest diff.
- "Approve & merge" → merges worktree, deletes branch, status=done.
- "Discard" → removes worktree, status=done with note.
- "Re-queue" → resets status=ready (for failed/carryover).
- "Toggle shift" → flip day↔night.
- "Edit spec" → opens spec.md in $EDITOR (mac `open -a` fallback).

### 7.3 Real-time

SSE stream from `events` table. During night, kanban shows live worker spawn / log lines.

---

## 8. Repository Layout

```
~/workspace/night-shift/
├── README.md
├── CLAUDE.md
├── package.json
├── config.json.example
├── data/
│   ├── topics/<id>/
│   │   ├── spec.md
│   │   ├── orchestrator.md
│   │   └── runs/run-NNN.{log,diff,meta.json}
│   ├── .worktrees/<id>/        (gitignored)
│   └── state.db                 (gitignored)
├── src/
│   ├── start-night.js           # cron entry
│   ├── orchestrator.js          # spawn long-running claude
│   ├── worker.js                # spawn one-shot claude in worktree
│   ├── watchdog.js              # heartbeat + budget guard
│   ├── ns-cli.js                # the `ns` CLI used as orchestrator's tools
│   ├── tools/
│   │   ├── spawn-worker.js
│   │   ├── read-run.js
│   │   ├── update-status.js
│   │   ├── deliver.js
│   │   └── budget.js
│   ├── db.js                    # better-sqlite3 wrapper
│   └── worktree.js              # git worktree helpers
├── web/
│   ├── server.js                # Express + SSE, port 3737
│   └── public/index.html
├── skill/
│   └── plan-night-task/SKILL.md
└── docs/
    └── design.md                (this file, copied in)
```

---

## 9. Error Handling

- Worker exits non-zero → orchestrator decides (re-plan, abort topic, etc.)
- Orchestrator silent > heartbeat threshold → watchdog kills, topic → `failed: heartbeat-lost`
- Token budget exceeded for a topic → orchestrator forced to `deliver` with whatever's done, status → `review` with note
- Window-end reached mid-topic → orchestrator finishes current worker, marks topic `carryover`
- Skill write fails → spec stays in `draft`, no SQLite row, kanban shows nothing (skill prompts user to retry)
- Daemon crash → systemd / launchd auto-restart; in-flight orchestrator is independent process and continues (it has its own watchdog)

---

## 10. Testing Strategy

- Unit tests on db, worktree, ns-cli (jest, like dev-team)
- Integration test of skill → SQLite + spec write
- Integration test of full night-loop with a stubbed `claude` binary that just echoes
- Manual: dry-run mode (`NS_DRY_RUN=1`) where orchestrator and workers are replaced with deterministic stubs, used to validate kanban, status transitions, watchdog, window-end behaviour without burning tokens

---

## 11. Out of Scope (v1)

- Multi-user
- Remote daemon (everything runs on user's Mac)
- Mobile UI
- Slack / Teams / email task ingestion
- Auto-import from Jira / GitHub
- Cross-topic coordination (each topic is independent)
- Cost forecasting
- Encrypted secrets management — relies on existing config.json pattern from dev-team

---

## 12. Open Questions / Risks

- Claude Code CLI's `--dangerously-skip-permissions` behaviour when running headless inside cron — needs validation before v1 is real.
- Token usage of long-running orchestrator-claude could be heavy; per-topic and per-night budgets are the primary mitigation, but monitoring is needed.
- Concurrent worker spawns vs. single-stream — v1 will stay single-stream (one worker at a time per orchestrator) to keep semantics simple. Parallel-per-topic can come later.

---

## 13. Build Order (rough, for the implementation plan)

1. Repo scaffold + `package.json` + lint/test config + CLAUDE.md
2. SQLite schema + db.js + worktree.js
3. `ns` CLI (the tools orchestrator will use), with stub claude
4. ns-daemon (Express, kanban skeleton, SSE)
5. plan-night-task skill (writes spec + SQLite row)
6. Worker invocation against real `claude` CLI, manually triggered
7. Orchestrator entry, manually triggered
8. Watchdog
9. Cron wiring, window-end handling, carryover
10. Kanban interactive actions (approve / discard / re-queue / edit / toggle)
11. Dry-run mode + integration tests
12. Polish, README, screenshots
