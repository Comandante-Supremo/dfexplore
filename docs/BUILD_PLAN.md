# Dark factory: build plan (v2)

Status: v2, 2026-10-07. v1 was built from the seven wave-one docs in `research/`. v2 folds in the eight wave-two docs (`research/wave2/W2-*.md`) after an independent audit of each (`research/wave2/audit/A-*.md`, run on a different model). Only edits the audits accepted or modified are applied; rejected edits are listed in section 6. Decisions marked **OPEN** need the owner before the phase that depends on them.

Evidence policy: figures tagged `[S]` (seen only in search summaries) or `[U]` (guess/design) are not defaults to hard-code. Where this plan states a number that is a guess, it says so.

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
`adapt`, `pin/hold`, `constrain`, `work around`, `escalate`. Pinning is a successful outcome, recorded in the **constraints registry** with reason, evidence, upstream link and a revisit trigger, and re-tested on a loop so pins cannot silently rot (doc 05).

Adapt attempts are expected to fail often (best measured LLM repair rates are 19 to 27% full fix on Java `[S]`). So adapt has a bounded budget and falls through automatically to pin/constrain. Adapt success is build-level only (no partial fixes), and a repair must pass a four-state check: the repair alone, without the version bump, must not make the new tests pass (DepBench `[S]`).

### Autonomy rule
Proceed when one option is clearly dominant. Auto-merge on green CI plus canary. Escalate only for architectural/design changes, genuine forks, and real ambiguity. Autonomy is set per repo in `.factory/policy.yaml`. Escalation is asynchronous: park the question, continue unblocked goals. Note: supervisor auto-merge deliberately goes beyond Copilot's human-merge baseline (A-08); it is this plan's own choice, not prior art. A worker may always choose "escalate/give up" as a valid outcome (reduced hacking in ImpossibleBench `[S]`).

### Architecture

**Supervisor.** Plain code (not an LLM), one daemon on SQLite (WAL) plus systemd timers. Owns the state machine, leases, budgets, ledger and stall detectors. Agents propose; the supervisor disposes. Temporal/Redis deferred. The Agent SDK and Claude Code provide per-run caps, local session persistence/resume and an optional external session store, but no goal state machine, cross-process lease, cross-run ledger, idempotent side effects or escalation (W2-02, verified), so the supervisor stays but is thin. SessionStore is used only for cross-host resume. Do not use `claude --bg` as a runner.

**Execution.** `claude -p --bare` or the Agent SDK, API-key auth, in rootless Podman plus a git worktree per attempt.
- Re-issue turn and USD caps on every invocation. `--max-budget-usd` ignores spend restored by `--resume`, so the factory's own ledger is the authoritative budget. Pin full model IDs and the Claude Code version; capture a stream-json fixture.
- Resume rule: fresh session per attempt by default; fresh vs resumed context is a per-model config (Anthropic's guidance is model-dependent).
- Stop on SIGINT then SIGTERM; handle `RESUME_INTERRUPTED_TURN`.
- "Keep going until CI is green" is a supervisor loop. Stop hooks (and `/goal`) are capped at 8 consecutive continuations and are only a backstop.
- Safety rests on deny rules and exit-2 hooks, not on PreToolUse command hooks (they fail open on timeout and non-2 exits). Run a startup self-test that proves each deny rule and hook path fires. If the SDK is used, prefer SDK callback hooks for policy gates (they fail closed).
- Sandbox: non-root, always `--bare` with explicit config, `dontAsk` mode with explicit allow and deny rules as the default.
- Classify all three limit responses: spend-cap 429, workspace 429 with retry-after, self-set-limit 400. Record the actual rate-limit tier; concurrency is capped from TPM. Treat any specific limit numbers in the docs as dated examples.
- Prompt cache: use the 1-hour TTL only for attempts that wait more than 5 minutes; measure hit rate before making it the default.
- Export OpenTelemetry (Prometheus port 9464 verified).
- `claude-code-action` is not on the core path (self-hosted runner support is undocumented; source handles non-ephemeral runners). Use ephemeral self-hosted runners.
- Haiku 5.5 costs more above 100K tokens; a safety fallback can silently swap a model, so flag verifier model-tier changes.

**Roles and models.** Planner, worker (fresh context, one task, cannot edit criteria, may escalate), verifier (separate context, sole writer of `satisfied`). Opus 5.5 plans and verifies, Sonnet 5.5 works, Haiku 5.5 for cheap subagents; Fable 5.1 only for the hardest greenfield planning. Re-check model IDs and prices on the public pricing page (doc 02 used a cached reference). Multi-agent is not the default for coding work.

**Verifier.** Independence means information isolation (A-04):
1. The builder cannot read or edit the scorer.
2. No git-history leak (`git log --all`, shared remote).
3. No network route to verifier artifacts, including third-party mirrors (BrowseComp's key leaked through a mirror).
Implementation: separate repo, runner label, token and required-status check. Deterministic checks first (contract, replay, held-out tests, surface diff), LLM judges last. One `VerifyRequest/VerifyResult` interface for all four goal types, with `judge_model_id` recorded in criteria evidence.
- Judges: pairwise, order-swapped; the judge sees a comment-stripped diff; blocking judge-only criteria need a non-Claude juror in a panel, otherwise they are advisory only, until a local experiment measures sibling-model bias. The outcomes grader is still Claude (relevant to OPEN #6).
- Calibration: few-shot plus hard thresholds plus a kappa gate (>= 0.61 on 50+ labelled examples, guess) before any judge gates. Pin the judge model id and recalibrate on change.
- Differential test against the prior version is a non-blocking divergence report `[U]`. Strengthened acceptance tests are required (the oracle can be wrong; 6 to 28% of SWE-bench-passing patches fail stronger tests `[S]`, with scope caveats in A-04).

**Anti-gaming.** Protected paths for criteria/tests, test-diff policy, mutation testing on changed lines only (gate on no regression, not an absolute threshold), quarantine-by-builder is an integrity violation recorded as `event(type='integrity_violation', payload=evidence)`. Sandbox egress denies fetching upstream fixes or solutions during `solve`/`update` attempts. Solution special-casing is common even when tampering is rare (EvilGenie `[S]`), so property/held-out tests matter for correctness.

**Sandbox security.**
- No long-lived forge/cloud/registry tokens in the sandbox. The model key is injected by a proxy or is short-lived. Default-deny egress except provider APIs, with request-level checks (an allow-listed host can carry an attacker's key).
- Harness env sets `DISABLE_UPDATES=1`, `KIMI_CODE_NO_AUTO_UPDATE=1`, an explicit permission mode, and per-run config/home dirs. Harnesses are invoked by absolute path and never with empty argv. Never run bare `kimi`.
- Untrusted upstream text (release notes, changelogs, issues, diffs) is data: the worker has no secrets or egress, and writes are confined to safe outputs.
- Equal, recorded sandbox resources for compare goals (resource limits alone swung scores by 6 points).

**Merge controls and kill switch.** One branch per attempt. The worker credential cannot approve or merge; the merge credential is held only by the supervisor. Rationale and confidence are recorded per decision. A fleet kill switch (flag checked before every lease, plus forge-token revocation) is good practice, not an evidenced pattern. Per-goal and per-day USD caps plus a consecutive-failure stop; no imported "3/20" numbers. Post-merge: a revert path (supervisor-generated revert PR, pin re-application), and a post-merge canary failure triggers revert. Verify that PRs opened with the default `GITHUB_TOKEN` do not trigger CI (general knowledge, unverified): use a PAT or GitHub App.

**Detection.** Self-hosted Renovate (needs Node 24) or the factory's own watchers for detection and PR mechanics only. The decision procedure lives in the factory (docs 01, 05). Generated Renovate config must null `minimumReleaseAge` for `lockFileMaintenance` and other unsupported update types (test this; unproven). Dependabot `ignore` can suppress security fixes, so run an independent advisory scan.

**Cooldown (policy, not evidence: no effectiveness study exists).** Default 3 days, with a 7-day profile (64.3% of adopters choose 7 days `[S]`). Verified native options: uv, pip, npm, pnpm (use strict), Dependabot (3-day default, version updates only). Cargo native cooldown needs Rust >= 1.100 (beta now; stable expected about 2026-11-12), so gate Cargo on crates.io `pubtime` in the factory until then. Go: prefer `index.golang.org` `Timestamp`, with the factory's first-seen store as fallback. The PyPI 14-day rule is not a cooldown. Security updates bypass cooldown but not the canary.

**Update-goal testing rules.**
- Reruns: 3 for deterministic classes (compile, import, missing symbol). For behavioural failures, use paired, interleaved reruns of the failing tests only, k = 5 with sequential stopping, on the new and current versions; record per-test flake history; mixed results escalate. (The earlier 3/10/30 tiers had no basis.)
- Bisect: confirm the boundary with paired reruns at `last_good`, `first_bad` and `first_bad+1`, then search.
- Constraint syntax: use a cap while the bad range's upper end is unknown; switch to exclusions once a fixed version is confirmed; compile per ecosystem (Cargo has no `!=`, npm uses `<X || >X`, Go uses exclude/retract). Tag `[U]`.
- Canary diffs transitive dependencies as well as direct ones.
- Evidence gap: no study covers Python or CLI contract breaks (your main targets), and BUMP is Java-only. Build the replay set from real breaking updates in the target repos' own history.

**Harness conformance (example repo for goal type `update`).** From W2-01 (source/doc-verified, runtime unverified; audited):
- Treat each harness as an upstream with a per-harness event grammar; terminal = terminal event or process exit. Liveness check: exit 0 without protocol output fails.
- Kimi: two adapters, `kimi-code` (npm `@moonshot-ai/kimi-code`) and `kimi-cli-legacy` (checks capped at 1.51.0). `kimi-cli==1.52.0` is banned in the registry: every entry point exits 0 with a notice and a bare run executes a shell string fetched from a CDN, so tests would record false successes.
- Contract covers invocation flags, not only event fields (Codex removed `--full-auto` 93 days after deprecation; one data point, not a default).
- Live smoke test within 24 hours of each new stable release. Use an API key, not a "throwaway account". Track Claude `stable` and `latest` separately.
- Candidate additional harnesses (all speak ACP): Gemini CLI, Cursor CLI, Copilot CLI. ACP is mid-migration (v1.24.1; v2 draft).
- Planned new-enum-value in an event is classified `ambiguous` or `breaking`.

**Escalation.** A `question` table (context, options, recommendation, default-after-timeout) with ntfy, a web answer page, and PR comments as transports. Answers must parse strictly; unparseable answers are rejected, not mapped to a default, and re-asks are capped (do not copy Attractor's first-choice fallback or no-default retry loop). ntfy `http` actions use a bearer header, max 3 actions, and always a `view` link; test the web-client CORS path. Webhooks: Cloudflare Access Bypass on the webhook path plus HMAC check, 10 s limit, dedupe on delivery GUID, 3-day redelivery reconciler, and receiver-side logging because Bypass is unlogged. The multi-harness messaging tool is one more adapter (conditional on it exposing MCP or ACP).

**Stall detection (uncalibrated defaults, calibrate in Phase 5).** Based on OpenHands semantics: 4 identical action+observation repeats; kill on the 4th consecutive same-action error, nudge at 3; 3 A/B cycles; monologue 3. Add `attempt.action_fingerprint`. Stall detectors must parse live stream-json (Phase 1). Flaky tests can cause false stalls; spend-cap stops are a separate stop reason.

**Planning guidance.** Decompose goals estimated over roughly 1 to 2 hours of human-equivalent work; log actual durations to calibrate. LLM-only self-grading never flips a criterion; verification is grounded in commands. Fresh-context-per-iteration is not evidence-backed in general (A-05); Ralph-style loops have no controlled evaluation.

## 2. Phases

Each phase ends with something runnable and a reboot/crash test where state is involved.

**Phase 0: Skeleton (no agent).** SQLite schema (doc 04, plus `attempt.action_fingerprint`), goal state machine, event log, one release poller for one dependency, generated `GOAL.md`/`PROGRESS.md` views. Exit: a goal moves through states from a poll event; `kill -9` mid-transition recovers with no duplicate effects.

**Phase 1: Sandbox runner.** Rootless Podman (check cgroup delegation, `/dev/kvm`) plus worktree launcher for `claude -p` with stream-json, transcript and artifact capture, caps re-issued per run, authoritative ledger, lease and heartbeat, pinned Claude Code version plus a captured stream-json fixture, startup self-test of deny rules, kill-switch flag, credential isolation, harness env hardening. Exit: an attempt runs, is killed (SIGINT then SIGTERM), and resumes from its last commit; a planted token in the worktree is unusable for outbound calls; a Phase 1 test records how a credits-only org bills `claude -p` versus SDK calls before any billing assumption is made.

**Phase 2: Update goals end to end (use case 1).** Constraints registry and `policy.yaml` schemas with validation; canary in a sandbox with a minimal consumed-symbol/import check plus lockfile and transitive diff (the full surface-diff library stays in Phase 3); decision procedure; bounded adapt with automatic fall-through; bisect worker with boundary confirmation and cost caps; PR creation (GitHub App or PAT), green/red read, auto-merge per policy; post-merge revert path; re-test loop for pins. First targets: one ordinary library repo (FastAPI-style pin) and one CLI-contract repo. Metrics: adapt success rate, reverted-auto-merge rate (with a rollback action), flake rate per test, pin age, pin count. Exit: a deliberate bad upstream release produces a pin with evidence and a later fixed release removes it automatically; the factory's adapt rate is measured on a replay set of real breaking updates from the target repos' history.

**Phase 3: Verifier and conformance.** Separate verifier repo and runner. Surface extraction and diff library (CLI `--help`, event schemas, Python public API, lockfile). Conformance layout: per-harness consumer contracts, recorded fixtures with fake-CLI shims, live smoke within 24 hours of each new stable version. Integrity checks. Judge calibration (kappa gate). Exit: planted test-weakening is caught; planted breaking CLI change is classified `breaking`; planted new enum value is `ambiguous`/`breaking`; exit-0-without-output fails liveness; planted verifier read, `git log --all` and network fetch of verifier artifacts all fail; planted protected-test edit is caught.

**Phase 4: Escalation.** Question table and service, ntfy plus web answer page, timeout/default executor, strict answer parsing with capped re-asks, PR-review-as-answer ingestion. Read-only dashboard (goals, active pins, median pin age, open questions). Exit: a forced fork pauses one goal and others continue; a reply resumes it; an unparseable reply is rejected.

**Phase 5: Solve goals (use case 2).** Planner/worker/verifier loop with fresh-context iterations (per-model config), progress metric, stall/loop/drift detectors with the corrected defaults, response ladder (retry, replan, allowed outcome, escalate). Exit: a multi-hour goal survives a reboot, stalls are detected without self-reports, and LLM-only grading never flips a criterion. Phase 5b (deferred): best-of-N reusing Phase 6 machinery.

**Phase 6: Compare goals (use case 3).** N sandboxes (default 3, max 4; guess) with equal recorded resources, scoring schema (gates, Pareto metrics, pairwise judges), dominance pre-check, comparison report as a choose-question when no option dominates. Hybrid verification (tests plus verifier) is the default for ranking gate survivors, behind a measurement (the evidence uses trained verifiers, not prompted judges). Copeland aggregation is a candidate (cheap at N <= 4). Exit: selector accuracy on a planted set that includes near-tie and polish-bias pairs; gate false-positive rate measured; a known-better option wins reliably across reruns. Evolutionary mode (numeric fitness only, budget-capped) is optional later.

**Phase 7: Build goals (use case 4).** Ambiguity-detection step before elicitation (design; models rarely ask about style/UX requirements), conversation-to-spec template, stop-asking rule (max about 5 questions, guess), factory-proposed definition of done with a separate agent writing acceptance tests, acceptance tests validated (mutants, second generation, human sign-off on expected values), `goals.yaml` sub-goal DAG, walking-skeleton first milestone, milestone verification, traceability drift metric judged by the verifier (not the builder), acceptance page, rejection loop (max 3 rounds, guess) that classifies feedback and feeds a "DoD lessons" file (an untested hypothesis with weak single-deployment evidence and an over-personalisation risk). Record the first-pass acceptance rate. Exit: a small greenfield project is built, rejected once with feedback, and rebuilt in a follow-up round.

Optional later: microVM isolation tier, Managed Agents pilot behind an executor interface (beta, $0.08 per session-hour, excluded from ZDR/HIPAA), evolutionary mode, durable engine if hand-rolled resume gets painful (revisit thresholds `[U]`).

## 3. Open decisions for the owner

1. **OPEN: Forge and CI.** GitHub with ephemeral JIT runners (`generate-jitconfig` plus a small launcher; admin access required), local Forgejo (v15 ephemeral runners are secondary-sourced only), or both. ARC needs Kubernetes (k3s/kind), probably overkill. Decides the Phase 2 CI path.
2. **OPEN: Auth.** Recommendation: API-key billing for an always-on factory. The Legal page says users may run the unmodified Claude Code binary with their own subscription; no page addresses an always-on factory either way. Max/Team credits ($100 or $200 a month, expiring) may not cover `claude -p` or Claude-Code-backed SDK traffic ("Credit balance too low" in credits-only orgs); require purchased credits or the Phase 1 test before relying on them.
3. **OPEN: Host capacity.** KVM (`/dev/kvm`) for a microVM tier? RAM for N concurrent sandboxes? Rootless cgroup delegation drop-in? Rust toolchain >= 1.100 if Cargo native cooldown is relied on?
4. **OPEN: Phone/remote reachability.** VPN (e.g. Tailscale) or public tunnel, for the answer page and ntfy actions.
5. **OPEN: Messaging tool API.** Inbound replies, buttons, threading, auth, and whether it exposes MCP or ACP.
6. **OPEN: Judge independence.** Builder and outcomes grader are Claude. Add a non-Anthropic juror for blocking judge-only criteria (policy in section 1), or accept advisory-only judging? Same-family different-tier judges are untested.
7. **OPEN: Upstream comments.** May the factory open or comment on upstream issues itself? Default: draft only, human approves.
8. **OPEN: Repo-bundled constraints** for upstream-controlled installs: ship the bad-version list with the tool or fetch it at runtime.
9. **OPEN: Which harnesses are in scope** beyond Claude Code, Codex, OpenCode and Kimi (Gemini CLI, Cursor CLI, Copilot CLI).

## 4. Verification status and remaining research

Status of the v1 second-pass list after wave two and audit:
1. Harness CLI schemas: **done at source/doc level** (W2-01, audited). Runtime behaviour is unverified because no binary was run.
2. Agent SDK / `claude -p`: **resolved**, except whether credits cover `claude -p` and the Managed Agents session-hour fee, whether a personal always-on subscription loop is permitted, and `claude-code-action` on self-hosted runners.
3. Renovate/Dependabot/registries: **done** (W2-07, audited), except the Dependabot rollout date and the Renovate lockFileMaintenance stall (unproven).
4. Forgejo / ntfy / Cloudflare: **partial**. ntfy and Cloudflare verified; Forgejo ephemeral runners still secondary-only; ntfy CORS and "http action runs on the phone" untested.
5. StrongDM / Stripe / Huntley: **still unverified** (hosts blocked). The public StrongDM repos are specs only, last commits 2026-02-23 to 2026-04-06 (cxdb 2026-08-28); treat them as reference, not live dependencies.
6. SDK vs hand-rolled supervisor: **resolved** (thin supervisor stays).
7. Spec formats and build failure rates: **partial** (abstract-level only).
8. DBOS SQLite production-readiness and stall thresholds: **open**; needs Phase 5 data.

**Needs network access to arXiv/ACM/IEEE/USENIX/OpenReview/ACL (blocked in this environment):** every paper-sourced number is `[S]`/`[S2]`. Highest-priority re-reads, in order: judge reliability (arXiv 2606.13685), reward-hacking rates (EvilGenie 2511.21654, Reward Hacking Benchmark 2605.02964), test-validity studies (UTBoost, PatchDiff, SWE-ABS, STING), dependency breakage studies (Hejderup, Jayasuriya, Venturini, Go, Gruber flaky reruns), best-of-N/verifier papers (Stroebl ICLR 2026, R2E-Gym, SWE-HERO Table 3), MAST taxonomy, METR time horizons, and the elicitation papers (Ambig-SWE, ClarifyGPT, LLMREI, ReqElicitGym).

## 5. Tensions between docs (resolved)

- Doc 01 says adopt Renovate; doc 05 says do not use it as the brain. Resolution: detection and PR mechanics only; decisions in the factory.
- Doc 05 says webhooks are unnecessary for registries; doc 07 says poll first, webhooks later. Compatible: poll everywhere, add webhooks only for latency-sensitive triggers.
- Verifier independence: A-04 shows the evidence supports information isolation, not a separate repo as such. Adopt the separate repo/runner/token as the implementation of information isolation (section 1).
- Doc 02's Managed Agents pilot conflicts with the "own state" principle only if sessions become the source of truth. Keep the executor interface thin and the SQLite store authoritative.
- Doc 07 resumes by session ID; W2-02 prefers fresh sessions. Resolved: fresh by default, per-model config.
- Doc 04 attributes premature-"done" fixes to an independent verifier; the Anthropic Nov 2025 post uses self-verification. A later (Mar 2026) Anthropic post calls separating doer from judge "a strong lever", so the independent verifier stays with better backing.

## 6. Wave-two edits rejected or narrowed by audit

- W2-06's "correction" to the SWE-HERO figures was **rejected**: wave one's 64.6 (Best@32) vs 79.8 (Pass@32) stand. W2-06's replacement numbers are dropped.
- W2-05's correction that premature "done" is rarer than non-termination was **rejected**: MAST's own figures give premature termination 8.64% > unaware of termination 6.54%. W2-05's MAST percentages (15.7/12.4/23.5) were wrong.
- Copilot as support for a merge `automation_level` enum was **rejected**: Copilot requires a human to merge and its automation level covers issue triage only.
- The 3/10/30 rerun tiers and the Gruber-170 justification were **rejected**.
- The "6 to 28% wrong patches" range, the TOGA 47.5% figure as "agent-written oracle", and nf-core 98.46%/72.13% were **dropped** (scope, mixed units, existence unconfirmed).
- The "repos alive through 2026-10" claim for StrongDM repos was **rejected** (commit dates above).
- "Mark research DONE", the "throwaway account" canary, and the 93-day deprecation default (one data point) were **modified** as noted above.
