# 04 - State and Long-Running Work: making autonomous agent work durable and convergent

Researched 2026-10-07. Depth note: this was a compact pass. Anthropic primary sources were fetched directly; Ralph-loop and durable-engine material is mostly secondary or vendor-authored and is flagged.

## Summary

- Agents lose all memory between sessions, so durable state must live outside the model: a machine-readable goal store plus git. Anthropic's own long-running harness uses exactly this (feature list JSON, progress log, git commits, init script).
- Fresh-context-per-iteration (Ralph loop) fixes context rot but moves risk into the task list, the on-disk state and the completion check. Its failure modes are premature DONE, phantom completion, livelock, and tests-pass-but-wrong.
- Fix the above by separating roles: planner writes criteria, worker implements, an independent verifier (not the worker) flips "passes". Completion is decided by machine-checkable commands, never by agent assertion.
- For one home server, use SQLite (WAL) plus systemd plus idempotent step design. A full engine (Temporal) is overkill; DBOS-style "library over a DB" is the upper bound worth considering.
- Detect stalls with progress signals (verified-criteria count, diff-of-state hash, repeated-error fingerprints), not with agent self-reports. Budget and iteration caps are the last-resort backstop.
- A concrete schema and state machine are proposed below.

## Findings

### 1. Anthropic guidance on long-running harnesses

Source: "Effective harnesses for long-running agents", Anthropic Engineering, 2025-11-26 (https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents).

- Problem: sessions start with no memory; compaction alone was insufficient for long projects.
- Initializer/worker split: the first session creates `init.sh`, a progress log, a feature list, and an initial git commit. Later "coding" sessions do one feature each and leave the repo in a mergeable clean state.
- Feature list is JSON, every item initially `failing`; workers may only change the `passes` field. Chosen because models overwrite JSON less readily than Markdown.
- Startup routine each session: check cwd, read git log and progress file, read feature list, run `init.sh`, run a basic end-to-end test before new work.
- Observed failures and fixes: declaring victory early (feature list, one feature at a time); undocumented progress or bugs (commits plus notes, smoke test at start); marking done without verification (end-to-end browser automation as a human would); time wasted on how to run the app (`init.sh`).
- Open question stated by the authors: single general agent vs specialized testing/QA/cleanup agents is unresolved.

Source: "Effective context engineering for AI agents", Anthropic Engineering, 2025-09-29 (https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents).

- Compaction: summarize near the limit and continue; tune for recall first, then precision. Over-aggressive compaction can drop subtle context that matters later.
- Tool-result clearing is the "safest lightest touch" form of compaction.
- Structured note-taking (to-do lists, NOTES.md) persists outside the window; a file-based memory tool exists in beta.
- Sub-agents use clean windows and return condensed summaries (about 1-2k tokens), but exploration cost is high (tens of thousands of tokens each).

### 2. Ralph loop (fresh context per iteration)

Secondary sources only; Huntley's original post was not reached (UNVERIFIED as to his exact wording). See https://futureagi.com/blog/ralph-loop-production-failure-modes/ and https://futureagi.com/blog/loop-engineering/ralph-loop/ (accessed 2026-10-07, dates not captured).

- Mechanism: the same prompt in a shell loop, new context each pass, progress persisted in files and git.
- Failure modes reported:
  - Nondeterministic false negatives: a pass concludes (e.g. after a ripgrep) that something is unimplemented and duplicates it. Needs an idempotent prompt ("do not assume not implemented").
  - Livelock via scope creep: list never shrinks. Fix: fixed definition of done; extras go to a non-counting backlog.
  - Premature DONE with vague completion condition. Fix: mechanical check (zero open items AND tests pass).
  - Phantom completion: diff is only a checkbox flip. Fix: require real diff and green tests per pass.
  - Tests pass but feature wrong; no ground truth in the loop. Fix: grader vs spec or human checkpoint.
  - Compounding small errors; evolving specs not handled; high token cost.
  - Bad input: most failures trace to the task list; human-written lists are most reliable.
- Takeaway: Ralph suits narrow tasks with objective stop criteria. For the factory it is the inner worker loop, wrapped by a verifier and a stall detector, not the whole system.

### 3. Role splits

- Anthropic's harness has initializer + worker; it flags a dedicated tester/QA as an open question. Community evidence above supports an independent verifier because workers mark their own work done prematurely (Anthropic, same post).
- Recommended: planner (decomposes, writes criteria as runnable checks), worker (one task, fresh context, cannot edit criteria), verifier (separate context, runs criteria, sole writer of `satisfied`), plus a supervisor process (non-LLM code) owning the state machine, budgets and stall detection.

### 4. Durable execution options for one server

All comparison material found is vendor or third-party-derived; no independent 2026 benchmark found (UNVERIFIED claims about latency).

- DBOS Transact: library (TS, Python, Java, Go) checkpointing steps into Postgres; Python quickstart uses SQLite but recommends Postgres for production; Go gained SQLite in June 2026. Sources: https://dbos.dev/dbos-transact, https://dbos.dev/blog/new-in-dbos-june-2026, https://www.mintlify.com/dbos-inc/dbos-transact-py/quickstart.
- Temporal: separate server cluster; strongest maturity and scale, heaviest ops (DBOS claim: https://dbos.dev/compare/dbos-vs-temporal, vendor).
- Restate: journal/replay like Temporal, lighter, still a separate service, shorter production record (https://unmeshed.io/blog/temporal-alternatives, third-party).
- Simple SQLite + systemd: no new service; you hand-write idempotency and resume logic.
- Key point: LLM agent steps are non-deterministic, so replay-based engines (Temporal/Restate determinism model) fit poorly unless every LLM call is a recorded activity. Checkpoint-at-step-boundary (DBOS style or hand-rolled) is the natural model. (Author's reasoning, not sourced.)

### 5. Crash/reboot recovery and idempotency (design reasoning, not sourced)

- Persist before acting: write intent row `attempt(started)` before launching a worker; write result after.
- Work happens in a git worktree/branch per attempt; commit = checkpoint. Recovery = find attempts in `running` with no live PID/lease, mark `interrupted`, resume from last commit on the attempt branch or discard worktree.
- Leases with heartbeat (worker updates `lease_expires`); supervisor reclaims expired leases on boot.
- Side effects (push, PR, deploy, comments) must be idempotent via a dedupe key (e.g. goal_id+step+hash) recorded in an `effects` table before the call and confirmed after; check remote state first on retry.
- systemd: supervisor as a service with `Restart=always`, `After=network-online.target`; runner/worker processes as child units so a reboot restarts only the supervisor, which then resumes work.

### 6. Stall, loop and drift detection

Signals (all computed by the supervisor, none self-reported):
- Progress metric: count of criteria verified passing, monotone-tracked; no increase in N attempts = stall.
- State fingerprint: hash of (git tree, criteria results); identical across attempts = loop.
- Error fingerprint: normalized failing test/command output repeated K times = stuck.
- Diff churn: files touched repeatedly with net-zero diff (flip-flopping).
- Drift: diff touches paths outside the goal's allowed scope, or changes criteria/constraints files (hard block).
- Budget burn vs progress ratio; wall-clock and token caps.
- Response ladder: retry with fresh context, then change strategy (planner re-plans), then try allowed outcome (pin/constrain/work around), then escalate asynchronously.

### 7. Compaction and memory pitfalls

- Summaries drop subtle constraints; keep constraints and criteria in files re-read at each start, never only in conversation (Anthropic, 2025-09-29).
- Stale or contradictory notes: progress logs are append-only; keep a curated `STATE.md` summary regenerated from DB, and treat the DB as truth.
- Storing in Markdown invites rewriting; use JSON/DB for fields with semantics (Anthropic, 2025-11-26).
- Memory without pruning degrades retrieval (secondary source, https://www.theneuron.ai/explainer-articles/anthropic-just-changed-the-rules-for-working-with-ai-and-prompting-isnt-the-main-game-anymore, weak).

## Implications for the dark factory

1. Store: SQLite (WAL, single supervisor writer) as source of truth, plus git repos for work product. Generate human-readable `GOAL.md`/`PROGRESS.md` views from the DB into each repo; workers read them but cannot write criteria. Reconsider DBOS only if you outgrow hand-rolled resume; skip Temporal.
2. The supervisor is plain code (Python/Go), not an LLM. It owns transitions, leases, budgets, and detectors. Agents propose; the supervisor disposes.
3. Only the verifier role can mark a criterion `passing`, and only by running its stored command in a clean checkout on the CI runner. The worker's claims are never trusted.
4. Each attempt = fresh context, new worktree/branch, startup routine copied from Anthropic (read state, run init/smoke test, then work on one task).
5. Criteria and constraints files are write-protected from workers (path allowlist check in the supervisor; fail attempt on violation).
6. Escalation is a row (`question`) plus parking the goal node `blocked_on_human`; the scheduler continues other unblocked nodes. A reply row unblocks it.
7. Reboot test is a first-class acceptance test: kill -9 supervisor and reboot mid-attempt in CI; goal must resume without duplicate side effects.

### Proposed schema (SQLite)

```
goal(id, parent_id NULL, kind[update|solve|compare|build], title, objective,
     state, state_reason, priority, budget_tokens, budget_usd, budget_wall_s,
     spent_tokens, spent_usd, created_at, updated_at, satisfied_at,
     escalation_rule, allowed_outcomes JSON)
criterion(id, goal_id, description, check_cmd, check_env, kind[test|lint|bench|manual_gate],
          last_result[pass|fail|unknown], last_checked_at, last_commit, required BOOL)
constraint(id, goal_id NULL /*NULL=global*/, text, check_cmd NULL, source, added_by, active BOOL)
goal_edge(goal_id, depends_on_goal_id, type[hard|soft])      -- DAG; cycle check on insert
attempt(id, goal_id, role[planner|worker|verifier], state[running|ok|failed|interrupted|killed],
        branch, worktree, base_commit, end_commit, lease_owner, lease_expires, pid,
        tokens, usd, started_at, ended_at, outcome_summary, error_fingerprint, state_fingerprint)
decision(id, goal_id, attempt_id, kind[adapt|pin|constrain|workaround|escalate|replan|other],
         rationale, alternatives JSON, reversible BOOL, created_at)     -- append-only
checkpoint(id, goal_id, git_ref, db_snapshot_path NULL, note, created_at)
event(id, goal_id, attempt_id, ts, type, payload JSON)                  -- append-only progress log
question(id, goal_id, text, options JSON, recommended, state[open|answered|expired], answer, asked_at, answered_at)
effect(dedupe_key PK, goal_id, attempt_id, kind, status[intent|done|failed], result JSON)
```

### State machine (goal)

```
created -> planning -> ready -> running -> verifying -> satisfied
                         ^        |  ^         |
                         |        v  |         v (criteria fail)
                         |     stalled/ -------> running (retry, new strategy)
                         |     failed_attempt
running|ready -> blocked_on_dep (hard edge unmet; auto-returns to ready)
running|ready|planning -> blocked_on_human (open question; returns to prior state on answer)
any non-terminal -> held (pin/hold outcome, with revisit condition) -> ready
any non-terminal -> abandoned (budget exhausted, human cancel, superseded; terminal, reason required)
satisfied: terminal, re-opened only by new goal (regression = new goal linked via parent_id)
```
Transitions are made only by the supervisor in a single transaction that also writes an `event`. `satisfied` requires all `required` criteria `pass` at the same `end_commit` with a verifier attempt. Parent goals become `satisfied` when children are satisfied and own criteria (integration checks) pass; this gives milestone verification for greenfield builds.

Decomposition: planner emits child goals plus edges plus criteria as proposals; supervisor validates DAG, scope and budget before activating. Scheduler runs any `ready` node with all hard deps satisfied, bounded by concurrency and per-repo locks.

## Open questions

- Does Huntley's original Ralph write-up add failure modes beyond the secondary sources? (Not reached.)
- Agent SDK/Claude Code built-in session resume, background tasks and routines: how much of the supervisor can they replace? (Not researched here; overlaps other researchers' topics.)
- Verifier independence: same model family may share blind spots; is a differently prompted or held-out test set enough?
- Checkpoint granularity: per-commit only, or also snapshot of agent scratch notes?
- Whether DBOS SQLite support in Python/TS is production-grade (docs say use Postgres for production).
- Calibrating stall thresholds (N, K) needs empirical data from early runs.

## Sources

- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents (2025-11-26)
- https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents (2025-09-29)
- https://futureagi.com/blog/ralph-loop-production-failure-modes/ (secondary, undated)
- https://futureagi.com/blog/loop-engineering/ralph-loop/ (secondary, undated)
- https://dbos.dev/dbos-transact ; https://dbos.dev/blog/new-in-dbos-june-2026 ; https://dbos.dev/compare/dbos-vs-temporal (vendor)
- https://www.mintlify.com/dbos-inc/dbos-transact-py/quickstart
- https://unmeshed.io/blog/temporal-alternatives (third-party)
- https://www.theneuron.ai/explainer-articles/anthropic-just-changed-the-rules-for-working-with-ai-and-prompting-isnt-the-main-game-anymore (weak secondary)
