# Dark factory: preliminary build plan

Status: preliminary, 2026-10-07. Built from the seven research docs in `research/`. Decisions marked **OPEN** need the owner before the phase that depends on them.

## 1. Agreed design

### Core abstraction: the goal
Every unit of work is a **goal**: objective, machine-checkable success criteria, constraints, allowed outcomes, external state, and an escalation rule. Goals can spawn sub-goals (a DAG). The four use cases are goal types:

| Type | Example | Notes |
|---|---|---|
| `update` | Upstream changed (library, CLI, API, model) | The harness-compat case is an example, not the spec |
| `solve` | Long-running hard problem | Needs a progress signal and stall detection |
| `compare` | Try N options, pick the best | Gates, then metrics, then judged criteria |
| `build` | Greenfield build from a conversational vision | Factory proposes definition of done; user owns acceptance; rejection spawns a new round |

### Allowed outcomes
`adapt`, `pin/hold`, `constrain`, `work around`, `escalate`. Pinning is a successful outcome, recorded in the **constraints registry** with reason, evidence, upstream link and a revisit trigger, and re-tested on a loop so pins cannot silently rot (docs 05).

### Autonomy rule
Proceed when one option is clearly dominant. Auto-merge on green CI plus canary. Escalate only for architectural/design changes, genuine forks, and real ambiguity. Autonomy is set per repo in `.factory/policy.yaml`. Escalation is asynchronous: park the question, continue unblocked goals.

### Architecture (from docs 02, 03, 04, 07)
- **Supervisor:** plain code (not an LLM), one daemon on SQLite (WAL) plus systemd timers. Owns the state machine, leases, budgets and stall detectors. Agents propose; the supervisor disposes. Temporal/Redis deferred.
- **Execution:** `claude -p --bare` or the Agent SDK, API-key auth, in rootless Podman plus a git worktree per attempt, with turn and USD caps on every run. The factory keeps its own state because sessions are local files and compaction is lossy.
- **Roles:** planner, worker (fresh context, one task, cannot edit criteria), verifier (separate context, sole writer of `satisfied`). Models per doc 02: Opus 5.5 plans and verifies, Sonnet 5.5 works, Haiku 5.5 for cheap subagents.
- **Verifier:** separate repo, runner label, token and required-status check. Deterministic checks first (contract, replay, held-out tests, surface diff), LLM judges last (pairwise, order-swapped, different tier or family where possible). One `VerifyRequest/VerifyResult` interface for all four goal types (docs 03).
- **Anti-gaming:** protected paths for criteria/tests, test-diff policy, mutation testing on changed lines, quarantine-by-builder counts as an integrity violation.
- **Detection:** self-hosted Renovate or the factory's own watchers for detection and PR mechanics only. The decision procedure lives in the factory (docs 01, 05). The cooldown is enforced in the factory's resolver as well as natively.
- **Escalation:** a `question` table (context, options, recommendation, default-after-timeout) with ntfy, a web answer page, and PR comments as transports. The multi-harness messaging tool becomes one more adapter (docs 07).

## 2. Phases

Each phase ends with something runnable and a reboot/crash test where state is involved.

**Phase 0: Skeleton (no agent).** SQLite schema (doc 04), goal state machine, event log, one release poller for one dependency, generated `GOAL.md`/`PROGRESS.md` views. Exit: a goal moves through states from a poll event; `kill -9` mid-transition recovers with no duplicate effects.

**Phase 1: Sandbox runner.** Rootless Podman plus worktree launcher for `claude -p` with JSON output, transcript and artifact capture, turn/USD caps, lease and heartbeat. Exit: an attempt runs, is killed, and resumes from its last commit.

**Phase 2: Update goals end to end (use case 1).** Constraints registry and `policy.yaml` schemas with validation; canary in a sandbox; decision procedure (adapt/pin/constrain/work around/escalate); bisect worker with cost caps; PR creation, green/red read, auto-merge per policy; re-test loop for pins. First targets: one ordinary library repo (FastAPI-style pin) and one CLI-contract repo. Exit: a deliberate bad upstream release produces a pin with evidence and a later fixed release removes it automatically.

**Phase 3: Verifier and conformance.** Separate verifier repo and runner. Surface extraction and diff library (CLI `--help`, event schemas, Python public API, lockfile). Conformance layout for the multi-harness example: per-harness consumer contracts, recorded fixtures with fake-CLI shims, live smoke on new versions only. Integrity checks (protected paths, test-diff policy, mutation on changed lines). Exit: planted test-weakening is caught; planted breaking CLI change is classified `breaking`.

**Phase 4: Escalation.** Question table and service, ntfy plus web answer page, timeout/default executor, PR-review-as-answer ingestion. Read-only dashboard (goals, active pins, median pin age, open questions). Exit: a forced fork pauses one goal and others continue; a reply resumes it.

**Phase 5: Solve goals (use case 2).** Planner/worker/verifier loop with fresh-context iterations, progress metric, stall/loop/drift detectors, response ladder (retry, replan, allowed outcome, escalate). Exit: a multi-hour goal survives a reboot and stalls are detected without self-reports.

**Phase 6: Compare goals (use case 3).** N sandboxes with resource accounting, scoring schema (gates, Pareto metrics, pairwise judges), dominance pre-check, successive halving, comparison report delivered as a choose-question when no option dominates. Exit: a known-better option wins reliably across reruns.

**Phase 7: Build goals (use case 4).** Conversation-to-spec template and stop-asking rule, factory-proposed definition of done with a separate agent writing acceptance tests, `goals.yaml` sub-goal DAG, walking-skeleton first milestone, milestone verification, traceability gate against scope creep, acceptance page, rejection loop that classifies feedback and feeds a "DoD lessons" file. Exit: a small greenfield project is built, rejected once with feedback, and rebuilt in a follow-up round.

Optional later: microVM isolation tier, Managed Agents pilot behind an executor interface, durable engine if hand-rolled resume gets painful.

## 3. Open decisions for the owner

1. **OPEN: Forge and CI.** GitHub with ephemeral JIT runners, local Forgejo, or both. Decides the Phase 2 CI path (docs 07).
2. **OPEN: Auth.** Doc 02 recommends API-key billing for an always-on factory. Whether an always-on personal factory on a Max subscription is permitted is unverified. Confirm the intended billing before Phase 1.
3. **OPEN: Host capacity.** KVM available (microVM tier)? RAM for N concurrent sandboxes? How many parallel options per compare goal?
4. **OPEN: Phone/remote reachability.** VPN (e.g. Tailscale) or public tunnel, for the answer page and ntfy actions.
5. **OPEN: Messaging tool API.** Inbound replies, buttons, threading, auth; needed to use it as an escalation transport.
6. **OPEN: Judge independence.** Builder is Claude; accept a different Claude tier as judge, or add a non-Anthropic model for judged criteria?
7. **OPEN: Upstream comments.** May the factory open or comment on upstream issues itself? Default in the research: draft only, human approves.
8. **OPEN: Policy on repo-bundled constraints** for upstream-controlled installs (the user's machine controls the CLI version): ship the bad-version list with the tool or fetch it at runtime.

## 4. Second-pass research (highest risk first)

Sourcing in the first pass was thin: most researchers ran 3 to 14 tool calls, and several could not fetch primary documentation. Verify before depending on these:

1. **Real event schemas of the harness CLIs** (Claude Code stream-json, Codex, Kimi, OpenCode): headless modes, output formats, exit codes, session files. Doc 03 did not verify any of these. This is the contract for use case 1's main example.
2. **Agent SDK / `claude -p` details:** session resume behavior, `--bare`, budget flags, whether `claude-code-action` works on self-hosted runners, whether API credits cover Managed Agents (doc 02 UNVERIFIED items). Re-check model IDs and prices on the public pricing page; doc 02 took them from a cached reference with no public URL.
3. **Renovate/Dependabot primary docs:** `packageRules`, `allowedVersions`, `lockFileMaintenance`, cooldown defaults (the 3-day Dependabot default is single-source), the PyPI 14-day upload change, crates/Go/Homebrew detection (doc 05).
4. **Forgejo ephemeral runner support, ntfy action syntax, Cloudflare Tunnel with GitHub webhooks** (doc 07).
5. **Primary sources for StrongDM Software Factory and Stripe Minions** (doc 01), and Huntley's original Ralph write-up (doc 04).
6. **Agent SDK resume/background features vs the hand-rolled supervisor:** how much can the SDK replace (doc 04 open question).
7. **Spec formats primary sources:** EARS, PRD/ADR templates, Spec Kit/Kiro/BMAD current state; evidence on autonomous-build failure rates (doc 06).
8. **DBOS SQLite production-readiness** and stall-threshold calibration (needs data from Phase 5 runs).

## 5. Known tensions between the docs

- Doc 01 says adopt Renovate; doc 05 says do not use it as the brain. Resolution: detection and PR mechanics only; decisions in the factory.
- Doc 05 says webhooks are unnecessary for registries; doc 07 says poll first, webhooks later. Compatible: poll everywhere, add webhooks only for latency-sensitive triggers.
- Docs 03 and 04 both require the verifier to be independent of the worker. Doc 03's independence (own repo, runner, token) is the stronger form; adopt it.
- Doc 02's Managed Agents pilot conflicts with the "own state" principle only if sessions become the source of truth. Keep the executor interface thin and the SQLite store authoritative.
