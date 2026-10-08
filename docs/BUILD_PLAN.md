# Dark factory: build plan

Status: v4, 2026-10-08. This file is only phases, exit tests, decisions and open items. The design is in `DESIGN.md`; the evidence is in `EVIDENCE.md` (cited as E1, E2, ...); worked examples are in `examples/`. `[U]` marks a guess.

Principles for the phases: each one ends with something runnable; every phase that touches state has a crash test; the first end-to-end loop runs as early as possible with the simplest version of each part; nothing auto-merges before the verifier exists.

## Phases

**Phase 0: Skeleton and inspection (no agent).**
Deliverables: SQLite schema (goal, criterion, constraint, goal_edge, attempt with `action_fingerprint`, decision, checkpoint, event, question, effect); goal state machine with transitions written in one transaction with their event; one release poller for one dependency; generated `GOAL.md` and `PROGRESS.md` views; a `factory` CLI with `ls`, `show <goal>`, `events <goal>` and `kill-switch on|off`; the kill-switch flag checked before any lease.
Exit: a poll event moves a goal through its states; `kill -9` mid-transition and restart leaves exactly one transition and one event (no duplicates, no gaps); the CLI shows the resulting state.

**Phase 1: Sandbox runner and key proxy.**
Deliverables: `keyproxy` (holds provider and forge credentials; injects them; per-attempt egress allowlist with request-level checks; enforces USD caps from the ledger; logs every request). Container launcher in the Docker Linux VM with a worktree per attempt, VM CPU and memory limits set, two Docker networks (`worker-net`, whose only egress is `keyproxy`; `verifier-net`, with no route from `worker-net`), arm64 image check for every tool image, bind-mount throughput measured. `claude -p --bare` runner: stream-json capture, caps re-issued per run, authoritative ledger, lease and heartbeat, pinned model ID and Claude Code version with a captured stream-json fixture, SIGINT-then-SIGTERM stop, deny-rule and hook startup self-test, environment hardening (updaters off, explicit permission mode, per-run config and home), live stream-json parsing for later detectors. A billing test that records how the chosen account bills `claude -p` versus SDK calls.
Exit: an attempt runs, is killed, and resumes from its last commit; a token planted in the worktree is unusable for any outbound call; a planted fetch from a worker container to `verifier-net` and to a non-allowlisted host both fail and are logged; measured concurrency (how many sandboxes run a real test suite at once) is recorded and replaces the `[U]` estimate; the billing result is recorded for the auth decision.

**Phase 2: Thin end-to-end slice for `update` goals (PR only, human merges).**
Deliverables: minimal constraints registry (pin and constrain entries with reason, evidence, upstream link, revisit trigger, max age) and minimal `policy.yaml`, both schema-validated; canary = install the candidate in a sandbox and run the repo's tests with the strict paired rerun rule; decision procedure limited to adopt, pin or constrain, and escalate (escalate = PR labelled `needs-human` and the goal parks); PR creation on Forgejo with a dedicated bot token (fallback: GitHub with JIT runners); the ledger reports cost per goal. Prerequisite check: Forgejo ephemeral runners work and the bot token's PRs trigger CI; if not, switch to the fallback and record it.
Exit: a deliberately bad upstream release produces a PR proposing a pin with evidence attached; a later fixed release produces a PR removing it; both appear within one poll cycle; cost per goal is in the ledger; the cost envelope in `DESIGN.md` section 5 is revised from actuals.

**Phase 3: Minimal verifier, auto-merge, self-change canary.**
Deliverables: the verifier as its own service on `verifier-net` with its own forge token, runner label and required-status check; deterministic checks only: re-run the canary in a clean checkout, protected-path and test-diff integrity, held-out tests, lockfile and transitive diff, liveness; the `VerifyRequest/VerifyResult` interface; the supervisor holds the merge credential and auto-merges only on green CI plus canary plus verifier pass plus size and sensitive-path limits; post-merge canary with a supervisor-generated revert PR on failure; the self-change canary: a replay set of recorded goals that every change to factory prompts, model IDs, effort settings or the pinned Claude Code version must pass, per model, before deploy.
Exit: planted test-weakening is caught; a planted verifier read, `git log --all` and a network fetch of verifier artifacts all fail; a planted good release auto-merges; a planted post-merge failure produces a revert PR; a deliberately broken prompt change is blocked by the self-change canary.

**Phase 4: `update` goals complete.**
Deliverables: adapt with a bounded budget, automatic fall-through to pin or constrain, and the four-state check; bisect with boundary confirmation and cost caps; surface extraction and diff library (CLI `--help`, event schemas, Python public API, lockfile) and the consumed-symbol check in the canary; contract fixtures with fake shims and a live smoke test within 24 hours of each new stable release; the pin re-test loop with revisit triggers and max age; per-ecosystem constraint syntax; optional Renovate for detection; metrics: adapt success rate, revert rate per agent and task class against a human control, flake rate per test, pin age and count; a replay set of real breaking updates from the target repos' history.
Exit: the factory's adapt rate is measured on the replay set; a planted upstream flag removal is classified `breaking`; a planted new enum value is `ambiguous` or `breaking`; exit-0-without-output fails liveness; a pin past its max age escalates rather than silently renewing.

**Phase 5: Escalation.**
Deliverables: question table and service; ntfy plus a web answer page over Tailscale; strict answer parsing with capped re-asks and timeout defaults; PR-review-as-answer ingestion; the upstream-draft approval flow (draft issue or comment parked as a question); a read-only dashboard (goals, active pins, median pin age, open questions, spend by goal type).
Exit: a forced fork parks one goal while others continue, and a reply resumes it; an unparseable reply is rejected; a drafted upstream comment is not posted until approved.

**Phase 6: LLM judges.**
Deliverables: Claude judge and non-Claude juror behind one judge interface; pairwise, order-swapped; comment-stripped diff `[U]`; a calibration set of 50 or more human-labelled examples; kappa gate per judge; `judge_model_id` pinned and recorded; judged criteria advisory until calibrated, blocking only through a panel that includes the non-Claude juror.
Exit: both judges pass calibration; sibling bias is measured locally (Claude vs non-Claude on the same pairs) and recorded; a planted misleading-comment pair does not flip the verdict.

**Phase 7: `solve` goals.**
Deliverables: planner, worker and verifier loop with fresh-context iterations (per-model config); progress metric; stall, loop and drift detectors with the OpenHands-derived defaults plus the duplicate-call metric; horizon budgeting at 30 to 60 minutes per leaf `[U]`, with actual durations logged; response ladder: retry, replan, allowed outcome, escalate.
Exit: a multi-hour goal survives a reboot; a planted loop is detected without self-reports; LLM-only grading never flips a criterion; detector thresholds are re-tuned from the logs and recorded.

**Phase 8: `compare` goals.**
Deliverables: N sandboxes (default 3, max 4 `[U]`, bounded by the Phase 1 measurement) with equal recorded resources; scoring schema (gates, Pareto metrics, pairwise judges); dominance pre-check; hybrid ranking (tests plus verifier) as the default behind a measurement; Copeland aggregation as a candidate; comparison report delivered as a choose-question when no option dominates.
Exit: selector accuracy on a planted set that includes near-tie and polish-bias pairs; gate false-positive rate measured; a known-better option wins reliably across reruns. Deferred: best-of-N for `solve`, evolutionary mode (numeric fitness only).

**Phase 9: `build` goals.**
Deliverables: ambiguity detection before elicitation with an explicit style and UX checklist (models almost never ask about style, E17); conversation-to-spec template; stop-asking rule (about 5 questions `[U]`); factory-proposed definition of done with a separate agent writing acceptance tests, validated by mutants, a second generation and human sign-off on expected values; `goals.yaml` sub-goal DAG; walking-skeleton first milestone; milestone verification; traceability drift metric judged by the verifier; acceptance page; rejection loop (about 3 rounds `[U]`) feeding a lessons file that is treated as an untested hypothesis (E16); first-pass acceptance rate recorded.
Exit: a small greenfield project is built, rejected once with feedback, and rebuilt in a follow-up round; the expectations are modest (full autonomous builds are unsolved even at the frontier, E16).

**Optional later:** microVM tier (needs a different host), Managed Agents pilot behind the executor interface, evolutionary mode, a durable-execution engine if hand-rolled resume becomes painful.

## Factory-level definition of done (proposal, `[U]`)

The factory is trusted to run unattended on `update` goals when, over 30 consecutive days across at least two repos: at least 20 goals complete; the revert rate is at or below the human control for the same repos; there are no unresolved integrity violations; every escalation was answered without a silent default; and spend stayed within the per-day cap. Each later goal type earns the same trust separately.

## Decisions

### Decided by the owner (2026-10-08)
- Auto-merge gate: green CI + canary + verifier pass + size and sensitive-path limits; otherwise escalate.
- Forge: local Forgejo; fallback GitHub with JIT ephemeral runners.
- Host: Apple Silicon Mac, 32 GB or more, Docker Linux VMs.
- Reachability: Tailscale or another VPN.
- Chat transports: later, as adapters; ntfy, web page and PR comments first.
- Upstream issues and comments: draft only; the owner approves.
- Known-bad versions in downstream tools: bundled per release from the registry; no runtime fetch.
- A non-Claude juror is wanted from the start; the provider is still open (below).
- Implementation stack: TypeScript on Node, written by Claude Code sessions. SQLite through a synchronous driver (better-sqlite3), the TypeScript Agent SDK for worker and hook control, launchd plists for the supervisor. Dependency footprint kept deliberately small, because the factory's own dependencies are themselves a target of the factory; the repo carries a `CLAUDE.md` with these rules so later sessions keep to them.

### Open
1. **Auth and billing** (deferred by the owner). Design for an API key behind a config setting. The Phase 1 billing test and the terms facts in E18 inform the choice. Decide before Phase 1 exits.
2. **Non-Claude juror provider.** Which vendor and model, through which path (API or CLI), at what cost per verification, and who holds that key in `keyproxy`. Phase 6 cannot exit without it.
3. **Forgejo viability.** Ephemeral runners and bot-token CI triggering are unverified (E19). Checked at the start of Phase 2; the fallback is already named.
4. **Factory-level definition of done.** The proposal above needs the owner's thresholds.

## Verification status

- Claude stack and tooling facts: verified from official docs and source (E18, E19), except credits coverage of `claude -p`, Forgejo runner behaviour, ntfy CORS, and the Renovate lockfile stall.
- Paper-sourced numbers: read from the PDFs and the highest-impact ones re-verified (E1 to E17); the disputes and the dropped claims are in E22; still-unreachable sources are in E23.
- Upstream CLI contracts for the worked example: source level only, no binary run (E20).
- Highest-value remaining work is measurement on our own repos, not more reading: adapt success rate, flake rates, revert rate against a human control, judge agreement with human labels, Claude-vs-non-Claude judge bias, stall thresholds, sandbox concurrency, cost per goal type. Each is an exit test above.
