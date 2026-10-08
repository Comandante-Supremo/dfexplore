# Dark factory: design

Status: v4, 2026-10-08. This is the set of design decisions. The numbers behind them live in `EVIDENCE.md` (cited as E1, E2, ...). The phases, exit tests and open items live in `BUILD_PLAN.md`. Worked examples (not core design) live in `examples/`. `[U]` marks a guess or design choice with no evidence behind it.

## 1. The goal

Every unit of work is a **goal**: objective, machine-checkable success criteria, constraints, allowed outcomes, external state, and an escalation rule. Goals can spawn sub-goals (a DAG). Four goal types cover the use cases:

| Type | Example | Notes |
|---|---|---|
| `update` | Upstream changed (library, CLI, API, model) | The worked examples are a FastAPI pin and a multi-harness messaging tool |
| `solve` | Long-running hard problem | Needs a progress signal and stall detection |
| `compare` | Try N options, pick the best | Gates, then metrics, then judged criteria |
| `build` | Greenfield build from a conversational vision | Factory proposes the definition of done; user owns acceptance; rejection spawns a new round |

**Allowed outcomes:** `adapt`, `pin/hold`, `constrain`, `work around`, `escalate`. Pinning is a success, recorded in the **constraints registry** (reason, evidence, upstream link, revisit trigger, max age) and re-tested on a loop so pins cannot rot.

**Adapt is expected to fail often** (best measured full-repair rate 27%, E1), so it has a bounded budget and falls through automatically to pin/constrain. Adapt success is build-level only. A repair must also pass a four-state check: the repair alone, without the version bump, must not make the new tests pass (adapted from DepBench's task oracle; using it on agent output is our design `[U]`).

## 2. Autonomy

Proceed when one option is clearly dominant. Escalate for architectural or design changes, genuine forks, and real ambiguity. A worker may always choose to escalate or give up (an abort option cut cheating sharply for some models, E9). Autonomy is set per repo in `.factory/policy.yaml`. Escalation is asynchronous: the goal parks, other goals continue.

**Auto-merge gate (owner decision):** green CI **and** canary pass **and** verifier pass with clean integrity **and** diff within size limits **and** no sensitive-path touches. Anything else escalates. Rationale: green CI alone is weak (tests catch 35 to 47% of injected dependency faults, E2; no study covers unattended auto-merge, E6). This deliberately exceeds Copilot's human-merge baseline (E21).

## 3. Components

**Supervisor.** Plain code, not an LLM. One daemon on SQLite (WAL) with launchd on the Mac host. Owns the goal state machine, leases and heartbeats, the cost ledger, budgets, stall detectors, the merge credential and the kill switch. Agents propose; the supervisor disposes. The Agent SDK and Claude Code give per-run caps and local session resume but no goal state machine, cross-process lease, cross-run ledger, idempotent side effects or escalation (E18), so the supervisor is thin but necessary. Temporal, Restate and DBOS are deferred (the objection is an extra server and workflow-code discipline, not a replay-model mismatch).

**Key proxy (`keyproxy`).** The only component that holds provider API keys and forge tokens. Sandboxes reach providers only through it. It injects credentials, enforces a per-attempt egress allowlist with request-level checks (an allow-listed host can still carry an attacker's key, E8), enforces the per-attempt and per-goal USD caps from the ledger, and logs every request. This is the most important security component and is a named Phase 1 deliverable.

**Sandbox runner.** A container per attempt inside the Docker Linux VM on the Mac, with a git worktree per attempt. Non-root; no credentials inside; self-updaters disabled; explicit permission mode; per-run config and home directories; tools invoked by absolute path with explicit arguments, never bare; an unvetted version is never run just to read its version string (see `examples/harness-compat.md` for why). Equal, recorded resources for compare goals (resource limits alone move scores by 6 points, E12). Concurrency is bounded by the VM's CPU and memory; the estimate is 4 to 6 sandboxes on a 32 GB machine `[U]`, to be measured.

**Worker execution.** `claude -p --bare` or the Agent SDK, with caps re-issued on every invocation; the factory's ledger is the authoritative budget because `--max-budget-usd` ignores spend restored by `--resume` (E18). Fresh session per attempt by default; fresh vs resumed is a per-model config (no controlled study, E14). Pin the model ID and the Claude Code version; capture a stream-json fixture. "Keep going until CI is green" is a supervisor loop; Stop hooks are a backstop capped at 8 continuations (E18). Safety rests on deny rules and SDK callback hooks, which fail closed; PreToolUse command hooks fail open (E18); a startup self-test proves each rule fires. Stop on SIGINT then SIGTERM. Classify the three limit responses (spend-cap 429, workspace 429 with retry-after, self-limit 400). Export OpenTelemetry.

**Roles and models.** Planner, worker (fresh context, one task, cannot edit criteria, may escalate), verifier (separate context, sole writer of `satisfied`). Default mapping: Opus 5.5 plans and verifies, Sonnet 5.5 works, Haiku 5.5 for cheap subagents (5x cost above 100K prompt tokens), Fable 5.1 only for the hardest greenfield planning. A worker runs as one agent; parallel subagents inside a worker are not the default for coding work.

**Verifier.** Independence means information isolation (E12, E13): the builder cannot read or edit the scorer; no git-history leak; and, most important, **no network route** to verifier artifacts, including third-party mirrors (network paths dominate leaks, E12). On one Mac this needs real separation, not a label: worker containers live on a Docker network whose only egress is `keyproxy`; the verifier runs on a separate Docker network with its own forge token and runner label, and its artifacts are never published where a worker could fetch them. Implementation order: deterministic checks first (contract, replay, held-out tests, surface diff, integrity), LLM judges last. One `VerifyRequest/VerifyResult` interface for all goal types, recording `judge_model_id`. Strengthened acceptance tests are required because oracles are wrong often enough to matter (E11).

**Anti-gaming.** Protected paths for criteria and tests; test-diff policy; mutation testing on changed lines only, gating on no regression rather than a threshold (E11); builder-added quarantines are integrity violations recorded as events; sandbox egress denies fetching upstream fixes during `solve` and `update`. Outright tampering is rare under benign conditions but special-casing is common (E9), so correctness rests on property and held-out tests, and gaming on environment hardening (combined hardening cut exploits from 6.5% to 0.8%, E9).

**LLM judges.** Pairwise and order-swapped (pairwise beats pointwise for code, E10). Claude judging Claude shows measured self-preference of about 1.7 to 1.8 on code (E10), so a judge-only criterion can block a merge only when a panel including a non-Claude juror agrees; until that juror exists, judged criteria are advisory. Each judge is calibrated separately against 50 or more human-labelled examples with a kappa gate (0.61 `[U]`), its model ID pinned, recalibrated on change. Trial counts and sampling schedules are guesses (the "86.6/90/95" figures measure self-consistency, not accuracy, E10). The judge sees a comment-stripped diff as a cheap injection mitigation `[U]`.

**Untrusted input.** Release notes, changelogs, issues and diffs are attacker-controllable data. Readers of untrusted text are split from privileged agents; the worker has no secrets or egress beyond `keyproxy`; writes are confined to safe outputs. Prompt-level defenses are not a control (78 to 93% bypass under adaptive attack, E8); scope is enforced by the sandbox, not the prompt (E7).

**Merge controls.** One branch per attempt. The worker credential cannot approve or merge; only the supervisor holds the merge credential; rationale and confidence are recorded per decision, but self-reported confidence never gates alone (E10). Escalate on size: agent PRs are an order of magnitude larger than human ones and more commits predict more follow-up fixes (E6). Track revert rate per agent and task class against a human control; budget for the 6 to 15% revert-proxy range seen with human review (E6). Fix-forward by the same agent is capped. Post-merge canary failure triggers a supervisor-generated revert PR and pin re-application.

**Kill switch.** A flag checked before every lease, plus forge-token revocation through `keyproxy`, outside any agent's reach. Per-goal and per-day USD caps and a consecutive-failure stop. Good practice; no source evidences it.

**Self-change canary.** Every change to the factory's own prompts, model IDs, effort settings or pinned Claude Code version runs against a recorded replay set of past goals, per model, before deploy, with a soak period (Anthropic's postmortem, E18). This matters because the factory may be used to improve the tools it depends on.

**Detection.** Self-hosted Renovate (Node 24) or the factory's own watchers do detection and PR mechanics only; decisions live in the factory. Generated Renovate config must null `minimumReleaseAge` for update types it does not cover (unproven, E4). Run an independent advisory scan because Dependabot `ignore` can suppress security fixes (E4).

**Cooldown (policy, not evidence).** Default 3 days, with a 7-day profile; no effectiveness study exists (E4). Security updates bypass cooldown but not the canary. Enforce in the factory's resolver and natively where available; Cargo needs Rust 1.100, so gate on crates.io `pubtime` until then; Go uses `index.golang.org` timestamps.

**Update-goal testing rules.**
- Reruns: 3 for deterministic failure classes. For behavioural failures, rerun only the failing tests, paired and interleaved on new and current versions, and blame the update only if every new run fails and every old run passes; choose k from the false-positive budget (k=3 gives 1.6% for a coin-flip test, k=5 gives 0.1%; "any failure" and majority rules have a 25% worst case, E5). Mixed results escalate. Keep per-test flake history.
- Bisect: confirm the boundary with paired reruns at `last_good`, `first_bad` and `first_bad+1`, then search.
- Constraint syntax: a cap while the bad range's upper end is unknown, exclusions once a fix is confirmed, compiled per ecosystem `[U]`.
- The canary diffs transitive dependencies and the consumed API surface, not only the test result (tests miss half of injected faults, E2).

**Contract testing for CLI and API upstreams.** Each upstream declares an observable contract: invocation flags, output or event schema, exit codes, config and auth paths, self-update behaviour. Fixtures with fake shims for fast checks; a live smoke test within 24 hours of each new stable release; a liveness check (exit 0 without expected output fails); a new enum value is `ambiguous` or `breaking`. Worked example: `examples/harness-compat.md`.

**Escalation.** A `question` table (context, options, recommendation, default-after-timeout) with ntfy, a web answer page and PR comments over Tailscale. Answers must parse strictly; an unparseable answer is rejected, never mapped to a default; re-asks are capped. User-owned chat tools are later transport adapters. Upstream issues and comments are drafted by the factory and posted only after the owner approves. Asking helps: clarifying questions raised resolve rates from 54.8% to 69.4% on hidden-information tasks (E17).

**Stall detection.** Start from the OpenHands SDK defaults (4 identical action+observation repeats; stuck on the 4th consecutive same-action error; 3 A/B cycles; monologue 3), which are tool defaults rather than evidence (E14). Add an action fingerprint and a duplicate-call metric (pass rates fall sharply with run length, E14). Detectors read live stream-json. Flaky tests can look like stalls; spend-cap stops are a separate reason.

**Planning.** Decompose into leaves of about 30 to 60 minutes of human-equivalent work `[U]` (the 80% horizon is 4 to 6x shorter than the 50% one, E14), log actual durations, calibrate. LLM-only self-grading never flips a criterion (E13). Expect the MAST v3 failure mix: repetition, not noticing termination, and verification gaps (E14).

## 4. Host and infrastructure (owner decisions)

- **Host:** Apple Silicon Mac, 32 GB or more, Docker Linux VMs. Consequences: launchd for the supervisor; containers in the VM rather than rootless Podman; no microVM tier; arm64 images for every tool.
- **Forge:** local Forgejo with Forgejo Actions. Ephemeral runner support and whether a bot token's PRs trigger CI are unverified (E19) and must be checked before the first PR phase. **Fallback:** GitHub with JIT ephemeral runners (APIs verified, E19).
- **Reachability:** Tailscale (or another VPN) for the answer page and ntfy actions; nothing public.
- **Known-bad versions in downstream tools:** bundled per release from the registry; no runtime fetch; warn by default, block for safety-class entries; version read from metadata, never by running the upstream.

## 5. Cost envelope (illustrative, `[U]`, to be replaced by Phase 2 measurements)

Using E18 list prices and guessed token volumes: a Sonnet 5.5 worker attempt of about 300K input and 30K output tokens costs about $0.90; an Opus 5.5 verification pass of about 100K input and 10K output about $0.60. An `update` goal with a canary, up to three adapt attempts and verification is then roughly $3 to $6; a `solve` goal of 20 iterations roughly $30 to $60; a `compare` goal at N=3 about three worker attempts plus judging. Anthropic reports a full harness at about 20x a solo run (E13), so treat these as lower bounds. The per-goal and per-day caps in `policy.yaml` are the control; the ledger reports actuals per goal type from Phase 2 onward, and the envelope is revised then.
