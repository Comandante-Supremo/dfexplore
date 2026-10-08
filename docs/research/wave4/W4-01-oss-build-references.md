# W4-01: Open-source build references per component

Date: 2026-10-08. Scope: for each DESIGN.md section 3 component, which existing OSS to read, borrow, adopt or avoid. The whole-factory survey is not repeated (see 01-prior-art-landscape.md, W2-08).

Method: web search for candidates, then `git clone --depth 1 --filter=blob:none` into the scratchpad and read the cited files. Nothing cloned was executed or installed. "Last commit" is the HEAD commit date of the shallow clone on 2026-10-08. Hosts `code.forgejo.org`, `codeberg.org` and `gitea.com` returned 403 from the agent proxy, so the Forgejo runner and Gitea act_runner source was NOT read (section 10 relies on Forgejo docs found by search; flagged). The `gh` search API is also blocked in this session. Where a claim comes from a search snippet rather than source, it is marked "(web, unread)".

## Summary table

| Project | Lang | License | Last commit | Component(s) | Verdict |
|---|---|---|---|---|---|
| anthropic-experimental/sandbox-runtime | TS | Apache-2.0 | 2026-10-07 | 2, 3 | BORROW code (keyproxy core); possibly ADOPT pieces |
| mattpocock/sandcastle | TS | MIT | 2026-10-08 | 3, 4 | READ for patterns (do not depend: pulls in Effect) |
| anthropics/claude-code-action | TS | MIT | 2026-10-07 | 3, 4 | READ (SDK invocation, retry) |
| anthropics/claude-code (devcontainer, ralph-wiggum plugin, gateway examples) | sh/misc | Proprietary (all rights reserved) | 2026-10-08 | 2, 3, 4 | READ only; do not copy files |
| anthropics/claude-agent-sdk-typescript | TS | Proprietary (Commercial Terms) | 2026-10-08 | 3, 11 | ADOPT the npm package (already planned); READ `examples/session-stores/shared/conformance.ts` |
| openworkflowdev/openworkflow | TS | Apache-2.0 | 2026-10-07 | 1 | BORROW schema and lease SQL; READ before choosing |
| karakeep-app/liteque | TS | MIT (package.json); repo has no LICENSE file | 2026-08-31 | 1 | READ (expiry-based dequeue); Drizzle dependency, AVOID adopting |
| restatedev/restate | Rust | BSL 1.1 | 2026-10-08 | 1 | AVOID (BSL, extra server; already deferred) |
| dbos-inc/dbos-transact-ts | TS | MIT | 2026-10-06 | 1 | READ only (Postgres-oriented; deferred by design) |
| BerriAI/litellm | Python | MIT (except `enterprise/`) | 2026-10-08 | 2 | BORROW price table and budget semantics; AVOID as a runtime dependency |
| maximhq/bifrost | Go | Apache-2.0 | 2026-10-08 | 2 | READ for virtual-key budget model; AVOID (Go service, large surface) |
| Portkey-AI/gateway | TS | MIT | 2026-05-25 | 2 | AVOID (stale >4 months, broad provider-translation surface) |
| musistudio/claude-code-router | TS | MIT | 2026-10-08 | 2 | AVOID (routing, not security; holds keys in config) |
| OpenHands/software-agent-sdk | Python | MIT | 2026-10-08 | 9 | READ then port (`stuck_detector.py`) |
| google-gemini/gemini-cli | TS | Apache-2.0 | 2026-10-07 | 9 | BORROW (`loopDetectionService.ts`) |
| sst/opencode | TS | MIT | 2026-10-08 | 9 | READ (`DOOM_LOOP_THRESHOLD` in `session/processor.ts`) |
| block/goose | Rust | Apache-2.0 | 2026-10-08 | 9 | READ (`tool_monitor.rs` RepetitionInspector) |
| frankbria/ralph-claude-code | Bash | MIT | 2026-10-07 | 4, 9 | READ for circuit-breaker states; AVOID as base |
| snarktank/ralph | shell/md | MIT | 2026-02-01 | 4 | READ only (stale, trivial) |
| BloopAI/vibe-kanban | Rust+TS | Apache-2.0 | 2026-09-19 | 4 | READ for worktree-per-task UX; AVOID as base |
| dagger/container-use | Go | Apache-2.0 | 2026-08-12 | 3, 4 | READ (experimental; Dagger engine dependency) |
| textcortex/claude-code-sandbox | TS | no license file found | 2026-02-20 | 3 | AVOID (stale, no license) |
| stryker-mutator/stryker-js | TS | Apache-2.0 | 2026-09-11 | 5 | ADOPT as tool for JS targets (`--mutate file:line-range`, `--incremental`) |
| sourcefrog/cargo-mutants | Rust | MIT | 2026-10-08 | 5 | ADOPT as tool for Rust targets (`--in-diff`) |
| boxed/mutmut | Python | BSD-style | 2026-09-12 | 5 | ADOPT as tool for Python targets (check line-restriction support) |
| mkdocstrings/griffe | Python | ISC | 2026-10-08 | 5 | ADOPT as tool (`griffe check`) for Python public-API diff |
| Tufin/oasdiff | Go | Apache-2.0 | 2026-10-08 | 5, 11 | ADOPT as tool for OpenAPI; BORROW its breaking-change taxonomy (355 files in `checker/`) |
| microsoft/rushstack (api-extractor) | TS | MIT | 2026-10-06 | 5 | ADOPT as tool for TS targets (`.api.md` report diff) |
| actions/dependency-review-action | TS | MIT | 2026-09-30 | 5, 7 | READ for lockfile diff semantics |
| danger/danger-js | TS | MIT | 2026-10-01 | 5 | READ (rule-on-diff pattern); do not adopt |
| promptfoo/promptfoo | TS | MIT-style | 2026-10-08 | 6, 11 | READ; AVOID as judge core (no kappa, no order swap found) |
| UKGovernmentBEIS/inspect_ai | Python | MIT | 2026-10-08 | 6 | READ (scorer/log design); no kappa or swap found |
| braintrustdata/autoevals | TS/Py | MIT | 2026-10-08 | 6 | READ (LLM-classifier scorers) |
| ossf/osv-schema | schema | Apache-2.0 | 2026-10-06 | 7 | BORROW schema shape for the constraints registry |
| renovatebot/renovate | TS | AGPL-3.0 | 2026-10-08 | 7 | READ only; run as a separate process (already planned), never link or copy |
| dependabot/dependabot-core | Ruby | MIT | 2026-10-08 | 7 | READ (`cooldown_calculation.rb`); do not copy |
| smeijer/where-broke | TS | MIT | 2023-05-27 | 7 | READ (idea only; stale); write own bisect |
| humanlayer/humanlayer | TS/Go | Apache-2.0 | 2026-06-18 | 8 | AVOID (README says code is deprecated) |
| caronc/apprise | Python | BSD-2 | 2026-10-05 | 8 | READ/optional sidecar for later chat transports |
| actions/actions-runner-controller | Go | Apache-2.0 | 2026-10-02 | 10 | READ (ephemeral runner lifecycle) for the GitHub fallback only |
| assert-rs/snapbox (trycmd) | Rust | MIT/Apache-2.0 | 2026-10-04 | 11 | READ (CLI golden-file design); AVOID as dependency (Rust) |
| nodejs/undici (snapshot agent) | JS | MIT | 2026-10-08 | 11 | BORROW record/replay design for HTTP fixtures |
| ryoppippi/ccusage | TS | MIT | 2026-10-08 | 3, 11 | READ (Claude transcript JSONL parsing) |
| Netflix/pollyjs | JS | Apache-2.0 | 2025-05-31 | 11 | AVOID (stale) |
| karakeep-app/karakeep | TS | AGPL-3.0 | 2026-10-04 | 1 | AVOID (AGPL) |

## 1. Supervisor

The target: goal state machine, SQLite job/attempt store, leases, heartbeats, idempotent effects, crash recovery. No project found does all of this; the closest SQLite-backed TS durable engine is OpenWorkflow.

**openworkflowdev/openworkflow (Apache-2.0, 2026-10-07).** Read `packages/openworkflow/sqlite/sqlite.ts` and `sqlite/backend.ts` (1,321 lines). It uses the built-in `node:sqlite`, `PRAGMA journal_mode = WAL`, and `BEGIN IMMEDIATE` for every claim and transition. Schema (`sqlite.ts` lines 81 to 135): `workflow_runs` (status, `idempotency_key`, `attempts`, `worker_id`, `available_at`, `deadline_at`, `error`, started/finished timestamps) and `step_attempts` (`workflow_run_id`, `step_name`, `kind`, `status`, `output`, `error`), with a migrations table. Lease model in `backend.ts`: claim selects rows with `available_at <= now` and sets `worker_id` and `available_at = now + leaseDurationMs`; `extendLease` (around line 586) moves `available_at`; completion and failure statements all carry `AND worker_id = ?` so a worker whose lease was reaped cannot write a stale result (`runningWorkflowRunOwnedParams`, lines 65 to 92). Take: the lease-fencing predicate on every write, the `idempotency_key` column, and the step-attempt memoisation table, mapped onto our `attempt` and `effect` tables. Avoid: adopting the engine itself. Its workflow model is code-replay ("step.run"), not our goal DAG with a supervisor that must also hold the ledger and kill switch, and it brings a framework API surface. Requires Node 20+; `node:sqlite` is fine on Node 24 but conflicts with the plan's "better-sqlite3 and little else"; decide once.

**karakeep-app/liteque (MIT per package.json, no LICENSE file in repo, 2026-08-31).** `src/queue.ts` `attemptDequeue({timeoutSecs})` sets `expireAt` on claim and re-picks rows where `expireAt < now`; `src/runner.ts` has per-job timeout and run-number tracking. Read it as the minimal visibility-timeout pattern. Avoid depending on it: Drizzle, no heartbeats, no fencing. Its parent app karakeep is AGPL-3.0, so do not copy from there.

**dbos-inc/dbos-transact-ts (MIT, 2026-10-06)** and **restatedev/restate (BSL 1.1, 2026-10-08).** Already deferred by DESIGN.md. Restate is BSL and needs a server: AVOID. DBOS is worth one read of its "workflow status + step output as the idempotency record" idea, but it targets Postgres.

Not found: an off-the-shelf pattern for a lease that also covers an external container's lifetime (supervisor lease plus container label reaping). Write it; the OpenWorkflow fencing predicate is the part to copy.

## 2. Key proxy

This is the best-covered component, thanks to one Anthropic repo.

**anthropic-experimental/sandbox-runtime (Apache-2.0, 2026-10-07).** TypeScript, directly relevant. Files read in `src/sandbox/`:
- `http-proxy.ts` (1,356 lines), `tls-terminate-proxy.ts` (1,339), `socks-proxy.ts`, `mux-proxy.ts`, `parent-proxy.ts`: a forward proxy with CONNECT handling, optional TLS termination (`mitm-ca.ts`, `mitm-leaf.ts`) so HTTPS requests can be inspected at request level, and chaining through an upstream proxy with correct NO_PROXY semantics.
- `request-filter.ts`: a `filterRequest(Request) => {action:'allow'|'deny', reason, status, setHeaders, removeHeaders}` hook. This is exactly the "request-level check on an allow-listed host" the design needs (E8).
- `decider-client.ts`: a length-prefixed JSON protocol where the proxy asks an external process for a verdict per request (`req`, `verdict`, `need_body`, with a single timer and strict field validation). Our spend-cap and ledger check can sit behind this boundary (decider = supervisor-owned ledger), which keeps the proxy small and fail-closed on decider timeout.
- `credential-sentinel.ts`, `body-substitution.ts`, `credential-mask-env.ts`: the sandbox sees `fake_value_<uuid>`; the proxy swaps sentinel for real key only for allow-listed destinations, in headers and streamed bodies, with a fail-safe direction (a missed substitution sends the fake, never a leak). This is precisely "no credentials inside the sandbox, keyproxy injects".
- `resolved-address-guard.ts`: resolves once, drops loopback, link-local and cloud-metadata addresses, dials the checked address (defeats DNS rebinding onto an allow-listed name). Copy this logic whole.
- `domain-pattern.ts`, `address.ts`: allow/deny matching and CIDR handling.

Take: the request-filter contract, decider protocol, sentinel scheme, resolved-address guard. Gaps (ours to add): per-attempt identity and caps (the library is per-session, not multi-tenant), USD accounting from response `usage` (including SSE streams), a ledger, request logging to SQLite, mid-stream cutoff on cap. Caveat: it is positioned as a CLI/library for macOS Seatbelt and Linux bubblewrap sandboxing, not a standalone gateway; we would vendor the proxy modules or depend on the package, and `proxy-main.ts` suggests a standalone entry point worth reading first. Verify it runs on arm64 Linux in the Docker VM.

**BerriAI/litellm (MIT outside `enterprise/`, 2026-10-08).** Do not run it as the proxy (Python, huge surface, its own DB, dependency risk is the exact thing the factory fights). Do borrow: `model_prices_and_context_window.json` (3.1 MB), which carries per-model `input_cost_per_token`, `cache_read_input_token_cost`, `cache_creation_input_token_cost`, 1-hour cache write price and `_above_200k_tokens` tiers (the >100K/200K price step in DESIGN.md "5x cost above 100K" must be checked against this table, since Anthropic tiers differ per model). Also read `litellm/proxy/hooks/max_budget_per_session_limiter.py`: it accumulates response cost per `session_id` and returns 429 once over budget; note that this is a post-hoc check, so one request can overshoot. Design lesson for us: reserve an estimated worst-case cost before forwarding, then reconcile. Avoid the `enterprise/` directory (separate license).

**maximhq/bifrost (Apache-2.0, 2026-10-08, Go).** Read its virtual-key budget and rate-limit model in `core/` and `framework/` for field naming; do not adopt (Go service, UI, Terraform, Helm).

**musistudio/claude-code-router (MIT, 2026-10-08)** and **Portkey-AI/gateway (MIT, last commit 2026-05-25).** AVOID. Router rewrites model routing for Claude Code and stores keys in its config; Portkey is a provider-translation gateway without per-run hard caps in the OSS core and is the oldest commit here.

**Anthropic reference firewall.** `anthropics/claude-code/.devcontainer/init-firewall.sh` is an iptables/ipset allowlist for a container. Read it as the defence-in-depth layer beneath the proxy (container with default-drop egress, only the proxy reachable). Licence is "all rights reserved", so re-implement, do not copy. In our design the second layer is Docker network topology (`worker-net` with only `keyproxy` egress), which makes the iptables script unnecessary except as a reference for DNS handling.

## 3. Sandbox runner

**mattpocock/sandcastle (MIT, 2026-10-08).** TS. Closest thing to our runner: Docker, Podman and no-sandbox providers, worktree per run (`WorktreeManager.ts`, `createWorktree.ts`), session capture so a Claude session can be resumed (`SessionStore.ts`, `resumePrecheck.ts`, and the Claude provider in `AgentProvider.ts` ~line 1181, which builds `claude --print --verbose --output-format stream-json --model ... --resume ... -p -` and reads the prompt from stdin to dodge arg-length limits), idle timeout with an error (`Orchestrator.ts` ~line 108), bounded output tails (`boundedTail.ts`), `shutdownRegistry.ts` for signal cleanup, sync-in/sync-out of git state with recovery messages (`RecoveryMessage.ts`). Take: the stdin-prompt command shape, idle-timeout loop, shutdown registry, host/sandbox session-path mapping for resume. Avoid as a dependency: it is built on Effect, it passes provider env into the sandbox (we must not), its default model constant lags the plan, and it has no caps ledger. Note it targets `claude` not `--bare`; add `--bare` and config isolation ourselves.

**anthropics/claude-code-action (MIT, 2026-10-07).** `base-action/src/run-claude-sdk.ts` (301 lines) shows the supported way to call `query()` from `@anthropic-ai/claude-agent-sdk`, handling `SDKMessage` / `SDKResultMessage` types and writing an execution file; `parse-sdk-options.ts` (355 lines) maps config to SDK options; `retry.ts` is a small retry wrapper. Read for typed message handling. Its security posture (GitHub token held in the runner, prompt carries untrusted issue text) is what our design avoids.

**anthropics/claude-agent-sdk-typescript (proprietary terms, 2026-10-08).** The repo is docs, changelog and examples; the package is the dependency. Useful: `examples/session-stores/` defines a `SessionStore` adapter interface (S3, Redis, Postgres reference adapters) and `shared/conformance.ts`, a 13-contract conformance suite. A SQLite `SessionStore` backed by the supervisor DB would give us resume across container restarts without copying JSONL files, and the conformance suite is a ready test harness (component 11 pattern as well).

**ryoppippi/ccusage (MIT, 2026-10-08).** TS parser for Claude Code's local JSONL transcripts with dedup and per-model cost. Read its loader to see transcript edge cases (duplicate message ids across resumed sessions, which is the same restored-spend problem E18 describes for `--max-budget-usd`).

**dagger/container-use (Apache-2.0, 2026-08-12)** and **BloopAI/vibe-kanban (Apache-2.0, 2026-09-19).** Both give each agent a git branch plus an isolated environment. Read for UX and lifecycle ideas; adopting either brings a Dagger engine or a large Rust+React app. **textcortex/claude-code-sandbox**: stale (2026-02-20), no licence file: AVOID.

Not found: any open source runner that combines per-attempt container, a credential-free sandbox, and ledger caps. This remains ours to build, from sandcastle (lifecycle) plus sandbox-runtime (egress and credentials).

## 4. Orchestrators driving Claude Code headlessly

- **sandcastle** (above) is the only well-kept TypeScript one: iteration loop (`iterations`, `completionSignal`, `idleTimeoutSeconds` in `Orchestrator.ts`), fresh context per iteration, parallel runs on branches.
- **anthropics/claude-code `plugins/ralph-wiggum`**: `hooks/stop-hook.sh` reads `.claude/ralph-loop.local.md` (frontmatter `iteration`, `max_iterations`, `completion_promise`) and blocks exit to re-feed the prompt. It validates numeric fields and deletes corrupt state. Useful as a reference for what Stop-hook loops look like and why we do the loop in the supervisor instead (hook cap of 8 continuations in E18, state in a markdown file with no crash semantics). Licence: proprietary; read only.
- **frankbria/ralph-claude-code (MIT, 2026-10-07)**: Bash. Interesting parts are `lib/circuit_breaker.sh` (see section 9), `lib/file_protection.sh` (verifies required control files exist before each loop: a weak protected-path check), `lib/response_analyzer.sh`, `lib/sandbox_docker.sh`, and `.claude_session_id` persistence. AVOID as a base: Bash with jq state files, no atomic transitions.
- **snarktank/ralph (MIT, last commit 2026-02-01)**: PRD-to-progress.txt loop. Stale and trivial; the idea (fresh context, progress file, tests as the gate) is already in DESIGN.md.
- **PR-driving bots:** claude-code-action is the only maintained one found; OpenHands' resolver was not re-read here (prior waves). No TypeScript multi-session runner with a durable store was found beyond sandcastle (web search for "claude agent sdk orchestrator" returned nothing relevant).

## 5. Verifier and integrity checks

No single project covers protected-path plus test-diff policy; those are 50 lines of our own code (git diff `--name-status` against a path-glob policy). What exists is per-language tooling to adopt as separate processes:

- **Mutation testing on changed lines only.** `sourcefrog/cargo-mutants`: `--in-diff <file>` (`src/in_diff.rs`, `diff_filter_file`) restricts mutants to those within a unified diff: exactly the design. `stryker-mutator/stryker-js`: `docs/incremental.md` shows `npx stryker run --incremental --force --mutate src/app.js:5-7` (line-range mutate) plus incremental reuse of prior results; script the ranges from `git diff -U0`. `boxed/mutmut` (Python): check whether the current version supports restricting by line or function; not confirmed in this pass. Run them inside the verifier container, never trust a score the worker produced.
- **Surface/API diff.** `mkdocstrings/griffe` (ISC): `src/griffe/_internal/diff.py` defines typed breakages (`ParameterRemovedBreakage`, `ParameterAddedRequiredBreakage`, `ObjectRemovedBreakage`, `ReturnChangedTypeBreakage`, `BreakageKind` enum) and `griffe check` compares two refs: the Python public-API diff, ADOPT. `Tufin/oasdiff` (Apache-2.0): `checker/` has ~355 files of breaking-change rules with levels (ERR/WARN/INFO): ADOPT for OpenAPI, and BORROW its severity taxonomy for our `breaking`/`ambiguous` classes. `microsoft/rushstack` api-extractor: `.api.md` report you diff for TS packages. For JSON-schema diff and CLI `--help` diff, no maintained tool was found; use a plain structural diff of normalised JSON and of parsed help text (see section 11).
- **Lockfile and transitive diff.** `actions/dependency-review-action` (MIT): `src/` parses before/after dependency sets and reports added, removed, changed packages with vulnerability and licence data via the GitHub API. Read for the diff-set semantics; the API dependency means it cannot run offline against Forgejo, so write the lockfile differ per ecosystem (lockfile formats are small) and feed OSV for advisories.
- **Rules-on-diff.** `danger/danger-js` shows the pattern of a policy file evaluated over a PR diff. Do not adopt; it is tied to CI providers.

Not found: held-out test infrastructure or four-state (DepBench-style) checking as OSS; build from `git worktree` plus overlay of held-out files at verify time.

## 6. LLM-as-judge

Searches found no maintained library that does order-swapped pairwise judging with kappa gating and a multi-vendor panel.
- **promptfoo/promptfoo (MIT-style, 2026-10-08):** TS, `llm-rubric` and `select-best` assertions (see `test/evaluator/select-best-minimal.integration.test.ts`), multi-provider. Grep of `src/` for kappa/cohen found nothing relevant. Reading material for provider abstraction; do not adopt (huge surface, telemetry, red-team code).
- **UKGovernmentBEIS/inspect_ai (MIT, 2026-10-08):** best-designed scorer and log format (`model_graded_qa`, eval logs as files). No kappa or swap found. READ for how to store sample-level judge outputs.
- **braintrustdata/autoevals (MIT, 2026-10-08):** small LLM-classifier scorers in TS and Python; READ for prompt templates.
- **yishaik/kappa (web, unread):** zero-dependency Python script for judge-vs-human agreement and prompt linting; not cloned or verified. Cohen's kappa is 20 lines; write it in TS with a test against scikit-learn values computed offline.
Verdict: implement `swap(A,B)`, agreement-in-both-orders, kappa and panel voting ourselves; the DESIGN.md Phase 6 interface is small. Borrow only prompt templates and the sample-log shape.

## 7. Dependency decision layer

No OSS project implements a constraints registry with reason, evidence and revisit trigger; nothing implements canary-before-upgrade for arbitrary repos.
- **ossf/osv-schema (Apache-2.0, 2026-10-06):** `docs/schema.md`: `affected[].package.ecosystem/name`, `affected[].ranges[]` of type SEMVER/ECOSYSTEM/GIT with `events` (`introduced`, `fixed`, `last_affected`, `limit`), `affected[].versions`, `database_specific` and `ecosystem_specific` extension fields, `aliases`, `related`, `withdrawn`, `modified`. BORROW this shape for the constraint registry (a range of bad versions, an evidence link, plus our extension fields `reason`, `revisit_trigger`, `max_age`, `upstream_url`) so entries can also be exported as OSV and consumed by osv-scanner.
- **dependabot/dependabot-core (MIT, 2026-10-08):** `common/lib/dependabot/update_checkers/cooldown_calculation.rb`: `within_cooldown_window?(release_date, cooldown_days)`, per semver-bump cooldown days, and, importantly, when no publish date is available it applies no cooldown and emits a "Cooldown was not applied" notice. That is fail-open; our resolver must fail closed (or escalate) on a missing date. READ.
- **renovatebot/renovate (AGPL-3.0, 2026-10-08):** `lib/util/minimum-release-age.ts`, `lib/workers/repository/process/lookup/filter-checks.ts`, `minimumReleaseAgeBehaviour` option (`lib/config/options/index.ts`). Good reference for per-datasource release timestamps and behaviours when the timestamp is missing. AGPL: use as a separate process only, do not copy code.
- **Bisecting:** `smeijer/where-broke` (MIT, stale since 2023) does binary search over published npm versions with a test script; `@sigma/bisect` on JSR (web, unread) is a generic runner. Neither does paired reruns or boundary confirmation. Write our own: about 60 lines plus the DESIGN.md rules.
- **Native cooldown knobs (web, unread; verify before relying):** npm `min-release-age` (11.10+), pnpm `minimumReleaseAge`, Yarn `npmMinimalAgeGate`, uv `exclude-newer` with durations, pip `--uploaded-prior-to`. No native Cargo; third-party `cargo-cooldown` exists (unverified). These support the "enforce natively where available" line.
- **Canary / downstream testing:** only the generic pattern (swap candidate into the consumer's install, run its tests) surfaced from Esprima, Bazel, Qt; no tool.

## 8. Escalation

Weakest-covered area. **humanlayer/humanlayer (Apache-2.0, 2026-06-18):** the README states the code is mostly deprecated and rebuilt as a product: AVOID. Web search surfaced `agentbell` (PyPI, ntfy approve/deny with request IDs), Nofax (MIT, local approval bridge) and PraisonAI approvals (Telegram buttons) (all web, unread; not evaluated). **caronc/apprise (BSD-2, 2026-10-05)** is a mature notification multiplexer (ntfy, Telegram, Matrix...); useful later as a sidecar for the "user-owned chat transports" in DESIGN.md, not for the answer protocol. Plan: write the `question` table and strict-parse answer logic ourselves; ntfy action buttons (HTTP actions posting to the answer page over Tailscale) need no library, just `fetch`. The ntfy server repo was not cloned (I could not confirm its exact repository path in this session); read its docs on actions and publish-with-priority before Phase 5.

## 9. Stall and loop detection

Beyond OpenHands, three maintained references, all readable in an afternoon:
- **google-gemini/gemini-cli `packages/core/src/services/loopDetectionService.ts` (Apache-2.0, 781 lines).** Three layers: tool-call repetition (`TOOL_CALL_LOOP_THRESHOLD = 5`, SHA-256 of tool name plus args), content repetition (`CONTENT_LOOP_THRESHOLD = 10` over `CONTENT_CHUNK_SIZE = 50` char chunks with a bounded history), and an LLM check (`LLM_CHECK_AFTER_TURNS = 30`, interval adaptive 5 to 15 turns, `LLM_CONFIDENCE_THRESHOLD = 0.9`, a double-check model). Take the hashing, the chunk-repetition detector, and the adaptive LLM-check cadence; port to run on our stream-json events. The LLM layer should call a Haiku-class model through keyproxy and be advisory.
- **sst/opencode `packages/opencode/src/session/processor.ts`:** `DOOM_LOOP_THRESHOLD = 3`: if the last three tool parts have the same tool and identical input, it raises a `doom_loop` permission request. Simple model of "same call three times".
- **block/goose `crates/goose/src/tool_monitor.rs`:** `RepetitionInspector` with `max_repetitions`, consecutive-repeat counter and per-tool total counts: the "duplicate-call metric" the design wants.
- **OpenHands `openhands-sdk/.../conversation/stuck_detector.py`:** action+observation repetition, action/error streak, monologue, alternating pattern thresholds, scanning events since the last user message. Already known.
- **frankbria/ralph-claude-code `lib/circuit_breaker.sh`:** a CLOSED / HALF_OPEN / OPEN state machine with `CB_NO_PROGRESS_THRESHOLD=3`, `CB_SAME_ERROR_THRESHOLD=5`, `CB_OUTPUT_DECLINE_THRESHOLD=70` (percent), `CB_PERMISSION_DENIAL_THRESHOLD=2`, and `CB_COOLDOWN_MINUTES=30` before OPEN becomes HALF_OPEN. The no-progress-by-file-change and output-decline signals are cheap, outcome-based detectors that do not depend on the model's self-report. Defaults are the author's choice, not evidence.
Verdict: TS port combining gemini-cli hashing plus goose counts plus a ralph-style circuit breaker state, persisted as `event` rows.

## 10. Forgejo/Gitea Actions and ephemeral runners

Source access to `code.forgejo.org`, `codeberg.org` and `gitea.com` was blocked by the proxy (403), so the Forgejo runner, Gitea act_runner and Forgejo server were NOT read. From Forgejo docs found by search (unread beyond snippets; verify on the real instance, this is E19's open item):
- Ephemeral: registration with `--ephemeral` (via `forgejo forgejo-cli actions register`; the standalone `forgejo-runner register` form is described as deprecated) or an API `ephemeral` field; the server enforces one job and then deletes the runner and invalidates its token. Ephemeral mode requires `forgejo-runner one-job`; an unused ephemeral runner can idle forever (no timeout control).
- Bot triggering: the automatic `GITEA_TOKEN`/`FORGEJO_TOKEN` is designed not to trigger further workflows; a PR created with a separate bot user's PAT should trigger `pull_request` workflows (inference from the docs, not stated for PR creation explicitly).
GitHub-side references (readable, for the stated fallback): `actions/actions-runner-controller` (Apache-2.0, 2026-10-02) for ephemeral runner lifecycle and JIT config handling; also consider `nektos/act` (not cloned). For a self-hosted launcher in TS on the Mac, the pattern is: supervisor registers an ephemeral runner via API, starts a `docker run` with the one-job command, and reaps containers whose registration vanished.

## 11. Contract and conformance tests for CLIs with JSON/NDJSON output

- **assert-rs/snapbox + trycmd (MIT/Apache-2.0, 2026-10-04).** Rust, but the design is the model: `.toml`/`.trycmd` cases with command, args, expected stdout/stderr/exit code, `[..]` wildcards and redactions for unstable output, and `TRYCMD=overwrite` to regenerate goldens. Copy the pattern (normalise, wildcard, regenerate with review) into a small TS harness; do not take the Rust dependency.
- **nodejs/undici snapshot agent (`lib/mock/snapshot-recorder.js`, `snapshot-agent.js`, `snapshot-utils.js`, MIT).** Record/replay of HTTP with request hashing (`hashId`), header filtering and excluded URLs, stored as JSON. Borrow for HTTP-based upstreams and for fixtures of the Anthropic stream through keyproxy. Netflix/pollyjs is stale (2025-05-31): AVOID.
- **Anthropic `conformance.ts`** (above) as an example of a numbered contract list run against several implementations; mirror for fake-CLI shims versus the real CLI: the same suite must pass against both the shim and a live smoke run.
- **Stream-json fixture capture:** no maintained public corpus of `claude -p --output-format stream-json` fixtures was found; ccusage's loaders and sandcastle's stream parser (`AgentStreamEmitter.ts`) are the places to see real event shapes. Capture our own per DESIGN.md.
- **oasdiff / griffe** double as contract diffing for API upstreams (section 5).

## Top five things to copy before Phase 0

1. Lease fencing from OpenWorkflow: every completion/failure UPDATE carries `AND worker_id = ?` (and lease unexpired), claims use `BEGIN IMMEDIATE`, plus an `idempotency_key` column (`sqlite/backend.ts`, `sqlite.ts` lines 81 to 135). Put it into the Phase 0 schema and crash test.
2. The OSV range shape for the constraints registry (`ossf/osv-schema docs/schema.md`, `affected[].ranges[].events`) with our reason/evidence/revisit/max-age extension fields, so the schema is stable before Phase 2.
3. sandbox-runtime's request-filter contract and decider protocol (`request-filter.ts`, `decider-client.ts`), sentinel credential scheme and `resolved-address-guard.ts`, as the keyproxy skeleton; read `proxy-main.ts` and confirm it builds on arm64 first.
4. LiteLLM's `model_prices_and_context_window.json` as the seed price table (vendored, dated, hash-checked), and its pre/post-call budget hook shape, adding a reserve-then-reconcile step so a single request cannot overshoot the cap.
5. A stall-detector interface fed by stream-json events, seeded from gemini-cli (hash of tool+args, chunk repetition), goose (per-tool counts) and ralph (no-progress, same-error, output-decline, HALF_OPEN cooldown), with every trip written as an `event` row.

Close runners-up: sandcastle's stdin-prompt `claude --print --output-format stream-json` command shape and idle timeout; SDK `SessionStore` conformance suite for a SQLite store; `cargo-mutants --in-diff` and `stryker --mutate file:range` wrapped behind one "mutate changed lines" verifier step.

## Gaps and licence cautions

Components with no good reference: (1) a supervisor combining goal DAG, ledger, kill switch and container-lifetime leases; (7) known-bad version registry with revisit triggers, and canary-before-upgrade, which are ours to design (OSV shape and Dependabot cooldown logic are only parts); (6) order-swapped, kappa-gated, multi-vendor judging as a library; (8) asynchronous question queue with strict parsing and default-after-timeout (only ntfy-button snippets and a deprecated HumanLayer); (10) any Forgejo runner or Forgejo token-trigger source read (blocked hosts); (5) held-out-test and four-state check tooling, JSON-schema diff and CLI `--help` diff; (11) public stream-json fixture corpora.

Licence cautions: `anthropics/claude-code` and `claude-agent-sdk-typescript` repos are "all rights reserved / Commercial Terms": read, do not copy files. Renovate and karakeep are AGPL-3.0. Restate is BSL 1.1. LiteLLM's `enterprise/` is separately licensed. Two repos had no top-level licence file (liteque, textcortex/claude-code-sandbox); treat as unlicensed until confirmed.

Search terms tried: "claude agent sdk orchestrator loop multi-session runner"; "LLM gateway per-key budget spend cap MIT TypeScript"; "SQLite job queue better-sqlite3 lease heartbeat visibility timeout"; "durable execution SQLite TypeScript"; "dependency cooldown minimum release age"; "bisect dependency versions broke tests"; "canary upgrade downstream testing"; "forgejo-runner ephemeral one-job"; "Forgejo Actions GITEA_TOKEN does not trigger workflow"; "LLM-as-judge pairwise position swap Cohen kappa"; "human-in-the-loop approval ntfy telegram timeout". Direct clones were chosen from known project names; not searched for and therefore possibly missed: Temporal TS SDK, graphile-worker, Squid/Envoy ext_authz configs, MITM-based sandboxes other than Anthropic's, DeepEval, Pact, Mend/OSV-scanner internals.
