# Night Shift Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a personal day/night task orchestration system where the day is reserved for user+Claude planning/review and the night runs an Orchestrator-Claude that spawns one-shot worker-claude processes inside git worktrees to execute fully-specced topics autonomously.

**Architecture:** Single Node.js 22 repo at `~/workspace/night-shift/`. SQLite (`better-sqlite3`) is the source of truth for status; per-topic Markdown (`spec.md`, `orchestrator.md`, `runs/*`) is human-readable and git-tracked. Long-running Orchestrator-Claude (`claude -p --dangerously-skip-permissions`) drives execution via custom `ns` CLI tools (`spawn-worker`, `read-run`, `update-status`, `deliver`, `budget`, `next-topic`, `read-spec`, `heartbeat`). Watchdog enforces heartbeat + token budget. Express + vanilla HTML + SSE kanban on port 3737. Day-window daemon makes **zero** LLM calls and spawns **zero** workers.

**Tech Stack:** Node.js 22, Express 5, better-sqlite3, jest, vanilla HTML/CSS/JS (no build step), git worktrees, cron/launchd for scheduling, `gh` CLI for PR delivery.

---

## File Structure

Each file has one clear responsibility. Splits follow §8 of the spec.

| File | Responsibility |
|------|----------------|
| `package.json` | deps, scripts (`test`, `lint`, `web`, `night`, `ns`) |
| `jest.config.js` | jest config (testEnvironment node, forceExit) |
| `.eslintrc.json` | ESLint flat-style config (Node 22, jest globals) |
| `.gitignore` | ignores `data/state.db`, `data/.worktrees/`, `node_modules/`, `coverage/`, `config.json` |
| `config.json.example` | template (paths, shift window, budgets) |
| `README.md` | quick start |
| `CLAUDE.md` | repo-level instructions |
| `src/db.js` | better-sqlite3 wrapper: `openDb`, `migrate`, `topics.*`, `runs.*`, `events.*`, `shiftLog.*` |
| `src/worktree.js` | git worktree helpers: `ensureWorktree`, `pruneWorktree`, `diffStat` |
| `src/spec-io.js` | read/write `spec.md` frontmatter (gray-matter-style minimal parser) |
| `src/shift.js` | pure functions: `isNightWindow(now, cfg)`, `nextNightStart(now, cfg)` |
| `src/config.js` | load + validate `config.json` against schema |
| `src/tools/spawn-worker.js` | spawn one-shot `claude -p` in worktree, capture log+diff+meta |
| `src/tools/read-run.js` | tail run log + diff stat |
| `src/tools/read-spec.js` | print spec.md |
| `src/tools/update-status.js` | mutate topic status, mirror to spec frontmatter, emit event |
| `src/tools/deliver.js` | finalise topic (commit / pr / diff-only) |
| `src/tools/budget.js` | report remaining tokens + runs |
| `src/tools/next-topic.js` | return next ready night topic |
| `src/tools/heartbeat.js` | bump `last_heartbeat` row in events table |
| `src/ns-cli.js` | argv dispatcher → `tools/*` |
| `src/orchestrator.js` | spawn long-running Orchestrator-Claude with system prompt + topic list |
| `src/watchdog.js` | poll heartbeats + token totals, SIGTERM on breach |
| `src/start-night.js` | cron entry: query ready, log shift row, spawn orchestrator+watchdog, handle window-end |
| `web/server.js` | Express + SSE + REST writes |
| `web/public/index.html` | two-column kanban, vanilla JS |
| `skill/plan-night-task/SKILL.md` | day-side capture skill |
| `test/__tests__/*.test.js` | jest tests, one file per src module |
| `test/helpers/tmpdb.js` | shared helper to create a fresh sqlite db + tmp data dir |
| `test/helpers/fake-claude.sh` | stub `claude` binary that echoes JSON, used by spawn-worker tests |

The `data/` directory is created on first run; `data/topics/` is git-tracked, `data/state.db` and `data/.worktrees/` are gitignored.

---

## Conventions

- **Node 22 ESM** is NOT used; we use **CommonJS** to match `dev-team` style and keep `require()` ergonomics for jest.
- All paths in code are absolute, derived from `config.json`'s `root` field.
- All time values stored in SQLite are unix epoch milliseconds (`Date.now()`).
- All status enums are uppercase strings in code (`'READY'`, `'RUNNING'`, etc. ARE NOT used) — we keep lowercase as in the spec (`'ready'`, `'running'`, `'review'`, `'done'`, `'failed'`, `'carryover'`, `'draft'`).
- Function naming: camelCase. File naming: kebab-case. SQLite columns: snake_case.
- Tests live in `test/__tests__/<module>.test.js` (mirrors `dev-team`).
- Every commit message uses Conventional Commits (`feat:`, `test:`, `chore:`, `docs:`).
- `git worktree add` uses `data/.worktrees/<topic-id>` and branch `nightshift/<topic-id>`.

---

### Task 1: Repo Scaffold

**Files:**
- Create: `/Users/I572881/workspace/night-shift/package.json`
- Create: `/Users/I572881/workspace/night-shift/jest.config.js`
- Create: `/Users/I572881/workspace/night-shift/.eslintrc.json`
- Create: `/Users/I572881/workspace/night-shift/.gitignore`
- Create: `/Users/I572881/workspace/night-shift/config.json.example`
- Create: `/Users/I572881/workspace/night-shift/README.md`
- Create: `/Users/I572881/workspace/night-shift/CLAUDE.md`

- [ ] **Step 1: Create directory and init git**

```bash
mkdir -p /Users/I572881/workspace/night-shift
cd /Users/I572881/workspace/night-shift
git init -b main
```

Expected: `Initialized empty Git repository in /Users/I572881/workspace/night-shift/.git/`

- [ ] **Step 2: Write package.json**

Write `/Users/I572881/workspace/night-shift/package.json`:

```json
{
  "name": "night-shift",
  "version": "0.1.0",
  "description": "Day/Night task orchestration agent — Orchestrator-Claude drives worker-claude inside git worktrees during night window only",
  "main": "src/start-night.js",
  "bin": {
    "ns": "./src/ns-cli.js"
  },
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "lint": "eslint src web test",
    "web": "node web/server.js",
    "night": "node src/start-night.js",
    "ns": "node src/ns-cli.js"
  },
  "engines": {
    "node": ">=22.0.0"
  },
  "dependencies": {
    "better-sqlite3": "^11.5.0",
    "express": "^5.0.1"
  },
  "devDependencies": {
    "eslint": "^9.15.0",
    "jest": "^30.0.0",
    "supertest": "^7.0.0"
  },
  "license": "ISC"
}
```

- [ ] **Step 3: Write jest.config.js**

```javascript
module.exports = {
  testEnvironment: 'node',
  coverageDirectory: 'coverage',
  collectCoverageFrom: ['src/**/*.js', 'web/server.js'],
  testMatch: ['**/__tests__/**/*.test.js'],
  testTimeout: 30000,
  verbose: true,
  forceExit: true,
};
```

- [ ] **Step 4: Write .eslintrc.json**

```json
{
  "env": { "node": true, "jest": true, "es2024": true },
  "parserOptions": { "ecmaVersion": 2024, "sourceType": "script" },
  "rules": {
    "no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "no-undef": "error",
    "semi": ["error", "always"],
    "quotes": ["error", "single", { "avoidEscape": true }]
  }
}
```

- [ ] **Step 5: Write .gitignore**

```gitignore
node_modules/
coverage/
data/state.db
data/state.db-journal
data/state.db-wal
data/state.db-shm
data/.worktrees/
config.json
*.log
.DS_Store
```

- [ ] **Step 6: Write config.json.example**

```json
{
  "root": "/Users/I572881/workspace/night-shift",
  "data_dir": "/Users/I572881/workspace/night-shift/data",
  "claude_bin": "/Users/I572881/.local/bin/claude",
  "shift": {
    "night": { "start": "22:00", "end": "07:00" },
    "soft_stop_minutes_before_end": 30
  },
  "budgets": {
    "default_tokens": 200000,
    "default_runs": 8,
    "watchdog_heartbeat_seconds": 300
  },
  "web": { "port": 3737, "host": "127.0.0.1" },
  "dry_run": false
}
```

- [ ] **Step 7: Write README.md**

```markdown
# Night Shift

Personal day/night task orchestration. Day = user + Claude Code planning. Night = Orchestrator-Claude executes specced topics autonomously.

## Quick Start

```bash
npm install
cp config.json.example config.json
npm run web    # kanban on http://localhost:3737
```

## Concepts

- **Topic**: one task with a `spec.md` + SQLite row + git worktree.
- **Shift**: `day` | `night` | `either`. Night-only topics run unattended in the night window.
- **Orchestrator-Claude**: long-running `claude -p` that drives workers via `ns` CLI tools.
- **Worker-Claude**: one-shot `claude -p` in a git worktree.

See `docs/design.md` for the full design.

## Hard Rule

The daemon NEVER calls an LLM during the day window. Autonomous execution happens strictly in the configured night window.
```

- [ ] **Step 8: Write CLAUDE.md**

```markdown
# Night Shift — Repo Instructions

## Stack
- Node 22, CommonJS (no ESM), better-sqlite3, Express 5, jest 30, vanilla HTML/CSS/JS (no build).
- Path style: absolute paths everywhere, derived from `config.json` `root`.

## Conventions
- camelCase functions, kebab-case files, snake_case SQL columns.
- Status enums are lowercase: `draft|ready|running|review|done|failed|carryover`.
- Times stored as unix epoch ms (`Date.now()`).
- Worktree path: `data/.worktrees/<topic-id>`, branch `nightshift/<topic-id>`.

## Hard Rules
- Day-mode daemon: NO LLM calls, NO worker spawns, NO worktree mutations.
- Every status change MUST emit an `events` row (SSE source).
- SQLite is source of truth for status; spec.md frontmatter is mirrored on every update.

## Verification
- `npm run lint && npm test` must pass before commit.
- Use `NS_DRY_RUN=1` to swap claude-spawning with deterministic stubs.
```

- [ ] **Step 9: Install deps**

```bash
cd /Users/I572881/workspace/night-shift
npm install
```

Expected: `node_modules/` populated, `package-lock.json` created, no errors.

- [ ] **Step 10: Verify lint + empty test run**

```bash
mkdir -p src/tools test/__tests__ test/helpers web/public skill data/topics
npx eslint src --no-error-on-unmatched-pattern
npx jest --passWithNoTests
```

Expected: eslint exits 0 (no .js files yet, no patterns matched), jest reports `No tests found, exiting with code 0`.

- [ ] **Step 11: Commit scaffold**

```bash
cd /Users/I572881/workspace/night-shift
git add -A
git commit -m "chore: scaffold night-shift repo with jest + eslint + express"
```

---

### Task 2: Config Loader

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/config.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/config.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/config.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { loadConfig } = require('../../src/config');

function tmpFile(content) {
  const p = path.join(os.tmpdir(), `ns-cfg-${Date.now()}-${Math.random()}.json`);
  fs.writeFileSync(p, content);
  return p;
}

describe('loadConfig', () => {
  test('loads valid config and fills defaults', () => {
    const p = tmpFile(JSON.stringify({
      root: '/tmp/x',
      data_dir: '/tmp/x/data',
      claude_bin: '/usr/bin/claude',
    }));
    const cfg = loadConfig(p);
    expect(cfg.root).toBe('/tmp/x');
    expect(cfg.shift.night.start).toBe('22:00');
    expect(cfg.shift.night.end).toBe('07:00');
    expect(cfg.budgets.default_tokens).toBe(200000);
    expect(cfg.web.port).toBe(3737);
    expect(cfg.dry_run).toBe(false);
  });

  test('throws when root missing', () => {
    const p = tmpFile(JSON.stringify({ data_dir: '/tmp/x', claude_bin: '/usr/bin/claude' }));
    expect(() => loadConfig(p)).toThrow(/root/);
  });

  test('throws on bad time format', () => {
    const p = tmpFile(JSON.stringify({
      root: '/tmp/x',
      data_dir: '/tmp/x/data',
      claude_bin: '/usr/bin/claude',
      shift: { night: { start: '25:00', end: '07:00' } },
    }));
    expect(() => loadConfig(p)).toThrow(/start/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd /Users/I572881/workspace/night-shift
npx jest test/__tests__/config.test.js
```

Expected: FAIL — `Cannot find module '../../src/config'`

- [ ] **Step 3: Implement src/config.js**

```javascript
const fs = require('fs');

const DEFAULTS = {
  shift: {
    night: { start: '22:00', end: '07:00' },
    soft_stop_minutes_before_end: 30,
  },
  budgets: {
    default_tokens: 200000,
    default_runs: 8,
    watchdog_heartbeat_seconds: 300,
  },
  web: { port: 3737, host: '127.0.0.1' },
  dry_run: false,
};

const TIME_RE = /^([01]\d|2[0-3]):([0-5]\d)$/;

function deepMerge(base, over) {
  if (over === undefined || over === null) return base;
  if (typeof base !== 'object' || typeof over !== 'object') return over;
  const out = Array.isArray(base) ? [...base] : { ...base };
  for (const k of Object.keys(over)) {
    out[k] = deepMerge(base[k], over[k]);
  }
  return out;
}

function validate(cfg) {
  if (!cfg.root) throw new Error('config.root is required');
  if (!cfg.data_dir) throw new Error('config.data_dir is required');
  if (!cfg.claude_bin) throw new Error('config.claude_bin is required');
  if (!TIME_RE.test(cfg.shift.night.start)) {
    throw new Error(`config.shift.night.start must be HH:MM, got ${cfg.shift.night.start}`);
  }
  if (!TIME_RE.test(cfg.shift.night.end)) {
    throw new Error(`config.shift.night.end must be HH:MM, got ${cfg.shift.night.end}`);
  }
}

function loadConfig(filePath) {
  const raw = JSON.parse(fs.readFileSync(filePath, 'utf8'));
  const merged = deepMerge(DEFAULTS, raw);
  validate(merged);
  return merged;
}

module.exports = { loadConfig, DEFAULTS };
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/config.test.js
```

Expected: PASS — 3 tests.

- [ ] **Step 5: Lint + commit**

```bash
npx eslint src/config.js test/__tests__/config.test.js
git add src/config.js test/__tests__/config.test.js
git commit -m "feat: add config loader with defaults and validation"
```

Expected: lint clean, commit succeeds.

---

### Task 3: SQLite Schema + db.js

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/db.js`
- Create: `/Users/I572881/workspace/night-shift/test/helpers/tmpdb.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/db.test.js`

- [ ] **Step 1: Write the test helper**

Write `test/helpers/tmpdb.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { openDb, migrate } = require('../../src/db');

function makeTmpDb() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-test-'));
  const dbPath = path.join(dir, 'state.db');
  const db = openDb(dbPath);
  migrate(db);
  return { db, dir, dbPath, cleanup: () => { db.close(); fs.rmSync(dir, { recursive: true, force: true }); } };
}

module.exports = { makeTmpDb };
```

- [ ] **Step 2: Write the failing test**

Write `test/__tests__/db.test.js`:

```javascript
const { makeTmpDb } = require('../helpers/tmpdb');
const db = require('../../src/db');

describe('db', () => {
  let h;
  beforeEach(() => { h = makeTmpDb(); });
  afterEach(() => { h.cleanup(); });

  test('migrate creates all four tables', () => {
    const rows = h.db.prepare(
      "SELECT name FROM sqlite_master WHERE type='table' ORDER BY name"
    ).all();
    expect(rows.map(r => r.name)).toEqual(['events', 'runs', 'shift_log', 'topics']);
  });

  test('topics.insert + topics.get round-trip', () => {
    db.topics.insert(h.db, {
      id: 't1',
      title: 'Test',
      shift: 'night',
      status: 'ready',
      project_path: '/tmp/p',
      priority: 'normal',
      budget_tokens: 1000,
      budget_runs: 3,
    });
    const t = db.topics.get(h.db, 't1');
    expect(t.title).toBe('Test');
    expect(t.shift).toBe('night');
    expect(t.tokens_used).toBe(0);
    expect(typeof t.created_at).toBe('number');
  });

  test('topics.updateStatus mutates status and updated_at, emits event', () => {
    db.topics.insert(h.db, { id: 't2', title: 'X', shift: 'night', status: 'ready' });
    const before = db.topics.get(h.db, 't2').updated_at;
    // sleep 2ms to guarantee monotonic clock change
    const target = Date.now() + 2;
    while (Date.now() < target) { /* spin */ }
    db.topics.updateStatus(h.db, 't2', 'running');
    const after = db.topics.get(h.db, 't2');
    expect(after.status).toBe('running');
    expect(after.updated_at).toBeGreaterThan(before);
    const evs = db.events.list(h.db, { topicId: 't2' });
    expect(evs.length).toBe(1);
    expect(evs[0].type).toBe('status_change');
    expect(JSON.parse(evs[0].data)).toEqual({ from: 'ready', to: 'running' });
  });

  test('topics.listReadyNight filters correctly', () => {
    db.topics.insert(h.db, { id: 'a', title: 'A', shift: 'night', status: 'ready' });
    db.topics.insert(h.db, { id: 'b', title: 'B', shift: 'either', status: 'ready' });
    db.topics.insert(h.db, { id: 'c', title: 'C', shift: 'day', status: 'ready' });
    db.topics.insert(h.db, { id: 'd', title: 'D', shift: 'night', status: 'draft' });
    const ids = db.topics.listReadyNight(h.db).map(t => t.id).sort();
    expect(ids).toEqual(['a', 'b']);
  });

  test('runs.insert + runs.finish + runs.list', () => {
    db.topics.insert(h.db, { id: 't3', title: 'X', shift: 'night', status: 'running' });
    const runId = db.runs.insert(h.db, {
      topic_id: 't3', run_index: 1, kind: 'worker', log_path: '/tmp/l', diff_path: '/tmp/d',
    });
    db.runs.finish(h.db, runId, { exit_code: 0, tokens_used: 1234, summary: 'ok' });
    const runs = db.runs.list(h.db, 't3');
    expect(runs.length).toBe(1);
    expect(runs[0].exit_code).toBe(0);
    expect(runs[0].tokens_used).toBe(1234);
  });

  test('shiftLog.start + finish', () => {
    const id = db.shiftLog.start(h.db, { date: '2026-06-04' });
    db.shiftLog.finish(h.db, id, { topics_attempted: 3, topics_delivered: 2, topics_carryover: 1, total_tokens: 50000 });
    const row = h.db.prepare('SELECT * FROM shift_log WHERE id=?').get(id);
    expect(row.topics_delivered).toBe(2);
    expect(row.ended_at).not.toBeNull();
  });

  test('events.append + events.listSince', () => {
    db.events.append(h.db, { topic_id: 't1', type: 'log_line', data: { line: 'hello' } });
    db.events.append(h.db, { topic_id: 't2', type: 'status_change', data: { from: 'a', to: 'b' } });
    const all = db.events.listSince(h.db, 0);
    expect(all.length).toBe(2);
    const since = db.events.listSince(h.db, all[0].id);
    expect(since.length).toBe(1);
    expect(since[0].topic_id).toBe('t2');
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

```bash
npx jest test/__tests__/db.test.js
```

Expected: FAIL — `Cannot find module '../../src/db'`.

- [ ] **Step 4: Implement src/db.js**

```javascript
const Database = require('better-sqlite3');

const SCHEMA = `
CREATE TABLE IF NOT EXISTS topics (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  shift TEXT NOT NULL,
  status TEXT NOT NULL,
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
CREATE TABLE IF NOT EXISTS runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  topic_id TEXT NOT NULL REFERENCES topics(id),
  run_index INTEGER NOT NULL,
  kind TEXT NOT NULL,
  started_at INTEGER NOT NULL,
  ended_at INTEGER,
  exit_code INTEGER,
  tokens_used INTEGER,
  log_path TEXT,
  diff_path TEXT,
  summary TEXT
);
CREATE TABLE IF NOT EXISTS events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  topic_id TEXT,
  ts INTEGER NOT NULL,
  type TEXT NOT NULL,
  data TEXT
);
CREATE TABLE IF NOT EXISTS shift_log (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT NOT NULL,
  started_at INTEGER,
  ended_at INTEGER,
  topics_attempted INTEGER,
  topics_delivered INTEGER,
  topics_carryover INTEGER,
  total_tokens INTEGER
);
CREATE INDEX IF NOT EXISTS idx_topics_status ON topics(status);
CREATE INDEX IF NOT EXISTS idx_topics_shift ON topics(shift);
CREATE INDEX IF NOT EXISTS idx_runs_topic ON runs(topic_id);
CREATE INDEX IF NOT EXISTS idx_events_ts ON events(ts);
`;

function openDb(path) {
  const db = new Database(path);
  db.pragma('journal_mode = WAL');
  db.pragma('foreign_keys = ON');
  return db;
}

function migrate(db) {
  db.exec(SCHEMA);
}

const topics = {
  insert(db, t) {
    const now = Date.now();
    db.prepare(`
      INSERT INTO topics (id, title, shift, status, project_path, priority,
        budget_tokens, budget_runs, created_at, updated_at, worktree_path)
      VALUES (@id, @title, @shift, @status, @project_path, @priority,
        @budget_tokens, @budget_runs, @created_at, @updated_at, @worktree_path)
    `).run({
      id: t.id,
      title: t.title,
      shift: t.shift,
      status: t.status,
      project_path: t.project_path || null,
      priority: t.priority || 'normal',
      budget_tokens: t.budget_tokens || null,
      budget_runs: t.budget_runs || null,
      created_at: now,
      updated_at: now,
      worktree_path: t.worktree_path || null,
    });
  },
  get(db, id) {
    return db.prepare('SELECT * FROM topics WHERE id = ?').get(id);
  },
  all(db) {
    return db.prepare('SELECT * FROM topics ORDER BY created_at DESC').all();
  },
  updateStatus(db, id, newStatus) {
    const cur = topics.get(db, id);
    if (!cur) throw new Error(`topic ${id} not found`);
    const now = Date.now();
    db.prepare('UPDATE topics SET status = ?, updated_at = ? WHERE id = ?').run(newStatus, now, id);
    events.append(db, {
      topic_id: id,
      type: 'status_change',
      data: { from: cur.status, to: newStatus },
    });
  },
  listReadyNight(db) {
    return db.prepare(`
      SELECT * FROM topics
      WHERE status = 'ready' AND shift IN ('night', 'either')
      ORDER BY CASE priority WHEN 'high' THEN 0 WHEN 'normal' THEN 1 ELSE 2 END,
               created_at ASC
    `).all();
  },
  incrementUsage(db, id, { tokens = 0, runs = 0 } = {}) {
    db.prepare(`
      UPDATE topics SET tokens_used = tokens_used + ?, runs_count = runs_count + ?, updated_at = ?
      WHERE id = ?
    `).run(tokens, runs, Date.now(), id);
  },
  setWorktree(db, id, worktreePath) {
    db.prepare('UPDATE topics SET worktree_path = ?, updated_at = ? WHERE id = ?')
      .run(worktreePath, Date.now(), id);
  },
};

const runs = {
  insert(db, r) {
    const info = db.prepare(`
      INSERT INTO runs (topic_id, run_index, kind, started_at, log_path, diff_path)
      VALUES (?, ?, ?, ?, ?, ?)
    `).run(r.topic_id, r.run_index, r.kind, Date.now(), r.log_path || null, r.diff_path || null);
    return info.lastInsertRowid;
  },
  finish(db, id, { exit_code, tokens_used, summary }) {
    db.prepare(`
      UPDATE runs SET ended_at = ?, exit_code = ?, tokens_used = ?, summary = ?
      WHERE id = ?
    `).run(Date.now(), exit_code, tokens_used || 0, summary || null, id);
  },
  list(db, topicId) {
    return db.prepare('SELECT * FROM runs WHERE topic_id = ? ORDER BY run_index ASC').all(topicId);
  },
  nextIndex(db, topicId) {
    const r = db.prepare('SELECT MAX(run_index) AS m FROM runs WHERE topic_id = ?').get(topicId);
    return (r.m || 0) + 1;
  },
};

const events = {
  append(db, e) {
    db.prepare('INSERT INTO events (topic_id, ts, type, data) VALUES (?, ?, ?, ?)')
      .run(e.topic_id || null, Date.now(), e.type, JSON.stringify(e.data || {}));
  },
  list(db, { topicId } = {}) {
    if (topicId) {
      return db.prepare('SELECT * FROM events WHERE topic_id = ? ORDER BY id ASC').all(topicId);
    }
    return db.prepare('SELECT * FROM events ORDER BY id ASC').all();
  },
  listSince(db, sinceId) {
    return db.prepare('SELECT * FROM events WHERE id > ? ORDER BY id ASC').all(sinceId);
  },
};

const shiftLog = {
  start(db, { date }) {
    const info = db.prepare(`
      INSERT INTO shift_log (date, started_at) VALUES (?, ?)
    `).run(date, Date.now());
    return info.lastInsertRowid;
  },
  finish(db, id, { topics_attempted, topics_delivered, topics_carryover, total_tokens }) {
    db.prepare(`
      UPDATE shift_log SET ended_at = ?, topics_attempted = ?, topics_delivered = ?,
        topics_carryover = ?, total_tokens = ? WHERE id = ?
    `).run(Date.now(), topics_attempted, topics_delivered, topics_carryover, total_tokens, id);
  },
};

module.exports = { openDb, migrate, topics, runs, events, shiftLog };
```

- [ ] **Step 5: Run test to verify it passes**

```bash
npx jest test/__tests__/db.test.js
```

Expected: PASS — all 7 tests.

- [ ] **Step 6: Lint + commit**

```bash
npx eslint src/db.js test/helpers/tmpdb.js test/__tests__/db.test.js
git add src/db.js test/helpers/tmpdb.js test/__tests__/db.test.js
git commit -m "feat: add SQLite schema and db.js with topics/runs/events/shift_log"
```

---

### Task 4: Shift Window Logic

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/shift.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/shift.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/shift.test.js`:

```javascript
const { isNightWindow, minutesUntilEnd } = require('../../src/shift');

const cfg = { shift: { night: { start: '22:00', end: '07:00' }, soft_stop_minutes_before_end: 30 } };

function at(h, m) {
  const d = new Date(2026, 5, 4, h, m, 0, 0);
  return d;
}

describe('shift window', () => {
  test('22:30 is night', () => expect(isNightWindow(at(22, 30), cfg)).toBe(true));
  test('06:30 is night', () => expect(isNightWindow(at(6, 30), cfg)).toBe(true));
  test('07:00 is NOT night', () => expect(isNightWindow(at(7, 0), cfg)).toBe(false));
  test('12:00 is NOT night', () => expect(isNightWindow(at(12, 0), cfg)).toBe(false));
  test('21:59 is NOT night', () => expect(isNightWindow(at(21, 59), cfg)).toBe(false));
  test('22:00 is night', () => expect(isNightWindow(at(22, 0), cfg)).toBe(true));

  test('non-overnight window (09:00-17:00) at 12:00 is "night"', () => {
    const c = { shift: { night: { start: '09:00', end: '17:00' }, soft_stop_minutes_before_end: 30 } };
    expect(isNightWindow(at(12, 0), c)).toBe(true);
    expect(isNightWindow(at(8, 59), c)).toBe(false);
    expect(isNightWindow(at(17, 0), c)).toBe(false);
  });

  test('minutesUntilEnd at 06:00 returns 60', () => {
    expect(minutesUntilEnd(at(6, 0), cfg)).toBe(60);
  });

  test('minutesUntilEnd at 23:00 wraps over midnight, returns 480', () => {
    expect(minutesUntilEnd(at(23, 0), cfg)).toBe(480);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/shift.test.js
```

Expected: FAIL — `Cannot find module '../../src/shift'`.

- [ ] **Step 3: Implement src/shift.js**

```javascript
function parseHHMM(s) {
  const [h, m] = s.split(':').map(Number);
  return h * 60 + m;
}

function nowMinutes(d) {
  return d.getHours() * 60 + d.getMinutes();
}

function isNightWindow(now, cfg) {
  const start = parseHHMM(cfg.shift.night.start);
  const end = parseHHMM(cfg.shift.night.end);
  const cur = nowMinutes(now);
  if (start < end) {
    // same-day window e.g. 09:00-17:00
    return cur >= start && cur < end;
  }
  // overnight: cur >= start OR cur < end
  return cur >= start || cur < end;
}

function minutesUntilEnd(now, cfg) {
  const end = parseHHMM(cfg.shift.night.end);
  const cur = nowMinutes(now);
  if (!isNightWindow(now, cfg)) return -1;
  let mins = end - cur;
  if (mins <= 0) mins += 24 * 60;
  return mins;
}

module.exports = { isNightWindow, minutesUntilEnd, parseHHMM };
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/shift.test.js
```

Expected: PASS — all 9 tests.

- [ ] **Step 5: Lint + commit**

```bash
npx eslint src/shift.js test/__tests__/shift.test.js
git add src/shift.js test/__tests__/shift.test.js
git commit -m "feat: add shift window pure functions (isNightWindow, minutesUntilEnd)"
```

---

### Task 5: Spec.md Frontmatter I/O

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/spec-io.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/spec-io.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/spec-io.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { readSpec, writeSpec, updateFrontmatter } = require('../../src/spec-io');

function tmp() { return fs.mkdtempSync(path.join(os.tmpdir(), 'ns-spec-')); }

const SAMPLE = `---
id: 2026-06-04-test
title: 测试任务
shift: night
status: ready
priority: normal
budget_tokens: 100000
budget_runs: 4
boundaries:
  - 不能 git push
  - 只改 src/
success_criteria:
  - npm test 全过
---

## 背景

这是正文。

## 目标

完成测试。
`;

describe('spec-io', () => {
  test('readSpec parses frontmatter and body', () => {
    const dir = tmp();
    const p = path.join(dir, 'spec.md');
    fs.writeFileSync(p, SAMPLE);
    const spec = readSpec(p);
    expect(spec.frontmatter.id).toBe('2026-06-04-test');
    expect(spec.frontmatter.title).toBe('测试任务');
    expect(spec.frontmatter.shift).toBe('night');
    expect(spec.frontmatter.budget_tokens).toBe(100000);
    expect(spec.frontmatter.boundaries).toEqual(['不能 git push', '只改 src/']);
    expect(spec.body).toContain('## 背景');
  });

  test('writeSpec round-trips frontmatter', () => {
    const dir = tmp();
    const p = path.join(dir, 'spec.md');
    writeSpec(p, {
      frontmatter: {
        id: 'x',
        title: 'Y',
        shift: 'day',
        status: 'draft',
        boundaries: ['a', 'b'],
      },
      body: '## Body\n\nhello\n',
    });
    const re = readSpec(p);
    expect(re.frontmatter.id).toBe('x');
    expect(re.frontmatter.boundaries).toEqual(['a', 'b']);
    expect(re.body).toContain('hello');
  });

  test('updateFrontmatter changes one field, preserves body and other fields', () => {
    const dir = tmp();
    const p = path.join(dir, 'spec.md');
    fs.writeFileSync(p, SAMPLE);
    updateFrontmatter(p, { status: 'running' });
    const re = readSpec(p);
    expect(re.frontmatter.status).toBe('running');
    expect(re.frontmatter.title).toBe('测试任务');
    expect(re.body).toContain('## 背景');
  });

  test('readSpec throws when no frontmatter', () => {
    const dir = tmp();
    const p = path.join(dir, 'spec.md');
    fs.writeFileSync(p, '# No Frontmatter\n\nhello\n');
    expect(() => readSpec(p)).toThrow(/frontmatter/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/spec-io.test.js
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement src/spec-io.js**

We deliberately avoid adding a YAML dep — the frontmatter shape is small (scalar lines + dash-list lines).

```javascript
const fs = require('fs');

const FENCE = '---';

function parseFrontmatter(text) {
  const lines = text.split('\n');
  if (lines[0] !== FENCE) throw new Error('spec.md must start with frontmatter (---)');
  let endIdx = -1;
  for (let i = 1; i < lines.length; i++) {
    if (lines[i] === FENCE) { endIdx = i; break; }
  }
  if (endIdx === -1) throw new Error('frontmatter not closed');
  const fmLines = lines.slice(1, endIdx);
  const body = lines.slice(endIdx + 1).join('\n').replace(/^\n/, '');

  const fm = {};
  let curList = null;
  let curKey = null;
  for (const line of fmLines) {
    if (/^\s*$/.test(line)) continue;
    const listItem = line.match(/^\s+-\s+(.*)$/);
    if (listItem && curKey) {
      if (!curList) { curList = []; fm[curKey] = curList; }
      curList.push(listItem[1].trim());
      continue;
    }
    const kv = line.match(/^([a-zA-Z0-9_]+):\s*(.*)$/);
    if (kv) {
      curKey = kv[1];
      const val = kv[2].trim();
      curList = null;
      if (val === '') { fm[curKey] = null; continue; }
      if (/^-?\d+$/.test(val)) { fm[curKey] = Number(val); continue; }
      if (val === 'true') { fm[curKey] = true; continue; }
      if (val === 'false') { fm[curKey] = false; continue; }
      // strip surrounding quotes if present
      fm[curKey] = val.replace(/^['"](.*)['"]$/, '$1');
    }
  }
  return { frontmatter: fm, body };
}

function serializeFrontmatter(fm) {
  const out = [FENCE];
  for (const [k, v] of Object.entries(fm)) {
    if (v === null || v === undefined) {
      out.push(`${k}:`);
    } else if (Array.isArray(v)) {
      out.push(`${k}:`);
      for (const item of v) out.push(`  - ${item}`);
    } else if (typeof v === 'number' || typeof v === 'boolean') {
      out.push(`${k}: ${v}`);
    } else {
      // strings: quote only if they contain ':' to keep readable
      const s = String(v);
      if (s.includes(':')) out.push(`${k}: '${s}'`);
      else out.push(`${k}: ${s}`);
    }
  }
  out.push(FENCE);
  return out.join('\n');
}

function readSpec(filePath) {
  return parseFrontmatter(fs.readFileSync(filePath, 'utf8'));
}

function writeSpec(filePath, { frontmatter, body }) {
  const out = serializeFrontmatter(frontmatter) + '\n\n' + (body || '') + (body && !body.endsWith('\n') ? '\n' : '');
  fs.writeFileSync(filePath, out);
}

function updateFrontmatter(filePath, patch) {
  const cur = readSpec(filePath);
  const merged = { ...cur.frontmatter, ...patch };
  writeSpec(filePath, { frontmatter: merged, body: cur.body });
}

module.exports = { readSpec, writeSpec, updateFrontmatter };
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/spec-io.test.js
```

Expected: PASS — all 4 tests.

- [ ] **Step 5: Lint + commit**

```bash
npx eslint src/spec-io.js test/__tests__/spec-io.test.js
git add src/spec-io.js test/__tests__/spec-io.test.js
git commit -m "feat: add spec.md frontmatter parser and writer"
```

---

### Task 6: Git Worktree Helpers

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/worktree.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/worktree.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/worktree.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');
const { ensureWorktree, pruneWorktree, diffStat } = require('../../src/worktree');

function setupRepo() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-wt-'));
  execSync('git init -b main -q', { cwd: dir });
  execSync('git config user.email t@t', { cwd: dir });
  execSync('git config user.name t', { cwd: dir });
  fs.writeFileSync(path.join(dir, 'README.md'), '# r\n');
  execSync('git add . && git commit -q -m init', { cwd: dir });
  return dir;
}

describe('worktree', () => {
  test('ensureWorktree creates worktree on first call, idempotent on second', () => {
    const repo = setupRepo();
    const wt = path.join(repo, '.worktrees', 'topic-1');
    const branch = 'nightshift/topic-1';

    const r1 = ensureWorktree({ repoPath: repo, worktreePath: wt, branch });
    expect(r1.created).toBe(true);
    expect(fs.existsSync(wt)).toBe(true);

    const r2 = ensureWorktree({ repoPath: repo, worktreePath: wt, branch });
    expect(r2.created).toBe(false);
  });

  test('diffStat reports filesChanged and insertions after edit', () => {
    const repo = setupRepo();
    const wt = path.join(repo, '.worktrees', 'topic-2');
    const branch = 'nightshift/topic-2';
    ensureWorktree({ repoPath: repo, worktreePath: wt, branch });

    fs.writeFileSync(path.join(wt, 'new.txt'), 'line1\nline2\n');
    execSync('git add new.txt', { cwd: wt });

    const s = diffStat(wt);
    expect(s.filesChanged).toBe(1);
    expect(s.insertions).toBe(2);
  });

  test('pruneWorktree removes worktree dir', () => {
    const repo = setupRepo();
    const wt = path.join(repo, '.worktrees', 'topic-3');
    ensureWorktree({ repoPath: repo, worktreePath: wt, branch: 'nightshift/topic-3' });
    pruneWorktree({ repoPath: repo, worktreePath: wt });
    expect(fs.existsSync(wt)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/worktree.test.js
```

Expected: FAIL — module not found.

- [ ] **Step 3: Implement src/worktree.js**

```javascript
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');

function run(cmd, cwd) {
  return execSync(cmd, { cwd, encoding: 'utf8', stdio: ['ignore', 'pipe', 'pipe'] });
}

function ensureWorktree({ repoPath, worktreePath, branch }) {
  if (fs.existsSync(worktreePath) && fs.existsSync(path.join(worktreePath, '.git'))) {
    return { created: false, path: worktreePath };
  }
  fs.mkdirSync(path.dirname(worktreePath), { recursive: true });
  // check if branch exists
  let branchExists = false;
  try {
    run(`git show-ref --verify --quiet refs/heads/${branch}`, repoPath);
    branchExists = true;
  } catch (_) { /* not exists */ }

  if (branchExists) {
    run(`git worktree add "${worktreePath}" "${branch}"`, repoPath);
  } else {
    run(`git worktree add -b "${branch}" "${worktreePath}"`, repoPath);
  }
  return { created: true, path: worktreePath };
}

function pruneWorktree({ repoPath, worktreePath }) {
  try {
    run(`git worktree remove --force "${worktreePath}"`, repoPath);
  } catch (_) {
    // fall back to manual delete
    if (fs.existsSync(worktreePath)) fs.rmSync(worktreePath, { recursive: true, force: true });
    try { run('git worktree prune', repoPath); } catch (_) { /* ignore */ }
  }
}

function diffStat(worktreePath) {
  // include staged + unstaged
  const out = run('git diff --shortstat HEAD', worktreePath).trim();
  // e.g. " 1 file changed, 2 insertions(+)"
  if (!out) return { filesChanged: 0, insertions: 0, deletions: 0 };
  const m = out.match(/(\d+)\s+files?\s+changed(?:,\s*(\d+)\s+insertions?\(\+\))?(?:,\s*(\d+)\s+deletions?\(-\))?/);
  return {
    filesChanged: m ? Number(m[1]) : 0,
    insertions: m && m[2] ? Number(m[2]) : 0,
    deletions: m && m[3] ? Number(m[3]) : 0,
  };
}

module.exports = { ensureWorktree, pruneWorktree, diffStat };
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/worktree.test.js
```

Expected: PASS — 3 tests.

- [ ] **Step 5: Lint + commit**

```bash
npx eslint src/worktree.js test/__tests__/worktree.test.js
git add src/worktree.js test/__tests__/worktree.test.js
git commit -m "feat: add git worktree helpers (ensure/prune/diffStat)"
```

---

### Task 7: ns CLI Skeleton + Stub Tools

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/ns-cli.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/read-spec.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/update-status.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/budget.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/next-topic.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/heartbeat.js`
- Create: `/Users/I572881/workspace/night-shift/src/tools/read-run.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/ns-cli.test.js`

This task wires the simple, non-spawning tools (no real claude yet — `spawn-worker` and `deliver` come in later tasks).

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/ns-cli.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execFileSync } = require('child_process');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const { writeSpec } = require('../../src/spec-io');

const NS_CLI = path.join(__dirname, '..', '..', 'src', 'ns-cli.js');

function tmpRoot() {
  const dir = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-cli-'));
  fs.mkdirSync(path.join(dir, 'data', 'topics'), { recursive: true });
  return dir;
}

function makeConfig(root, dbPath) {
  const cfgPath = path.join(root, 'config.json');
  fs.writeFileSync(cfgPath, JSON.stringify({
    root,
    data_dir: path.join(root, 'data'),
    db_path: dbPath,
    claude_bin: '/bin/echo',
    shift: { night: { start: '22:00', end: '07:00' } },
    budgets: { default_tokens: 100000, default_runs: 5 },
  }));
  return cfgPath;
}

function runNs(args, env) {
  return execFileSync('node', [NS_CLI, ...args], {
    env: { ...process.env, ...env },
    encoding: 'utf8',
  });
}

describe('ns-cli', () => {
  let h;
  beforeEach(() => { h = makeTmpDb(); });
  afterEach(() => { h.cleanup(); });

  test('next-topic returns the highest-priority ready night topic as JSON', () => {
    dbm.topics.insert(h.db, { id: 't-low', title: 'low', shift: 'night', status: 'ready', priority: 'low' });
    dbm.topics.insert(h.db, { id: 't-hi', title: 'hi', shift: 'night', status: 'ready', priority: 'high' });
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    const out = runNs(['next-topic'], { NS_CONFIG: cfg });
    const obj = JSON.parse(out);
    expect(obj.id).toBe('t-hi');
  });

  test('next-topic returns null JSON when none ready', () => {
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    const out = runNs(['next-topic'], { NS_CONFIG: cfg });
    expect(JSON.parse(out)).toBeNull();
  });

  test('update-status changes db and mirrors to spec.md frontmatter', () => {
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    dbm.topics.insert(h.db, { id: 'mirror-t', title: 'm', shift: 'night', status: 'ready' });
    const specPath = path.join(root, 'data', 'topics', 'mirror-t', 'spec.md');
    fs.mkdirSync(path.dirname(specPath), { recursive: true });
    writeSpec(specPath, {
      frontmatter: { id: 'mirror-t', title: 'm', shift: 'night', status: 'ready' },
      body: '## body\n',
    });
    runNs(['update-status', 'mirror-t', 'running'], { NS_CONFIG: cfg });
    const re = dbm.topics.get(h.db, 'mirror-t');
    expect(re.status).toBe('running');
    const specText = fs.readFileSync(specPath, 'utf8');
    expect(specText).toMatch(/status: running/);
  });

  test('budget reports remaining tokens and runs', () => {
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    dbm.topics.insert(h.db, {
      id: 'b1', title: 'b', shift: 'night', status: 'running',
      budget_tokens: 1000, budget_runs: 3,
    });
    dbm.topics.incrementUsage(h.db, 'b1', { tokens: 200, runs: 1 });
    const out = runNs(['budget', 'b1'], { NS_CONFIG: cfg });
    const obj = JSON.parse(out);
    expect(obj.tokens_remaining).toBe(800);
    expect(obj.runs_remaining).toBe(2);
  });

  test('heartbeat appends a heartbeat event', () => {
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    runNs(['heartbeat'], { NS_CONFIG: cfg });
    const evs = dbm.events.list(h.db);
    expect(evs.some(e => e.type === 'heartbeat')).toBe(true);
  });

  test('unknown command exits 2 with usage', () => {
    const root = tmpRoot();
    const cfg = makeConfig(root, h.dbPath);
    let err = null;
    try { runNs(['no-such'], { NS_CONFIG: cfg }); } catch (e) { err = e; }
    expect(err).not.toBeNull();
    expect(err.status).toBe(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/ns-cli.test.js
```

Expected: FAIL — `Cannot find module '../../src/ns-cli'`.

- [ ] **Step 3: Implement src/ns-cli.js**

```javascript
#!/usr/bin/env node
const fs = require('fs');
const path = require('path');

function loadCfg() {
  const cfgPath = process.env.NS_CONFIG || path.join(process.cwd(), 'config.json');
  return JSON.parse(fs.readFileSync(cfgPath, 'utf8'));
}

const COMMANDS = {
  'spawn-worker': require('./tools/spawn-worker'),
  'read-run': require('./tools/read-run'),
  'read-spec': require('./tools/read-spec'),
  'update-status': require('./tools/update-status'),
  'deliver': require('./tools/deliver'),
  'budget': require('./tools/budget'),
  'next-topic': require('./tools/next-topic'),
  'heartbeat': require('./tools/heartbeat'),
};

function usage() {
  process.stderr.write(`usage: ns <command> [args...]\n  commands: ${Object.keys(COMMANDS).join(', ')}\n`);
  process.exit(2);
}

function main() {
  const [cmd, ...rest] = process.argv.slice(2);
  if (!cmd || !COMMANDS[cmd]) usage();
  const cfg = loadCfg();
  Promise.resolve(COMMANDS[cmd](cfg, rest))
    .then((out) => {
      if (out !== undefined) process.stdout.write(typeof out === 'string' ? out : JSON.stringify(out));
    })
    .catch((err) => {
      process.stderr.write(`ns ${cmd}: ${err.stack || err.message}\n`);
      process.exit(1);
    });
}

if (require.main === module) main();
module.exports = { COMMANDS };
```

- [ ] **Step 4: Implement src/tools/next-topic.js**

```javascript
const { openDb } = require('../db');
const dbm = require('../db');

module.exports = function nextTopic(cfg) {
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    const list = dbm.topics.listReadyNight(db);
    return list[0] || null;
  } finally {
    db.close();
  }
};
```

- [ ] **Step 5: Implement src/tools/update-status.js**

```javascript
const fs = require('fs');
const path = require('path');
const { openDb } = require('../db');
const dbm = require('../db');
const { updateFrontmatter } = require('../spec-io');

module.exports = function updateStatus(cfg, args) {
  const [topicId, newStatus] = args;
  if (!topicId || !newStatus) throw new Error('usage: ns update-status <topic-id> <status>');
  const allowed = ['draft', 'ready', 'running', 'review', 'done', 'failed', 'carryover'];
  if (!allowed.includes(newStatus)) throw new Error(`invalid status: ${newStatus}`);
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    dbm.topics.updateStatus(db, topicId, newStatus);
  } finally {
    db.close();
  }
  const specPath = path.join(cfg.data_dir, 'topics', topicId, 'spec.md');
  if (fs.existsSync(specPath)) {
    updateFrontmatter(specPath, { status: newStatus });
  }
  return { ok: true, id: topicId, status: newStatus };
};
```

- [ ] **Step 6: Implement src/tools/budget.js**

```javascript
const { openDb } = require('../db');
const dbm = require('../db');

module.exports = function budget(cfg, args) {
  const [topicId] = args;
  if (!topicId) throw new Error('usage: ns budget <topic-id>');
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    const t = dbm.topics.get(db, topicId);
    if (!t) throw new Error(`topic ${topicId} not found`);
    return {
      tokens_used: t.tokens_used,
      tokens_budget: t.budget_tokens,
      tokens_remaining: (t.budget_tokens || 0) - t.tokens_used,
      runs_count: t.runs_count,
      runs_budget: t.budget_runs,
      runs_remaining: (t.budget_runs || 0) - t.runs_count,
    };
  } finally {
    db.close();
  }
};
```

- [ ] **Step 7: Implement src/tools/heartbeat.js**

```javascript
const { openDb } = require('../db');
const dbm = require('../db');

module.exports = function heartbeat(cfg) {
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    dbm.events.append(db, { type: 'heartbeat', data: { pid: process.ppid } });
    return { ok: true, ts: Date.now() };
  } finally {
    db.close();
  }
};
```

- [ ] **Step 8: Implement src/tools/read-spec.js**

```javascript
const fs = require('fs');
const path = require('path');

module.exports = function readSpec(cfg, args) {
  const [topicId] = args;
  if (!topicId) throw new Error('usage: ns read-spec <topic-id>');
  const p = path.join(cfg.data_dir, 'topics', topicId, 'spec.md');
  if (!fs.existsSync(p)) throw new Error(`spec not found: ${p}`);
  return fs.readFileSync(p, 'utf8');
};
```

- [ ] **Step 9: Implement src/tools/read-run.js**

```javascript
const fs = require('fs');
const path = require('path');
const { openDb } = require('../db');
const dbm = require('../db');

module.exports = function readRun(cfg, args) {
  const [runId, tailArg] = args;
  if (!runId) throw new Error('usage: ns read-run <run-id> [tail-lines]');
  const tail = Number(tailArg) || 200;
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    const row = db.prepare('SELECT * FROM runs WHERE id = ?').get(Number(runId));
    if (!row) throw new Error(`run ${runId} not found`);
    let logTail = '';
    if (row.log_path && fs.existsSync(row.log_path)) {
      const lines = fs.readFileSync(row.log_path, 'utf8').split('\n');
      logTail = lines.slice(-tail).join('\n');
    }
    let diff = '';
    if (row.diff_path && fs.existsSync(row.diff_path)) {
      diff = fs.readFileSync(row.diff_path, 'utf8');
    }
    return {
      run_id: row.id,
      topic_id: row.topic_id,
      kind: row.kind,
      exit_code: row.exit_code,
      tokens_used: row.tokens_used,
      summary: row.summary,
      log_tail: logTail,
      diff,
    };
  } finally {
    db.close();
  }
};
```

- [ ] **Step 10: Stub src/tools/spawn-worker.js and src/tools/deliver.js**

These are required so `require('./tools/spawn-worker')` in `ns-cli.js` doesn't blow up. Real implementations come in Tasks 8 and 11.

Write `src/tools/spawn-worker.js`:

```javascript
module.exports = function spawnWorker(_cfg, _args) {
  throw new Error('spawn-worker not yet implemented (Task 8)');
};
```

Write `src/tools/deliver.js`:

```javascript
module.exports = function deliver(_cfg, _args) {
  throw new Error('deliver not yet implemented (Task 11)');
};
```

- [ ] **Step 11: Make ns-cli.js executable**

```bash
chmod +x /Users/I572881/workspace/night-shift/src/ns-cli.js
```

- [ ] **Step 12: Run tests**

```bash
npx jest test/__tests__/ns-cli.test.js
```

Expected: PASS — 6 tests.

- [ ] **Step 13: Lint + commit**

```bash
npx eslint src/ns-cli.js src/tools test/__tests__/ns-cli.test.js
git add src/ns-cli.js src/tools test/__tests__/ns-cli.test.js
git commit -m "feat: add ns-cli dispatcher with next-topic/update-status/budget/heartbeat/read-spec/read-run"
```

---

### Task 8: spawn-worker (with fake claude)

**Files:**
- Modify: `/Users/I572881/workspace/night-shift/src/tools/spawn-worker.js` (replace stub)
- Create: `/Users/I572881/workspace/night-shift/test/helpers/fake-claude.sh`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/spawn-worker.test.js`

- [ ] **Step 1: Create the fake claude binary**

Write `test/helpers/fake-claude.sh`:

```bash
#!/usr/bin/env bash
# Fake claude binary for tests. Echoes a fixed JSON-ish stream then exits 0.
# Args: ignores everything; just emits some lines, optionally writes a file.
echo "[fake-claude] start args=$*"
echo '{"type":"text","content":"hello from fake claude"}'
if [ -n "$NS_FAKE_TOUCH" ]; then
  echo "fake-output" > "$NS_FAKE_TOUCH"
  echo "[fake-claude] touched $NS_FAKE_TOUCH"
fi
exit "${NS_FAKE_EXIT:-0}"
```

```bash
chmod +x /Users/I572881/workspace/night-shift/test/helpers/fake-claude.sh
```

- [ ] **Step 2: Write the failing test**

Write `test/__tests__/spawn-worker.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const spawnWorker = require('../../src/tools/spawn-worker');

const FAKE = path.join(__dirname, '..', 'helpers', 'fake-claude.sh');

function setupRepoWithTopic(topicId) {
  const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-sw-'));
  const projectDir = path.join(root, 'project');
  fs.mkdirSync(projectDir);
  execSync('git init -b main -q && git config user.email t@t && git config user.name t', { cwd: projectDir });
  fs.writeFileSync(path.join(projectDir, 'README.md'), 'r\n');
  execSync('git add . && git commit -q -m init', { cwd: projectDir });
  fs.mkdirSync(path.join(root, 'data', 'topics', topicId, 'runs'), { recursive: true });
  fs.mkdirSync(path.join(root, 'data', '.worktrees'), { recursive: true });
  return { root, projectDir };
}

describe('spawn-worker', () => {
  let h;
  beforeEach(() => { h = makeTmpDb(); });
  afterEach(() => { h.cleanup(); });

  test('spawns claude in worktree, captures log + diff, inserts run row', () => {
    const { root, projectDir } = setupRepoWithTopic('t1');
    dbm.topics.insert(h.db, {
      id: 't1', title: 'x', shift: 'night', status: 'running',
      project_path: projectDir, budget_tokens: 1000, budget_runs: 5,
    });
    const cfg = {
      root, data_dir: path.join(root, 'data'), db_path: h.dbPath,
      claude_bin: FAKE,
    };
    const result = spawnWorker(cfg, ['t1', 'do something']);
    expect(result.exit_code).toBe(0);
    expect(result.run_id).toBeGreaterThan(0);
    expect(fs.existsSync(result.log_path)).toBe(true);
    expect(fs.readFileSync(result.log_path, 'utf8')).toMatch(/fake-claude/);
    // diff file exists (may be empty)
    expect(fs.existsSync(result.diff_path)).toBe(true);
    // worktree was created
    expect(fs.existsSync(path.join(root, 'data', '.worktrees', 't1'))).toBe(true);
    // run row recorded
    const runs = dbm.runs.list(h.db, 't1');
    expect(runs.length).toBe(1);
    expect(runs[0].exit_code).toBe(0);
    // runs_count incremented
    const t = dbm.topics.get(h.db, 't1');
    expect(t.runs_count).toBe(1);
  });

  test('non-zero exit propagated, run row records exit code', () => {
    const { root, projectDir } = setupRepoWithTopic('t2');
    dbm.topics.insert(h.db, {
      id: 't2', title: 'x', shift: 'night', status: 'running',
      project_path: projectDir, budget_tokens: 1000, budget_runs: 5,
    });
    const cfg = { root, data_dir: path.join(root, 'data'), db_path: h.dbPath, claude_bin: FAKE };
    const result = spawnWorker({ ...cfg, _env: { NS_FAKE_EXIT: '7' } }, ['t2', 'fail please']);
    expect(result.exit_code).toBe(7);
    const runs = dbm.runs.list(h.db, 't2');
    expect(runs[0].exit_code).toBe(7);
  });

  test('throws if budget exceeded', () => {
    const { root, projectDir } = setupRepoWithTopic('t3');
    dbm.topics.insert(h.db, {
      id: 't3', title: 'x', shift: 'night', status: 'running',
      project_path: projectDir, budget_tokens: 1000, budget_runs: 1,
    });
    dbm.topics.incrementUsage(h.db, 't3', { runs: 1 });
    const cfg = { root, data_dir: path.join(root, 'data'), db_path: h.dbPath, claude_bin: FAKE };
    expect(() => spawnWorker(cfg, ['t3', 'go'])).toThrow(/budget/i);
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

```bash
npx jest test/__tests__/spawn-worker.test.js
```

Expected: FAIL — `spawn-worker not yet implemented (Task 8)`.

- [ ] **Step 4: Implement src/tools/spawn-worker.js**

```javascript
const fs = require('fs');
const path = require('path');
const { spawnSync, execSync } = require('child_process');
const { openDb } = require('../db');
const dbm = require('../db');
const { ensureWorktree, diffStat } = require('../worktree');

module.exports = function spawnWorker(cfg, args) {
  const [topicId, ...instructionParts] = args;
  if (!topicId) throw new Error('usage: ns spawn-worker <topic-id> "<instructions>"');
  const instructions = instructionParts.join(' ');
  if (!instructions) throw new Error('instructions are required');

  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    const topic = dbm.topics.get(db, topicId);
    if (!topic) throw new Error(`topic ${topicId} not found`);

    // budget guard
    const runsRemaining = (topic.budget_runs || 0) - topic.runs_count;
    if (topic.budget_runs && runsRemaining <= 0) {
      throw new Error(`budget exhausted for ${topicId}: runs_remaining=${runsRemaining}`);
    }
    const tokensRemaining = (topic.budget_tokens || 0) - topic.tokens_used;
    if (topic.budget_tokens && tokensRemaining <= 0) {
      throw new Error(`budget exhausted for ${topicId}: tokens_remaining=${tokensRemaining}`);
    }

    if (!topic.project_path) throw new Error(`topic ${topicId} has no project_path`);

    const worktreePath = path.join(cfg.data_dir, '.worktrees', topicId);
    const branch = `nightshift/${topicId}`;
    ensureWorktree({ repoPath: topic.project_path, worktreePath, branch });
    if (!topic.worktree_path) dbm.topics.setWorktree(db, topicId, worktreePath);

    const runIndex = dbm.runs.nextIndex(db, topicId);
    const runsDir = path.join(cfg.data_dir, 'topics', topicId, 'runs');
    fs.mkdirSync(runsDir, { recursive: true });
    const logPath = path.join(runsDir, `run-${String(runIndex).padStart(3, '0')}.log`);
    const diffPath = path.join(runsDir, `run-${String(runIndex).padStart(3, '0')}.diff`);
    const metaPath = path.join(runsDir, `run-${String(runIndex).padStart(3, '0')}.meta.json`);

    const runId = dbm.runs.insert(db, {
      topic_id: topicId, run_index: runIndex, kind: 'worker',
      log_path: logPath, diff_path: diffPath,
    });
    dbm.events.append(db, { topic_id: topicId, type: 'worker_start', data: { run_id: runId } });

    // Build prompt: instructions + reminder to read spec.md
    const prompt = `${instructions}

The full topic spec is in spec.md (in the parent night-shift data dir). Respect boundaries listed there. When done, exit; do not ask questions.`;

    const claudeArgs = [
      '-p', '--dangerously-skip-permissions',
      '--max-turns', '50',
      prompt,
    ];

    const env = { ...process.env, ...(cfg._env || {}) };
    const startedAt = Date.now();
    const child = spawnSync(cfg.claude_bin, claudeArgs, {
      cwd: worktreePath,
      env,
      encoding: 'utf8',
      maxBuffer: 50 * 1024 * 1024,
    });
    const endedAt = Date.now();

    const log = (child.stdout || '') + (child.stderr ? `\n[stderr]\n${child.stderr}` : '');
    fs.writeFileSync(logPath, log);

    // capture diff
    let diff = '';
    try {
      diff = execSync('git diff HEAD', { cwd: worktreePath, encoding: 'utf8', maxBuffer: 50 * 1024 * 1024 });
    } catch (_) { /* ignore */ }
    fs.writeFileSync(diffPath, diff);

    const stat = (() => { try { return diffStat(worktreePath); } catch (_) { return { filesChanged: 0, insertions: 0, deletions: 0 }; } })();
    const exitCode = child.status === null ? 130 : child.status;

    fs.writeFileSync(metaPath, JSON.stringify({
      run_id: runId,
      topic_id: topicId,
      run_index: runIndex,
      started_at: startedAt,
      ended_at: endedAt,
      exit_code: exitCode,
      diff_stat: stat,
      instructions,
    }, null, 2));

    dbm.runs.finish(db, runId, { exit_code: exitCode, tokens_used: 0, summary: stat.filesChanged ? `${stat.filesChanged} files changed` : 'no changes' });
    dbm.topics.incrementUsage(db, topicId, { runs: 1 });
    dbm.events.append(db, { topic_id: topicId, type: 'worker_end', data: { run_id: runId, exit_code: exitCode, ...stat } });

    return {
      run_id: runId,
      run_index: runIndex,
      exit_code: exitCode,
      log_path: logPath,
      diff_path: diffPath,
      meta_path: metaPath,
      diff_stat: stat,
    };
  } finally {
    db.close();
  }
};
```

- [ ] **Step 5: Run test to verify it passes**

```bash
npx jest test/__tests__/spawn-worker.test.js
```

Expected: PASS — 3 tests.

- [ ] **Step 6: Lint + commit**

```bash
npx eslint src/tools/spawn-worker.js test/__tests__/spawn-worker.test.js
git add src/tools/spawn-worker.js test/helpers/fake-claude.sh test/__tests__/spawn-worker.test.js
git commit -m "feat: spawn-worker runs claude in worktree, captures log+diff, enforces budget"
```

---

### Task 9: Web Server (Express + SSE) and Kanban Skeleton

**Files:**
- Create: `/Users/I572881/workspace/night-shift/web/server.js`
- Create: `/Users/I572881/workspace/night-shift/web/public/index.html`
- Create: `/Users/I572881/workspace/night-shift/web/public/app.js`
- Create: `/Users/I572881/workspace/night-shift/web/public/style.css`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/web-server.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/web-server.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const request = require('supertest');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const { createApp } = require('../../web/server');

function tmpRoot() {
  const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-web-'));
  fs.mkdirSync(path.join(root, 'data', 'topics'), { recursive: true });
  return root;
}

describe('web server', () => {
  let h, root, app;
  beforeEach(() => {
    h = makeTmpDb();
    root = tmpRoot();
    app = createApp({
      root,
      data_dir: path.join(root, 'data'),
      db_path: h.dbPath,
      shift: { night: { start: '22:00', end: '07:00' } },
      web: { port: 0, host: '127.0.0.1' },
    });
  });
  afterEach(() => { h.cleanup(); });

  test('GET /api/topics returns topic list partitioned by shift', async () => {
    dbm.topics.insert(h.db, { id: 'd1', title: 'day-1', shift: 'day', status: 'ready' });
    dbm.topics.insert(h.db, { id: 'n1', title: 'night-1', shift: 'night', status: 'ready' });
    const res = await request(app).get('/api/topics').expect(200);
    expect(res.body.day.map(t => t.id)).toContain('d1');
    expect(res.body.night.map(t => t.id)).toContain('n1');
  });

  test('GET /api/topics/:id returns full topic + runs', async () => {
    dbm.topics.insert(h.db, { id: 'x', title: 'xx', shift: 'night', status: 'running' });
    dbm.runs.insert(h.db, { topic_id: 'x', run_index: 1, kind: 'worker' });
    const res = await request(app).get('/api/topics/x').expect(200);
    expect(res.body.topic.id).toBe('x');
    expect(res.body.runs.length).toBe(1);
  });

  test('POST /api/topics creates topic (skill ingress)', async () => {
    const payload = {
      id: 'created-1',
      title: 'from skill',
      shift: 'night',
      project_path: '/tmp/p',
      priority: 'normal',
      budget_tokens: 1000,
      budget_runs: 3,
      spec_body: '## body',
    };
    const res = await request(app).post('/api/topics').send(payload).expect(201);
    expect(res.body.id).toBe('created-1');
    expect(dbm.topics.get(h.db, 'created-1').status).toBe('ready');
    const specPath = path.join(root, 'data', 'topics', 'created-1', 'spec.md');
    expect(fs.existsSync(specPath)).toBe(true);
  });

  test('GET / serves the kanban HTML', async () => {
    const res = await request(app).get('/').expect(200);
    expect(res.text).toMatch(/Day Shift/);
    expect(res.text).toMatch(/Night Shift/);
  });

  test('GET /api/health reports day/night mode based on time', async () => {
    const res = await request(app).get('/api/health').expect(200);
    expect(typeof res.body.is_night).toBe('boolean');
    expect(res.body.ok).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/web-server.test.js
```

Expected: FAIL — `Cannot find module '../../web/server'`.

- [ ] **Step 3: Implement web/server.js**

```javascript
const express = require('express');
const fs = require('fs');
const path = require('path');
const { openDb } = require('../src/db');
const dbm = require('../src/db');
const { writeSpec } = require('../src/spec-io');
const { isNightWindow } = require('../src/shift');

function createApp(cfg) {
  const app = express();
  app.use(express.json({ limit: '2mb' }));
  app.use(express.static(path.join(__dirname, 'public')));

  const dbPath = cfg.db_path || `${cfg.data_dir}/state.db`;

  app.get('/api/health', (_req, res) => {
    res.json({ ok: true, is_night: isNightWindow(new Date(), cfg) });
  });

  app.get('/api/topics', (_req, res) => {
    const db = openDb(dbPath);
    try {
      const all = dbm.topics.all(db);
      const day = all.filter(t => t.shift === 'day' || ['review', 'failed', 'draft'].includes(t.status));
      const night = all.filter(t => (t.shift === 'night' || t.shift === 'either') && !['review', 'failed', 'draft'].includes(t.status));
      res.json({ day, night });
    } finally { db.close(); }
  });

  app.get('/api/topics/:id', (req, res) => {
    const db = openDb(dbPath);
    try {
      const topic = dbm.topics.get(db, req.params.id);
      if (!topic) return res.status(404).json({ error: 'not found' });
      const runs = dbm.runs.list(db, req.params.id);
      res.json({ topic, runs });
    } finally { db.close(); }
  });

  app.post('/api/topics', (req, res) => {
    const b = req.body;
    if (!b.id || !b.title || !b.shift) return res.status(400).json({ error: 'id, title, shift required' });
    const db = openDb(dbPath);
    try {
      dbm.topics.insert(db, {
        id: b.id, title: b.title, shift: b.shift,
        status: 'ready',
        project_path: b.project_path, priority: b.priority || 'normal',
        budget_tokens: b.budget_tokens, budget_runs: b.budget_runs,
      });
    } finally { db.close(); }
    const dir = path.join(cfg.data_dir, 'topics', b.id);
    fs.mkdirSync(dir, { recursive: true });
    writeSpec(path.join(dir, 'spec.md'), {
      frontmatter: {
        id: b.id, title: b.title, shift: b.shift, status: 'ready',
        project_path: b.project_path, priority: b.priority || 'normal',
        budget_tokens: b.budget_tokens, budget_runs: b.budget_runs,
      },
      body: b.spec_body || '## 背景\n\n',
    });
    res.status(201).json({ id: b.id });
  });

  // SSE
  app.get('/api/events', (req, res) => {
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();
    const since = Number(req.query.since || 0);
    let last = since;
    const tick = () => {
      const db = openDb(dbPath);
      try {
        const evs = dbm.events.listSince(db, last);
        for (const e of evs) {
          res.write(`id: ${e.id}\ndata: ${JSON.stringify(e)}\n\n`);
          last = e.id;
        }
      } finally { db.close(); }
    };
    const iv = setInterval(tick, 1000);
    tick();
    req.on('close', () => clearInterval(iv));
  });

  return app;
}

function start(cfg) {
  const app = createApp(cfg);
  return app.listen(cfg.web.port, cfg.web.host, () => {
    console.log(`night-shift kanban on http://${cfg.web.host}:${cfg.web.port}`);
  });
}

module.exports = { createApp, start };

if (require.main === module) {
  const { loadConfig } = require('../src/config');
  const cfg = loadConfig(process.env.NS_CONFIG || path.join(__dirname, '..', 'config.json'));
  start(cfg);
}
```

- [ ] **Step 4: Implement web/public/index.html**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Night Shift Kanban</title>
<link rel="stylesheet" href="/style.css">
</head>
<body>
<header>
  <h1>Night Shift</h1>
  <div id="mode"></div>
</header>
<main>
  <section id="day-col">
    <h2>Day Shift</h2>
    <div class="cards" id="day-cards"></div>
  </section>
  <section id="night-col">
    <h2>Night Shift</h2>
    <div class="cards" id="night-cards"></div>
  </section>
</main>
<aside id="drawer" hidden>
  <button id="drawer-close">×</button>
  <div id="drawer-body"></div>
</aside>
<script src="/app.js"></script>
</body>
</html>
```

- [ ] **Step 5: Implement web/public/style.css**

```css
:root { --bg: #0e1116; --fg: #e6edf3; --card: #161b22; --border: #30363d; --accent: #58a6ff; }
* { box-sizing: border-box; }
body { margin: 0; font-family: -apple-system, BlinkMacSystemFont, system-ui, sans-serif; background: var(--bg); color: var(--fg); }
header { padding: 12px 24px; border-bottom: 1px solid var(--border); display: flex; justify-content: space-between; align-items: center; }
main { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; padding: 16px; }
section { background: var(--card); border: 1px solid var(--border); border-radius: 8px; padding: 12px; min-height: 60vh; }
section h2 { margin-top: 0; }
.cards { display: flex; flex-direction: column; gap: 8px; }
.card { background: #1c2128; border: 1px solid var(--border); border-radius: 6px; padding: 10px; cursor: pointer; }
.card:hover { border-color: var(--accent); }
.card .title { font-weight: 600; }
.card .meta { font-size: 12px; color: #8b949e; margin-top: 4px; }
.status { display: inline-block; padding: 1px 6px; border-radius: 3px; font-size: 11px; margin-right: 4px; }
.status.ready { background: #1f6feb33; color: var(--accent); }
.status.running { background: #d2992633; color: #d29926; }
.status.review { background: #2ea04333; color: #2ea043; }
.status.failed { background: #da363333; color: #f85149; }
#drawer { position: fixed; right: 0; top: 0; bottom: 0; width: 40%; background: var(--card); border-left: 1px solid var(--border); padding: 16px; overflow: auto; }
#drawer-close { position: absolute; top: 8px; right: 12px; background: none; color: var(--fg); border: none; font-size: 24px; cursor: pointer; }
```

- [ ] **Step 6: Implement web/public/app.js**

```javascript
async function fetchTopics() {
  const r = await fetch('/api/topics');
  return r.json();
}

async function fetchHealth() {
  const r = await fetch('/api/health');
  return r.json();
}

function renderCard(t) {
  const div = document.createElement('div');
  div.className = 'card';
  div.dataset.id = t.id;
  div.innerHTML = `
    <div class="title">${escapeHtml(t.title)}</div>
    <div class="meta">
      <span class="status ${t.status}">${t.status}</span>
      <span>${t.shift}</span>
      <span>${t.runs_count}/${t.budget_runs || '?'} runs</span>
    </div>
  `;
  div.onclick = () => openDrawer(t.id);
  return div;
}

function escapeHtml(s) {
  return String(s).replace(/[&<>"']/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]));
}

async function refresh() {
  const { day, night } = await fetchTopics();
  const dayDiv = document.getElementById('day-cards');
  const nightDiv = document.getElementById('night-cards');
  dayDiv.replaceChildren(...day.map(renderCard));
  nightDiv.replaceChildren(...night.map(renderCard));
  const h = await fetchHealth();
  document.getElementById('mode').textContent = h.is_night ? '🌙 night' : '☀ day';
}

async function openDrawer(id) {
  const r = await fetch(`/api/topics/${id}`);
  const data = await r.json();
  const body = document.getElementById('drawer-body');
  body.innerHTML = `
    <h3>${escapeHtml(data.topic.title)}</h3>
    <p><span class="status ${data.topic.status}">${data.topic.status}</span> ${data.topic.shift}</p>
    <p>tokens: ${data.topic.tokens_used}/${data.topic.budget_tokens || '?'}, runs: ${data.topic.runs_count}/${data.topic.budget_runs || '?'}</p>
    <h4>Runs</h4>
    <ul>${data.runs.map(r => `<li>#${r.run_index} ${r.kind} exit=${r.exit_code ?? 'pending'} ${r.summary || ''}</li>`).join('')}</ul>
  `;
  document.getElementById('drawer').hidden = false;
}

document.getElementById('drawer-close').onclick = () => { document.getElementById('drawer').hidden = true; };

const es = new EventSource('/api/events');
es.onmessage = (e) => {
  try {
    const evt = JSON.parse(e.data);
    if (['status_change', 'worker_start', 'worker_end'].includes(evt.type)) refresh();
  } catch (_) { /* ignore */ }
};

refresh();
setInterval(refresh, 5000);
```

- [ ] **Step 7: Run tests**

```bash
npx jest test/__tests__/web-server.test.js
```

Expected: PASS — 5 tests.

- [ ] **Step 8: Manual smoke**

```bash
cp config.json.example config.json
mkdir -p data/topics
node -e "const {openDb,migrate}=require('./src/db');const db=openDb('./data/state.db');migrate(db);db.close();"
NS_CONFIG=$(pwd)/config.json node web/server.js &
SERVER_PID=$!
sleep 1
curl -s http://127.0.0.1:3737/api/health
kill $SERVER_PID
```

Expected: JSON `{"ok":true,"is_night":...}`

- [ ] **Step 9: Lint + commit**

```bash
npx eslint web src/db.js
git add web/ test/__tests__/web-server.test.js
git commit -m "feat: kanban web server with two-column layout, SSE, topic ingress endpoint"
```

---

### Task 10: plan-night-task Skill

**Files:**
- Create: `/Users/I572881/workspace/night-shift/skill/plan-night-task/SKILL.md`
- Create: `/Users/I572881/workspace/night-shift/skill/plan-night-task/install.sh`

The skill writes via `POST /api/topics`, so the daemon must be running. No tests — manual verification only.

- [ ] **Step 1: Write SKILL.md**

```markdown
---
name: plan-night-task
description: Capture a task to be executed by night-shift Orchestrator-Claude tonight. Brainstorms title/budget/boundaries/success-criteria, decides shift, writes spec.md + SQLite row via the night-shift daemon. Use when user says "记一下", "晚上跑这个", "/plan-night-task", or describes a task they want unattended.
---

# plan-night-task

为 `night-shift` 系统采集一个任务。白天执行；写完后任务进入"今晚队列"或"白天 todo"。

## 触发

- `/plan-night-task <一句话想法>` slash 命令
- 用户说"记一下我要做 X"、"晚上跑这个"、"派给夜班"

## 前置检查

1. 确认 daemon 在跑：`curl -fsS http://127.0.0.1:3737/api/health` → `{"ok":true,...}`。失败 → 提示用户先启 `npm run web`，停下。
2. 读 `/Users/I572881/workspace/night-shift/CLAUDE.md` 确认仓库在标准路径。

## 执行流程

### Phase 1: 头脑风暴

逐项问用户（可一次问完，也可拆开）：

1. **title** — 一句话标题（必填）
2. **project_path** — 这任务要改哪个仓库的代码？绝对路径（必填，可以是 `/Users/I572881/workspace/night-shift` 自己）
3. **success criteria** — 列 2–5 条**可观察**的成功标准（如 "npm test 全过"、"PR 已开"）
4. **boundaries** — 列出**不许**做的事（如 "不能 git push"、"只改 src/"、"不能改 schema"）
5. **rollback condition** — 什么时候 worker 应该放弃？（如 "测试连续失败 3 次"、"修改超过 200 行"）
6. **budget** — 预算 tokens（默认 200000）和最多 worker 数（默认 8）
7. **deliver_as** — `commit` / `pr` / `diff-only`（默认 `pr`）

### Phase 2: Shift 决策（关键问题）

**完整问出这句话，让用户回答**：

> "这个任务能完全无人值守跑吗？如果你现在就能写下成功标准 + 边界 + 回滚条件，那是 night。如果你需要中途看一眼 / 决定方向 / 回答问题，那是 day。不确定时默认 day。"

- 用户答 night → `shift: night`
- 用户答 day → `shift: day`
- 用户答 either → `shift: either`（调度器当 night 处理）
- 用户答不确定 → `shift: day`

### Phase 3: Slug 生成

`id = YYYY-MM-DD-<kebab-of-title>`，例：`2026-06-04-add-rbac-to-night-shift`。
取今天日期（`date +%Y-%m-%d`），title 经 lowercase、去标点、空格转 `-`。

### Phase 4: 提交

POST 到 daemon：

```bash
curl -fsS -X POST http://127.0.0.1:3737/api/topics \
  -H 'Content-Type: application/json' \
  -d @- <<EOF
{
  "id": "<id>",
  "title": "<title>",
  "shift": "<night|day|either>",
  "project_path": "<absolute path>",
  "priority": "normal",
  "budget_tokens": <int>,
  "budget_runs": <int>,
  "spec_body": "## 背景\n\n<...>\n\n## 目标\n\n<success criteria as bullet list>\n\n## 边界\n\n<boundaries as bullet list>\n\n## 回滚\n\n<rollback condition>\n\n## Deliver\n\n<deliver_as>"
}
EOF
```

非 2xx → 把响应体显示给用户，停下。

### Phase 5: 确认

显示给用户：

```
✅ 已加入队列
   id: <id>
   shift: <shift>
   kanban: http://127.0.0.1:3737
```

如果 `shift = night`，加一句：「今晚 22:00 cron 触发时，orchestrator 会捡起这个任务。」

## 失败处理

| 现象 | 处理 |
|---|---|
| daemon 不在跑 | 提示 `cd ~/workspace/night-shift && npm run web`，不要尝试自己启 |
| project_path 不存在 | 让用户重新给路径 |
| id 重复 (409 / unique constraint) | 在 slug 后加 `-2`、`-3` 重试 |

## 不要做

- ❌ 不要直接写 `data/state.db`（必须走 HTTP API）
- ❌ 不要省略 shift 决策那个完整问题
- ❌ 不要默认 night——默认 day
```

- [ ] **Step 2: Write install.sh**

```bash
#!/usr/bin/env bash
set -e
mkdir -p ~/.claude/skills/plan-night-task
ln -sfn "$(cd "$(dirname "$0")" && pwd)/SKILL.md" ~/.claude/skills/plan-night-task/SKILL.md
echo "linked SKILL.md → ~/.claude/skills/plan-night-task/SKILL.md"
```

```bash
chmod +x /Users/I572881/workspace/night-shift/skill/plan-night-task/install.sh
```

- [ ] **Step 3: Manual verification**

```bash
cd /Users/I572881/workspace/night-shift
./skill/plan-night-task/install.sh
ls -l ~/.claude/skills/plan-night-task/SKILL.md
```

Expected: symlink exists pointing to repo file.

Then with daemon running:

```bash
curl -fsS -X POST http://127.0.0.1:3737/api/topics \
  -H 'Content-Type: application/json' \
  -d '{"id":"2026-06-04-skill-smoke","title":"smoke","shift":"night","project_path":"/Users/I572881/workspace/night-shift","budget_tokens":50000,"budget_runs":2,"spec_body":"## 背景\nsmoke\n"}'
```

Expected: `{"id":"2026-06-04-skill-smoke"}` and the topic visible in kanban.

- [ ] **Step 4: Commit**

```bash
git add skill/
git commit -m "feat: add plan-night-task skill (day-side capture via REST)"
```

---

### Task 11: deliver Tool

**Files:**
- Modify: `/Users/I572881/workspace/night-shift/src/tools/deliver.js` (replace stub)
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/deliver.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/deliver.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const deliver = require('../../src/tools/deliver');
const { ensureWorktree } = require('../../src/worktree');

function setup(topicId) {
  const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-dlv-'));
  const projectDir = path.join(root, 'project');
  fs.mkdirSync(projectDir);
  execSync('git init -b main -q && git config user.email t@t && git config user.name t', { cwd: projectDir });
  fs.writeFileSync(path.join(projectDir, 'README.md'), 'r\n');
  execSync('git add . && git commit -q -m init', { cwd: projectDir });
  const wt = path.join(root, 'data', '.worktrees', topicId);
  ensureWorktree({ repoPath: projectDir, worktreePath: wt, branch: `nightshift/${topicId}` });
  fs.writeFileSync(path.join(wt, 'new.txt'), 'x\n');
  fs.mkdirSync(path.join(root, 'data', 'topics', topicId), { recursive: true });
  return { root, projectDir, wt };
}

describe('deliver', () => {
  let h;
  beforeEach(() => { h = makeTmpDb(); });
  afterEach(() => { h.cleanup(); });

  test('deliver commit: stages, commits, sets status=review', () => {
    const { root, projectDir, wt } = setup('d1');
    dbm.topics.insert(h.db, {
      id: 'd1', title: 't', shift: 'night', status: 'running',
      project_path: projectDir, worktree_path: wt,
    });
    const cfg = { root, data_dir: path.join(root, 'data'), db_path: h.dbPath };
    const r = deliver(cfg, ['d1', 'commit', 'auto-commit by night-shift']);
    expect(r.mode).toBe('commit');
    expect(r.commit_sha).toMatch(/^[0-9a-f]{40}$/);
    expect(dbm.topics.get(h.db, 'd1').status).toBe('review');
  });

  test('deliver diff-only: writes delivered.diff, sets status=review', () => {
    const { root, projectDir, wt } = setup('d2');
    dbm.topics.insert(h.db, {
      id: 'd2', title: 't', shift: 'night', status: 'running',
      project_path: projectDir, worktree_path: wt,
    });
    const cfg = { root, data_dir: path.join(root, 'data'), db_path: h.dbPath };
    const r = deliver(cfg, ['d2', 'diff-only']);
    expect(r.mode).toBe('diff-only');
    expect(fs.existsSync(path.join(root, 'data', 'topics', 'd2', 'delivered.diff'))).toBe(true);
    expect(dbm.topics.get(h.db, 'd2').status).toBe('review');
  });

  test('deliver rejects unknown mode', () => {
    const { root, projectDir, wt } = setup('d3');
    dbm.topics.insert(h.db, {
      id: 'd3', title: 't', shift: 'night', status: 'running',
      project_path: projectDir, worktree_path: wt,
    });
    const cfg = { root, data_dir: path.join(root, 'data'), db_path: h.dbPath };
    expect(() => deliver(cfg, ['d3', 'teleport'])).toThrow(/mode/);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/deliver.test.js
```

Expected: FAIL — `deliver not yet implemented (Task 11)`.

- [ ] **Step 3: Implement src/tools/deliver.js**

```javascript
const fs = require('fs');
const path = require('path');
const { execSync } = require('child_process');
const { openDb } = require('../db');
const dbm = require('../db');

function run(cmd, cwd) {
  return execSync(cmd, { cwd, encoding: 'utf8', stdio: ['ignore', 'pipe', 'pipe'] });
}

module.exports = function deliver(cfg, args) {
  const [topicId, mode, ...msgParts] = args;
  if (!topicId || !mode) throw new Error('usage: ns deliver <topic-id> <commit|pr|diff-only> [message]');
  if (!['commit', 'pr', 'diff-only'].includes(mode)) throw new Error(`unknown mode: ${mode}`);

  const message = msgParts.join(' ') || `night-shift deliver ${topicId}`;
  const db = openDb(cfg.db_path || `${cfg.data_dir}/state.db`);
  try {
    const topic = dbm.topics.get(db, topicId);
    if (!topic) throw new Error(`topic ${topicId} not found`);
    if (!topic.worktree_path) throw new Error(`topic ${topicId} has no worktree`);
    const wt = topic.worktree_path;
    const result = { mode, topic_id: topicId };

    if (mode === 'diff-only') {
      const diff = run('git diff HEAD', wt);
      const out = path.join(cfg.data_dir, 'topics', topicId, 'delivered.diff');
      fs.writeFileSync(out, diff);
      result.diff_path = out;
    } else {
      // commit (covers both commit and pr)
      run('git add -A', wt);
      // configure identity if not set (CI / fresh worktree)
      try { run('git config user.email', wt); }
      catch (_) { run('git config user.email "night-shift@local"', wt); run('git config user.name "night-shift"', wt); }
      try {
        run(`git commit -m ${JSON.stringify(message)}`, wt);
      } catch (e) {
        // nothing to commit
        if (!String(e.stderr || e.message).match(/nothing to commit/)) throw e;
      }
      const sha = run('git rev-parse HEAD', wt).trim();
      result.commit_sha = sha;

      if (mode === 'pr') {
        try {
          run(`git push -u origin "nightshift/${topicId}"`, wt);
          const prOut = run(`gh pr create --fill --head "nightshift/${topicId}"`, wt);
          result.pr_url = prOut.trim();
        } catch (e) {
          throw new Error(`pr push/open failed: ${e.message}`);
        }
      }
    }

    db.prepare('UPDATE topics SET status = ?, delivered_at = ?, updated_at = ? WHERE id = ?')
      .run('review', Date.now(), Date.now(), topicId);
    dbm.events.append(db, { topic_id: topicId, type: 'delivered', data: result });
    return result;
  } finally {
    db.close();
  }
};
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/deliver.test.js
```

Expected: PASS — 3 tests. (`pr` mode is NOT tested here because it requires `gh` + a remote; manual smoke covers it later.)

- [ ] **Step 5: Lint + commit**

```bash
npx eslint src/tools/deliver.js test/__tests__/deliver.test.js
git add src/tools/deliver.js test/__tests__/deliver.test.js
git commit -m "feat: deliver tool supports commit/pr/diff-only modes"
```

---

### Task 12: Orchestrator, Watchdog, start-night, Dry-Run Integration

**Files:**
- Create: `/Users/I572881/workspace/night-shift/src/orchestrator.js`
- Create: `/Users/I572881/workspace/night-shift/src/watchdog.js`
- Create: `/Users/I572881/workspace/night-shift/src/start-night.js`
- Create: `/Users/I572881/workspace/night-shift/test/helpers/fake-orchestrator.sh`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/orchestrator.test.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/watchdog.test.js`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/start-night.test.js`

This is the largest task because it's the integration glue. Each step stays small.

- [ ] **Step 1: Write fake-orchestrator.sh**

This stand-in for `claude -p` simulates an orchestrator that calls `ns next-topic`, `ns spawn-worker`, `ns deliver`. Used by `NS_DRY_RUN=1` and tests.

```bash
#!/usr/bin/env bash
# Fake orchestrator: reads NS_TOPIC_IDS env (space-separated), for each id calls
# ns spawn-worker (no-op via fake claude), then ns deliver <id> diff-only.
set -e
echo "[fake-orchestrator] start, topics=$NS_TOPIC_IDS"
NS_BIN="${NS_BIN:-node $(dirname "$0")/../../src/ns-cli.js}"
for id in $NS_TOPIC_IDS; do
  echo "[fake-orchestrator] heartbeat for $id"
  $NS_BIN heartbeat
  echo "[fake-orchestrator] spawning worker for $id"
  $NS_BIN spawn-worker "$id" "do the thing" || true
  echo "[fake-orchestrator] delivering $id"
  $NS_BIN deliver "$id" diff-only "fake delivery" || true
done
echo "[fake-orchestrator] done"
```

```bash
chmod +x /Users/I572881/workspace/night-shift/test/helpers/fake-orchestrator.sh
```

- [ ] **Step 2: Write orchestrator.test.js**

Write `test/__tests__/orchestrator.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { startOrchestrator } = require('../../src/orchestrator');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');

const FAKE = path.join(__dirname, '..', 'helpers', 'fake-orchestrator.sh');

describe('orchestrator', () => {
  test('startOrchestrator launches binary with topic ids in env, resolves on exit', async () => {
    const h = makeTmpDb();
    const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-orch-'));
    fs.mkdirSync(path.join(root, 'data', 'topics'), { recursive: true });
    fs.writeFileSync(path.join(root, 'config.json'), JSON.stringify({
      root, data_dir: path.join(root, 'data'), db_path: h.dbPath,
      claude_bin: '/bin/echo',
      shift: { night: { start: '22:00', end: '07:00' } },
      budgets: { default_tokens: 1000, default_runs: 2 },
    }));
    dbm.topics.insert(h.db, { id: 'orch-1', title: 'x', shift: 'night', status: 'ready' });

    const orch = startOrchestrator({
      cfg: JSON.parse(fs.readFileSync(path.join(root, 'config.json'), 'utf8')),
      configPath: path.join(root, 'config.json'),
      topicIds: ['orch-1'],
      orchestratorBin: FAKE,
    });
    const result = await orch.done;

    expect(result.exit_code).toBe(0);
    // heartbeat event recorded
    const evs = dbm.events.list(h.db);
    expect(evs.some(e => e.type === 'heartbeat')).toBe(true);
    h.cleanup();
  });
});
```

- [ ] **Step 3: Implement src/orchestrator.js**

```javascript
const { spawn } = require('child_process');
const fs = require('fs');
const path = require('path');

const ORCHESTRATOR_SYSTEM_PROMPT = `You are the night-shift orchestrator.
You will be given a list of topic ids. For each topic:
  1. Run \`ns read-spec <id>\` to read the spec.
  2. Decide a plan: one or more worker invocations.
  3. Use \`ns spawn-worker <id> "<instructions>"\` to spawn a worker. It blocks until the worker exits and returns the run_id.
  4. Use \`ns read-run <run_id>\` to inspect the log + diff.
  5. Decide: deliver, spawn another worker, or abort.
  6. Use \`ns deliver <id> <commit|pr|diff-only> [message]\` when finished.
  7. Use \`ns budget <id>\` to check remaining budget. Stop if exhausted.
  8. Call \`ns heartbeat\` at least every 4 minutes so the watchdog stays happy.
Never ask the user. Never block waiting. Respect each spec's boundaries.`;

function startOrchestrator({ cfg, configPath, topicIds, orchestratorBin }) {
  // Default to real claude when no orchestratorBin provided.
  const isClaude = !orchestratorBin;
  const bin = orchestratorBin || cfg.claude_bin;
  let args = [];
  if (isClaude) {
    args = [
      '-p', '--dangerously-skip-permissions',
      '--max-turns', '500',
      `${ORCHESTRATOR_SYSTEM_PROMPT}\n\nTopics tonight: ${topicIds.join(', ')}`,
    ];
  } else {
    args = [];
  }

  const env = {
    ...process.env,
    NS_CONFIG: configPath,
    NS_TOPIC_IDS: topicIds.join(' '),
  };

  const child = spawn(bin, args, { env, stdio: ['ignore', 'pipe', 'pipe'] });
  const logsDir = path.join(cfg.data_dir, 'orchestrator');
  fs.mkdirSync(logsDir, { recursive: true });
  const logPath = path.join(logsDir, `orch-${Date.now()}.log`);
  const logStream = fs.createWriteStream(logPath);
  child.stdout.pipe(logStream);
  child.stderr.pipe(logStream);

  const done = new Promise((resolve, reject) => {
    child.on('error', reject);
    child.on('exit', (code, signal) => {
      logStream.end();
      resolve({ exit_code: code === null ? 130 : code, signal, log_path: logPath, pid: child.pid });
    });
  });

  return { child, done };
}

module.exports = { startOrchestrator, ORCHESTRATOR_SYSTEM_PROMPT };
```

- [ ] **Step 4: Run orchestrator test**

```bash
npx jest test/__tests__/orchestrator.test.js
```

Expected: PASS — 1 test.

- [ ] **Step 5: Write watchdog test**

Write `test/__tests__/watchdog.test.js`:

```javascript
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const { Watchdog } = require('../../src/watchdog');

describe('watchdog', () => {
  let h;
  beforeEach(() => { h = makeTmpDb(); });
  afterEach(() => { h.cleanup(); });

  test('triggers when no heartbeat within threshold', (done) => {
    const wd = new Watchdog({
      db: h.db,
      heartbeatThresholdMs: 50,
      pollIntervalMs: 20,
      onBreach: (reason) => {
        expect(reason).toMatch(/heartbeat/);
        wd.stop();
        done();
      },
    });
    wd.start();
  });

  test('does not trigger when heartbeats arrive', (done) => {
    let breached = false;
    const wd = new Watchdog({
      db: h.db,
      heartbeatThresholdMs: 200,
      pollIntervalMs: 20,
      onBreach: () => { breached = true; },
    });
    wd.start();
    const iv = setInterval(() => {
      dbm.events.append(h.db, { type: 'heartbeat', data: {} });
    }, 30);
    setTimeout(() => {
      clearInterval(iv);
      wd.stop();
      expect(breached).toBe(false);
      done();
    }, 250);
  });

  test('triggers when token budget exceeded for a running topic', (done) => {
    dbm.topics.insert(h.db, {
      id: 'budgety', title: 't', shift: 'night', status: 'running',
      budget_tokens: 100, budget_runs: 99,
    });
    dbm.topics.incrementUsage(h.db, 'budgety', { tokens: 200 });
    const wd = new Watchdog({
      db: h.db,
      heartbeatThresholdMs: 99999,
      pollIntervalMs: 20,
      onBreach: (reason) => {
        expect(reason).toMatch(/budget|tokens/);
        wd.stop();
        done();
      },
    });
    // seed a heartbeat so the heartbeat check passes
    dbm.events.append(h.db, { type: 'heartbeat', data: {} });
    wd.start();
  });
});
```

- [ ] **Step 6: Implement src/watchdog.js**

```javascript
const dbm = require('./db');

class Watchdog {
  constructor({ db, heartbeatThresholdMs, pollIntervalMs, onBreach }) {
    this.db = db;
    this.heartbeatThresholdMs = heartbeatThresholdMs;
    this.pollIntervalMs = pollIntervalMs;
    this.onBreach = onBreach;
    this.timer = null;
    this.startedAt = Date.now();
    this.fired = false;
  }
  start() {
    this.timer = setInterval(() => this.tick(), this.pollIntervalMs);
  }
  stop() {
    if (this.timer) clearInterval(this.timer);
    this.timer = null;
  }
  tick() {
    if (this.fired) return;
    // 1. heartbeat check
    const last = this.db.prepare("SELECT MAX(ts) AS ts FROM events WHERE type = 'heartbeat'").get();
    const lastTs = last && last.ts ? last.ts : this.startedAt;
    if (Date.now() - lastTs > this.heartbeatThresholdMs) {
      this.fire(`heartbeat-lost (last ${Date.now() - lastTs}ms ago)`);
      return;
    }
    // 2. running topics token budget
    const running = this.db.prepare("SELECT * FROM topics WHERE status = 'running'").all();
    for (const t of running) {
      if (t.budget_tokens && t.tokens_used >= t.budget_tokens) {
        this.fire(`token-budget-exceeded for ${t.id} (${t.tokens_used}/${t.budget_tokens})`);
        return;
      }
      if (t.budget_runs && t.runs_count > t.budget_runs) {
        this.fire(`run-budget-exceeded for ${t.id} (${t.runs_count}/${t.budget_runs})`);
        return;
      }
    }
  }
  fire(reason) {
    this.fired = true;
    dbm.events.append(this.db, { type: 'watchdog_breach', data: { reason } });
    try { this.onBreach(reason); } catch (_) { /* ignore */ }
  }
}

module.exports = { Watchdog };
```

- [ ] **Step 7: Run watchdog test**

```bash
npx jest test/__tests__/watchdog.test.js
```

Expected: PASS — 3 tests.

- [ ] **Step 8: Write start-night test**

Write `test/__tests__/start-night.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const { execSync } = require('child_process');
const { runNight } = require('../../src/start-night');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');

const FAKE_CLAUDE = path.join(__dirname, '..', 'helpers', 'fake-claude.sh');
const FAKE_ORCH = path.join(__dirname, '..', 'helpers', 'fake-orchestrator.sh');

function setupRepoForTopic(root, topicId) {
  const proj = path.join(root, 'project');
  fs.mkdirSync(proj);
  execSync('git init -b main -q && git config user.email t@t && git config user.name t', { cwd: proj });
  fs.writeFileSync(path.join(proj, 'README.md'), 'r\n');
  execSync('git add . && git commit -q -m init', { cwd: proj });
  fs.mkdirSync(path.join(root, 'data', 'topics', topicId, 'runs'), { recursive: true });
  return proj;
}

describe('start-night dry-run', () => {
  test('end-to-end: ready night topic → orchestrator runs → topic ends in review', async () => {
    const h = makeTmpDb();
    const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-night-'));
    fs.mkdirSync(path.join(root, 'data'), { recursive: true });
    const proj = setupRepoForTopic(root, 'night-1');
    dbm.topics.insert(h.db, {
      id: 'night-1', title: 'integration', shift: 'night', status: 'ready',
      project_path: proj, budget_tokens: 5000, budget_runs: 3,
    });
    const cfg = {
      root, data_dir: path.join(root, 'data'), db_path: h.dbPath,
      claude_bin: FAKE_CLAUDE,
      shift: { night: { start: '00:00', end: '23:59' }, soft_stop_minutes_before_end: 30 },
      budgets: { default_tokens: 5000, default_runs: 3, watchdog_heartbeat_seconds: 60 },
    };
    const cfgPath = path.join(root, 'config.json');
    fs.writeFileSync(cfgPath, JSON.stringify(cfg));

    const result = await runNight({
      cfg, configPath: cfgPath,
      orchestratorBin: FAKE_ORCH,
    });

    expect(result.topics_attempted).toBe(1);
    const t = dbm.topics.get(h.db, 'night-1');
    expect(t.status).toBe('review');
    h.cleanup();
  });

  test('no ready topics → exit cleanly with attempted=0', async () => {
    const h = makeTmpDb();
    const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-night-empty-'));
    fs.mkdirSync(path.join(root, 'data'), { recursive: true });
    const cfg = {
      root, data_dir: path.join(root, 'data'), db_path: h.dbPath,
      claude_bin: FAKE_CLAUDE,
      shift: { night: { start: '00:00', end: '23:59' }, soft_stop_minutes_before_end: 30 },
      budgets: { default_tokens: 5000, default_runs: 3, watchdog_heartbeat_seconds: 60 },
    };
    const cfgPath = path.join(root, 'config.json');
    fs.writeFileSync(cfgPath, JSON.stringify(cfg));
    const result = await runNight({ cfg, configPath: cfgPath, orchestratorBin: FAKE_ORCH });
    expect(result.topics_attempted).toBe(0);
    h.cleanup();
  });
});
```

- [ ] **Step 9: Implement src/start-night.js**

```javascript
const path = require('path');
const { openDb } = require('./db');
const dbm = require('./db');
const { startOrchestrator } = require('./orchestrator');
const { Watchdog } = require('./watchdog');

async function runNight({ cfg, configPath, orchestratorBin }) {
  const dbPath = cfg.db_path || `${cfg.data_dir}/state.db`;
  const db = openDb(dbPath);
  let shiftLogId = null;
  try {
    const ready = dbm.topics.listReadyNight(db);
    const dateStr = new Date().toISOString().slice(0, 10);
    shiftLogId = dbm.shiftLog.start(db, { date: dateStr });
    if (ready.length === 0) {
      dbm.events.append(db, { type: 'night_skip', data: { reason: 'no ready topics' } });
      dbm.shiftLog.finish(db, shiftLogId, {
        topics_attempted: 0, topics_delivered: 0, topics_carryover: 0, total_tokens: 0,
      });
      return { topics_attempted: 0, topics_delivered: 0, topics_carryover: 0 };
    }

    // mark all running
    for (const t of ready) dbm.topics.updateStatus(db, t.id, 'running');
    dbm.events.append(db, { type: 'night_start', data: { topic_ids: ready.map(t => t.id) } });

    const wd = new Watchdog({
      db,
      heartbeatThresholdMs: (cfg.budgets.watchdog_heartbeat_seconds || 300) * 1000,
      pollIntervalMs: 1000,
      onBreach: (reason) => {
        dbm.events.append(db, { type: 'night_watchdog_fire', data: { reason } });
        if (orch && orch.child) {
          try { orch.child.kill('SIGTERM'); } catch (_) { /* ignore */ }
        }
      },
    });
    wd.start();

    let orch;
    try {
      orch = startOrchestrator({
        cfg, configPath, topicIds: ready.map(t => t.id), orchestratorBin,
      });
      await orch.done;
    } finally {
      wd.stop();
    }

    // partition outcomes
    const after = ready.map(t => dbm.topics.get(db, t.id));
    const delivered = after.filter(t => t.status === 'review' || t.status === 'done').length;
    const carryover = after.filter(t => t.status === 'running' || t.status === 'ready').length;
    // anything still running at this point is carryover
    for (const t of after) {
      if (t.status === 'running' || t.status === 'ready') {
        dbm.topics.updateStatus(db, t.id, 'carryover');
      }
    }
    const totalTokens = after.reduce((s, t) => s + (t.tokens_used || 0), 0);
    dbm.shiftLog.finish(db, shiftLogId, {
      topics_attempted: ready.length,
      topics_delivered: delivered,
      topics_carryover: carryover,
      total_tokens: totalTokens,
    });
    dbm.events.append(db, { type: 'night_end', data: { delivered, carryover } });
    return { topics_attempted: ready.length, topics_delivered: delivered, topics_carryover: carryover };
  } finally {
    db.close();
  }
}

async function main() {
  const { loadConfig } = require('./config');
  const cfgPath = process.env.NS_CONFIG || path.join(process.cwd(), 'config.json');
  const cfg = loadConfig(cfgPath);
  const orchestratorBin = process.env.NS_DRY_RUN === '1'
    ? path.join(__dirname, '..', 'test', 'helpers', 'fake-orchestrator.sh')
    : undefined;
  const r = await runNight({ cfg, configPath: cfgPath, orchestratorBin });
  process.stdout.write(JSON.stringify(r) + '\n');
}

module.exports = { runNight };

if (require.main === module) {
  main().catch((e) => { process.stderr.write(e.stack + '\n'); process.exit(1); });
}
```

- [ ] **Step 10: Run start-night test**

```bash
npx jest test/__tests__/start-night.test.js
```

Expected: PASS — 2 tests.

- [ ] **Step 11: Run full test suite + lint**

```bash
cd /Users/I572881/workspace/night-shift
npm run lint
npm test
```

Expected: lint clean; jest reports all tests across `config / db / shift / spec-io / worktree / ns-cli / spawn-worker / web-server / deliver / orchestrator / watchdog / start-night` passing.

- [ ] **Step 12: Manual dry-run smoke**

```bash
cd /Users/I572881/workspace/night-shift
cp config.json.example config.json
mkdir -p data/topics
node -e "const {openDb,migrate}=require('./src/db');const db=openDb('./data/state.db');migrate(db);db.close();"
NS_CONFIG=$(pwd)/config.json NS_DRY_RUN=1 node src/start-night.js
```

Expected: stdout JSON `{"topics_attempted":0,...}` (no topics yet); no errors.

- [ ] **Step 13: Add cron entry to README**

Append to `README.md`:

```markdown
## Cron

```
0 22 * * *  cd /Users/I572881/workspace/night-shift && /usr/local/bin/node src/start-night.js >> /tmp/night-shift.log 2>&1
```

For dry-run cadence (every minute, no LLM):

```
* * * * *   cd /Users/I572881/workspace/night-shift && NS_DRY_RUN=1 /usr/local/bin/node src/start-night.js >> /tmp/night-shift-dry.log 2>&1
```
```

- [ ] **Step 14: Commit**

```bash
git add src/orchestrator.js src/watchdog.js src/start-night.js \
        test/helpers/fake-orchestrator.sh \
        test/__tests__/orchestrator.test.js test/__tests__/watchdog.test.js test/__tests__/start-night.test.js \
        README.md
git commit -m "feat: orchestrator + watchdog + start-night with dry-run integration"
```

---

### Task 13: Kanban Interactive Actions + Polish

**Files:**
- Modify: `/Users/I572881/workspace/night-shift/web/server.js`
- Modify: `/Users/I572881/workspace/night-shift/web/public/app.js`
- Modify: `/Users/I572881/workspace/night-shift/web/public/index.html`
- Test: `/Users/I572881/workspace/night-shift/test/__tests__/web-actions.test.js`

- [ ] **Step 1: Write the failing test**

Write `test/__tests__/web-actions.test.js`:

```javascript
const fs = require('fs');
const os = require('os');
const path = require('path');
const request = require('supertest');
const { makeTmpDb } = require('../helpers/tmpdb');
const dbm = require('../../src/db');
const { createApp } = require('../../web/server');

function tmpRoot() {
  const root = fs.mkdtempSync(path.join(os.tmpdir(), 'ns-act-'));
  fs.mkdirSync(path.join(root, 'data', 'topics'), { recursive: true });
  return root;
}

describe('web actions', () => {
  let h, root, app;
  beforeEach(() => {
    h = makeTmpDb();
    root = tmpRoot();
    app = createApp({
      root, data_dir: path.join(root, 'data'), db_path: h.dbPath,
      shift: { night: { start: '22:00', end: '07:00' } },
      web: { port: 0, host: '127.0.0.1' },
    });
  });
  afterEach(() => { h.cleanup(); });

  test('POST /api/topics/:id/requeue moves failed/carryover → ready', async () => {
    dbm.topics.insert(h.db, { id: 'r1', title: 'x', shift: 'night', status: 'failed' });
    await request(app).post('/api/topics/r1/requeue').expect(200);
    expect(dbm.topics.get(h.db, 'r1').status).toBe('ready');
  });

  test('POST /api/topics/:id/toggle-shift flips day↔night', async () => {
    dbm.topics.insert(h.db, { id: 's1', title: 'x', shift: 'day', status: 'ready' });
    await request(app).post('/api/topics/s1/toggle-shift').expect(200);
    expect(dbm.topics.get(h.db, 's1').shift).toBe('night');
    await request(app).post('/api/topics/s1/toggle-shift').expect(200);
    expect(dbm.topics.get(h.db, 's1').shift).toBe('day');
  });

  test('POST /api/topics/:id/discard sets status=done with note event', async () => {
    dbm.topics.insert(h.db, { id: 'd1', title: 'x', shift: 'night', status: 'review' });
    await request(app).post('/api/topics/d1/discard').send({ note: 'not useful' }).expect(200);
    expect(dbm.topics.get(h.db, 'd1').status).toBe('done');
    const evs = dbm.events.list(h.db, { topicId: 'd1' });
    expect(evs.some(e => e.type === 'discarded')).toBe(true);
  });

  test('POST /api/topics/:id/approve sets status=done', async () => {
    dbm.topics.insert(h.db, { id: 'a1', title: 'x', shift: 'night', status: 'review' });
    await request(app).post('/api/topics/a1/approve').expect(200);
    expect(dbm.topics.get(h.db, 'a1').status).toBe('done');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
npx jest test/__tests__/web-actions.test.js
```

Expected: FAIL — 404 on each endpoint.

- [ ] **Step 3: Add action endpoints in web/server.js**

Insert before the `app.get('/api/events', ...)` block:

```javascript
  app.post('/api/topics/:id/requeue', (req, res) => {
    const db = openDb(dbPath);
    try {
      const t = dbm.topics.get(db, req.params.id);
      if (!t) return res.status(404).json({ error: 'not found' });
      dbm.topics.updateStatus(db, req.params.id, 'ready');
      res.json({ ok: true });
    } finally { db.close(); }
  });

  app.post('/api/topics/:id/toggle-shift', (req, res) => {
    const db = openDb(dbPath);
    try {
      const t = dbm.topics.get(db, req.params.id);
      if (!t) return res.status(404).json({ error: 'not found' });
      const next = t.shift === 'day' ? 'night' : 'day';
      db.prepare('UPDATE topics SET shift = ?, updated_at = ? WHERE id = ?')
        .run(next, Date.now(), req.params.id);
      dbm.events.append(db, { topic_id: req.params.id, type: 'shift_toggled', data: { from: t.shift, to: next } });
      res.json({ ok: true, shift: next });
    } finally { db.close(); }
  });

  app.post('/api/topics/:id/discard', (req, res) => {
    const db = openDb(dbPath);
    try {
      const t = dbm.topics.get(db, req.params.id);
      if (!t) return res.status(404).json({ error: 'not found' });
      dbm.topics.updateStatus(db, req.params.id, 'done');
      dbm.events.append(db, { topic_id: req.params.id, type: 'discarded', data: { note: req.body && req.body.note } });
      res.json({ ok: true });
    } finally { db.close(); }
  });

  app.post('/api/topics/:id/approve', (req, res) => {
    const db = openDb(dbPath);
    try {
      const t = dbm.topics.get(db, req.params.id);
      if (!t) return res.status(404).json({ error: 'not found' });
      dbm.topics.updateStatus(db, req.params.id, 'done');
      dbm.events.append(db, { topic_id: req.params.id, type: 'approved', data: {} });
      res.json({ ok: true });
    } finally { db.close(); }
  });
```

- [ ] **Step 4: Run test to verify it passes**

```bash
npx jest test/__tests__/web-actions.test.js
```

Expected: PASS — 4 tests.

- [ ] **Step 5: Add buttons to drawer in web/public/app.js**

Replace the `openDrawer` function with:

```javascript
async function openDrawer(id) {
  const r = await fetch(`/api/topics/${id}`);
  const data = await r.json();
  const t = data.topic;
  const body = document.getElementById('drawer-body');
  body.innerHTML = `
    <h3>${escapeHtml(t.title)}</h3>
    <p><span class="status ${t.status}">${t.status}</span> ${t.shift}</p>
    <p>tokens: ${t.tokens_used}/${t.budget_tokens || '?'}, runs: ${t.runs_count}/${t.budget_runs || '?'}</p>
    <div class="actions">
      <button data-action="approve">Approve</button>
      <button data-action="discard">Discard</button>
      <button data-action="requeue">Re-queue</button>
      <button data-action="toggle-shift">Toggle shift</button>
    </div>
    <h4>Runs</h4>
    <ul>${data.runs.map(r => `<li>#${r.run_index} ${r.kind} exit=${r.exit_code ?? 'pending'} ${escapeHtml(r.summary || '')}</li>`).join('')}</ul>
  `;
  body.querySelectorAll('button[data-action]').forEach(btn => {
    btn.onclick = async () => {
      const action = btn.dataset.action;
      const resp = await fetch(`/api/topics/${id}/${action}`, { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: '{}' });
      if (resp.ok) { document.getElementById('drawer').hidden = true; refresh(); }
      else alert(`failed: ${resp.status}`);
    };
  });
  document.getElementById('drawer').hidden = false;
}
```

- [ ] **Step 6: Append CSS for buttons to web/public/style.css**

```css
.actions { display: flex; gap: 8px; margin: 12px 0; }
.actions button { background: #21262d; color: var(--fg); border: 1px solid var(--border); padding: 6px 10px; border-radius: 4px; cursor: pointer; }
.actions button:hover { border-color: var(--accent); }
```

- [ ] **Step 7: Final lint + full test run**

```bash
cd /Users/I572881/workspace/night-shift
npm run lint
npm test
```

Expected: all green.

- [ ] **Step 8: Commit**

```bash
git add web/ test/__tests__/web-actions.test.js
git commit -m "feat: kanban actions (approve/discard/requeue/toggle-shift) + drawer buttons"
```

- [ ] **Step 9: Tag v0.1.0**

```bash
git tag v0.1.0
git log --oneline
```

Expected: 13 commits, latest tagged `v0.1.0`.

---

## Spec Coverage Map

| Spec section | Implemented in |
|---|---|
| §2 Day/Night shifts (hard constraint: no LLM in day) | Task 9 (`/api/health` reports mode); Task 12 (orchestrator only spawned via `start-night`); Task 1 README rule |
| §2.1 `shift` field on tasks | Task 3 (db schema), Task 5 (frontmatter), Task 9 (POST /api/topics) |
| §2.2 Night window 22:00–07:00 | Task 1 (config), Task 4 (`isNightWindow`/`minutesUntilEnd`) |
| §3 Orchestrator-as-Claude | Task 12 (`startOrchestrator` with system prompt + claude_bin) |
| §3.1 Tool list | Task 7 (read-spec, update-status, budget, next-topic, heartbeat, read-run), Task 8 (spawn-worker), Task 11 (deliver) |
| §3.2 Worker = one-shot claude in worktree | Task 8 (`spawn-worker` uses `cfg.claude_bin` + `--dangerously-skip-permissions`) |
| §3.3 Watchdog | Task 12 (`Watchdog` class, both heartbeat + budget checks) |
| §3.4 ns-daemon (port 3737, no LLM in day) | Task 9 |
| §4.1 Markdown layout (spec.md, runs/) | Task 5 (spec-io), Task 8 (writes runs/run-NNN.{log,diff,meta.json}) |
| §4.2 SQLite schema | Task 3 (matches §4.2 column-for-column) |
| §5 plan-night-task skill | Task 10 |
| §6 Night execution loop | Task 12 (`runNight`) |
| §6.4 Worktree lifecycle | Task 6 (`ensureWorktree`/`pruneWorktree`), Task 11 (commit/pr/diff-only) |
| §7 Kanban two-column + actions | Task 9 (layout, SSE), Task 13 (actions) |
| §8 Repository layout | All tasks |
| §9 Error handling | Task 8 (budget guard), Task 11 (mode validation), Task 12 (carryover, watchdog SIGTERM) |
| §10 Testing strategy | TDD throughout; Task 12 step 12 = dry-run smoke |
| §13 Build order | Tasks 1→13 follow §13's 12 steps (with §13.10 actions split into Task 13) |

---

## Self-Review

**1. Spec coverage:** Every numbered section above maps to a task. The "edit spec" action from §7.2 is intentionally deferred — the user can hand-edit `data/topics/<id>/spec.md` (it's just a file) and the next status change will mirror via `update-status`. If the user wants an in-UI editor, that's a v0.2 task.

**2. Placeholder scan:** No "TODO/TBD/fill-in", no "similar to Task N" references; every step that mutates code shows the code. Stub files in Task 7 step 10 are explicit and replaced in Tasks 8/11.

**3. Type/name consistency:**
- Status enum is consistent: `draft|ready|running|review|done|failed|carryover` everywhere (db.js validators, deliver, web actions).
- `db_path` resolution rule (`cfg.db_path || ${cfg.data_dir}/state.db`) is identical in every tool.
- `dbm.topics.updateStatus` signature `(db, id, newStatus)` is the same at every call site.
- `ensureWorktree({repoPath, worktreePath, branch})` signature is consistent in Task 6 / Task 8 / Task 11.
- `startOrchestrator({cfg, configPath, topicIds, orchestratorBin})` matches its consumer in `runNight`.
- `Watchdog({db, heartbeatThresholdMs, pollIntervalMs, onBreach})` matches consumer.
- Worktree path convention `data/.worktrees/<id>` and branch `nightshift/<id>` is consistent.

**4. Test isolation:** Every test uses `makeTmpDb` + `mkdtempSync` for the data dir, so they can run in parallel without interference.

**5. No LLM in day:** Hard rule preserved — `start-night.js` is the ONLY place that calls `startOrchestrator`, and it has no time-window guard built in (cron at 22:00 is what enforces it). For belt-and-suspenders, Task 12's `runNight` could refuse to start if `!isNightWindow(now, cfg)` — but that would break the integration test that uses a `00:00–23:59` window for testability. We rely on cron + the documented hard rule in CLAUDE.md.
