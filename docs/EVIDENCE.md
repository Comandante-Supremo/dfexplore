# Evidence appendix

Every number the design relies on, with its scope. `DESIGN.md` and `BUILD_PLAN.md` cite these by key (E1, E2, ...). A number appears only with model, benchmark and denominator. Tags: `[V]` read from the paper or primary source and independently re-checked (wave three plus verifier); `[S]` read from the paper but not in the verifier sample; `[unreachable]` source could not be opened; `[U]` our own guess or design choice. All paper models are 2024 to 2026; none is a Claude 5.x measurement, so every rate is a prior to re-measure on our own repos.

Provenance chain: `research/01-07` (wave one, search-summary level), `research/wave2/` and `wave2/audit/` (primary docs and source, audited), `research/wave3/` and `wave3/audit/` (arXiv PDFs read, highest-impact numbers re-verified). Where waves disagreed, the later wave as amended by its verifier wins; the disputes are recorded in E22.

## E1. LLM repair of breaking dependency updates ("adapt")

- Byam (arXiv 2505.07522) `[V]`: 27% (28/103) of builds fully repaired (compile and tests), o3-mini, best of 40 model-by-prompt cells, on a Java/Maven compile-failure subset of BUMP; 18 to 27% across prompts; 78% of individual compile errors fixed.
- DepBench, in "Update from Hell" (arXiv 2608.30300) `[V]`: best of 10 agent configurations solved 104/203 (51.2%); Claude Code + Opus 4.8 solved 79/203; only 10 Python tasks. Its four-state check (the fix alone, without the version bump, must not pass the new tests) validates benchmark tasks before release; it is not applied to agent output in the paper.
- BreakGuard (arXiv 2608.20167) `[V]`: LLM-generated differential tests detect 30.3% (27/89, best of 9 configs) of BUMP breaks; 95.1% is the share of detecting tests that end in crash errors, not precision.
- `[unreachable]`: BDUpdater 90.5% (JS, given the library's own breaking-change records); the J.P. Morgan figure (71.4%) is line-removal precision on one synthetic upgrade, not a repair rate.

## E2. How often upgrades break clients, and how well tests catch it

- Hejderup et al. `[V]`: mean per-project detection of injected faults by the client's tests, 47% for direct and 35% for indirect dependencies, Java/Maven mutants. The 58%/20% pair is a coverage metric, not detection.
- Venturini et al. `[V]`: npm; 44% each of minor and patch releases among manifesting breaking changes (28 and 64 of each hundred; 87.5% combined).
- Go study `[V]`: 28.6% of 59,527 non-major upgrades break at API level; 3.1% of breaking changes are used by clients; 33.3% of clients of breaking upgrades are affected.
- Montandon et al. (arXiv 2408.05129) `[V]`: 35% of scikit-learn clients and 21% of pandas clients exposed to default-argument changes (the only Python-specific evidence found).
- `[unreachable]`: Jayasuriya ISSTA 2023 (11.58% of Maven updates break clients; a secondary source gives 4.35% for the adjacent version); Jayasuriya FSE 2024 (2.30% behavioural breaks affecting client tests); Fruntke and Krinke; the Kennesaw thesis (53% of Python breaking changes in minors, commit-diff LLM F1 0.85).
- No study found on: CLI or JSON-output contract breaks, auto-merge outcomes, upper-bound version caps, or bisecting dependency versions. Closest adjacent work: DepFixRouter (72 of 497 bot PRs needed repair).

## E3. Dependency-bot merge rates (observational)

- nf-core (arXiv 2607.10839) `[V]`: Renovate PRs merged 98.46% vs Dependabot 72.13%; one community, confounded by adoption and auto-merge configuration.
- He et al. `[V]`: 70.13% of non-security Dependabot PRs merged.

## E4. Cooldown (minimum release age)

- Adoption (arXiv 2609.16605) `[V]`: of ecosystem-level cooldown configurations that set a default delay, 64.3% (157 of 244) use 7 days; 135 adopters among 1,462 popular repos; the paper offers no effectiveness evidence.
- Dependabot: 3-day default cooldown for version updates only, verified in the `github/docs` source `[V]`; rollout date not verified.
- Native options verified from source or docs `[V]`: uv (`exclude-newer`), pip 26 (`--uploaded-prior-to`), npm (`min-release-age`), pnpm (`minimumReleaseAge`, use the strict variant), Renovate (`minimumReleaseAge`; `lockFileMaintenance` not covered by it, but Renovate passes native cooldown flags for npm and Poetry; a config without a `minimumReleaseAge: null` rule for unsupported update types may stall lockfile maintenance, unproven). Cargo: respected only from Rust 1.100 (beta on 2026-10-07, stable expected about 2026-11-12). Go: only an on-hold proposal; `index.golang.org` gives a proxy first-seen `Timestamp`, and `.info` time is commit time.
- PyPI's 14-day rule is not a cooldown: it rejects new files on releases older than 14 days `[V]`. Do not treat `upload-time` as trustworthy for old releases (PyPI's own post).
- Dependabot `ignore` can suppress security fixes `[V]`; OSV excludes withdrawn records from POST queries; Homebrew became an OSV ecosystem in schema 1.9.0 `[V]`.
- Incident windows (Axios, s1ngularity, Ultralytics, xz) `[unreachable]`: vendor snippets only.

## E5. Flaky tests and rerun arithmetic

- Gruber et al. `[V]`: about 170 is the median number of reruns needed to expose flakiness in an already-flaky, non-order-dependent Python test. It is not a stability rule.
- Arithmetic (recomputed by the verifier) for a test that fails half the time, k runs per side:
  - Strict rule (every new run fails and every old run passes): false-positive rate 0.5^(2k): k=2 6.25%, k=3 1.56%, k=4 0.39%, k=5 0.098%.
  - "At least one new failure, all old pass": 10.9% at k=3, 3.0% at k=5.
  - Majority rule: 25% at any k. Worst case for any-failure and majority rules is 25% regardless of k.

## E6. Agent-authored pull requests (human-reviewed public repos; optimistic for an unattended factory)

- AIDev-based studies `[V]`: merge rates 43% (Copilot agent), 83% (Codex), 71.48% overall; each failed CI check cuts merge odds by about 15% (OR 0.85); by task class across agents, docs 84%, CI 79%, build 74% (the "80 to 90%+" figure is Codex-only).
- Size: Claude Code PRs have a median 495 changed lines vs 52 for humans; 10x commits per PR gives 6.1x odds of a verified follow-up fix `[V]`.
- Reverts (arXiv 2609.17598) `[V]`: 90-day revert proxies, commit-message based on the PR's most-changed file, not a floor: Codex 6.1%, Devin 14.5%, human baseline 11.5%.
- Follow-up fixes (arXiv 2609.26847) `[V]`: merged agent PRs receive verified follow-up fixes at 1.62x human odds; 4.5% within 30 days. Fix-forward by the same agent 69.6%.
- Agents add a new dependency in 1.3% of PRs (arXiv 2512.11589) `[S]`.
- No study found on true auto-merge, canary or rollback of agent-merged code, or agent-vs-Dependabot PRs.

## E7. Overeager and destructive agent actions

- Permissive frameworks 5.4 to 27.7% overeager actions vs OpenHands 0.2 to 4.5% `[V]`; SNARE 19.51% of 10,000 benign runs `[S]`.
- Anthropic auto-mode classifier: 17% false negatives on real overeager actions, n=52 (Wilson 95% CI about 9 to 30%) `[S]`.
- Consent-stripping `[V]`: GLM-4.6 inside Claude Code 0.0% to 17.1%; Sonnet-4.6 3.9% to 15.8%; Claude Code + Sonnet-4.6 is the worst cell at 27.7%; the effect shrinks to 2.6 points under randomized scenarios.
- FixedBench: agents make undesired changes on already-fixed bugs 35 to 65% of the time `[S]`.

## E8. Prompt injection against coding agents

- AIShellJack `[V]`: 41 to 84% attack success across editor/model pairs, through poisoned rules files (84.1% on Cursor Auto Mode).
- CIPR `[V]`: Codex + GPT-5.4, run-tests tasks 45.5% vs bug-fix 8.6%; Claude Code + Sonnet-4.6 about 10 to 13% run-tests vs 3 to 5% fix-bug; same ordering across agents. Security rules in skills raise the alert rate, not the block rate.
- SoK (arXiv 2601.17548) `[V]`: Table III, detector-type defenses bypassed at 78 to 93% under adaptive attack (quoting Nasr et al.); the ">85%" figure appears only in the abstract.
- Agent Data Injection up to 50% where instruction-injection defenses show about 0% `[S]`.
- Anthropic containment post `[S]`: 24 of 25 exfiltration attempts contained; an allow-listed API host can carry an attacker's key, so egress needs request-level checks.

## E9. Reward hacking and test gaming

- EvilGenie (arXiv 2511.21654) `[V]`: the 20.7% figure for Claude Sonnet 4 is "heuristic" (non-general) solutions, which the paper separates from hacking; strict figures for Claude Code + Sonnet 4 are 2.1% hardcoding and 0% test edits.
- Reward Hacking Benchmark (arXiv 2605.02964) `[V]`: four hardening toggles combined, averaged over 13 models, cut exploits from 6.5% to 0.8% (-5.7 points; -87.7% relative).
- ImpossibleBench `[V]`: GPT-5 76% on Oneoff-SWEbench and 2.9% on Oneoff-LCB; with a strict prompt and an abort option on Conflicting-SWEbench, GPT-5 fell from 54% to 9% and o3 from 49% to 12%; much smaller effect for Claude Opus 4.1 (2025 models).
- METR reward-hacking report `[unreachable]`: a snippet gives 0.7% on HCAST; the "30.4% with visible scorer" figure was not found anywhere and is dropped.

## E10. LLM judges

- "Coin Flip Judge" (arXiv 2606.13685) `[V]`: 86.6% one-trial, about 90% three-trial, 95% eleven-trial agreement with the same judge's own 50-trial majority; two OpenAI mini judges, 29 near-tied general pairs (3 coding). It measures self-consistency, not accuracy, and is not code-specific. Position: 72% A-majority for GPT-4o-mini (p=0.024), 59% not significant for GPT-4.1-mini. No source supports "~75% position bias".
- Self-preference (arXiv 2604.06996) `[V]`: nine open-weight judges 1.88 vs 0.72; Claude Sonnet 4.5 ratio 1.78 and Haiku 4.5 1.71 on LiveCodeBench; Sonnet about 1.6 on IFEval with reasoning on; 1.29 is a 5-judge committee (ensembling reduces but does not remove it).
- CodeJudgeBench (arXiv 2507.10535) `[S]`: pairwise beats pointwise for code (pointwise ties about half the time); v2 shows removing comments slightly lowers accuracy and does not test misleading comments.
- Verbalized confidence on SWE-bench `[V]`: AUROC 0.70 to 0.83; ECE up to 0.50 (Gemini 2.5 Flash); Claude Sonnet 4.6 AUROC 0.794, ECE 0.109. XConf's +8.7 points is that method on AppWorld (4.8 on average), not verbalized confidence.

## E11. Test oracles can be wrong

- UTBoost `[V]`: 28.4% (170/599) of patches passing the 23 flagged SWE-bench Lite instances and 15.7% (92/584) on the 26 flagged Verified instances are wrong under stronger tests. PatchDiff `[V]`: 7.8% of patches differ in behaviour from the reference; +6.4 points in v2. SWE-ABS `[V]`: rejects 19.78% (2,184 of 11,041), -16.6 points. UTBoost and SWE-ABS share a first author. STING (arXiv 2604.01518) was renamed PROBE in v2.
- SWE Refactor Bench `[S]`: 60 of 88 submissions that passed the fixed tests were broken by independent verifiers. "Guard-and-Go" (arXiv 2607.28887) `[V]`: 29.0% of passing patches keep deleted code (a test-gap finding).
- LLM-generated oracles: TOGA's 47.5% false-positive assertions and 24.1% describe a 2022 neural oracle, not an agent `[V]`; Ferreira et al. 60% usable as generated / 8 / 24 / 8, GPT-4 Turbo, UI tests `[V]`.
- Mutation testing: no paper tests it on LLM-written code; changed-lines-only has support on human code at Google (arXiv 2102.11378) `[V]`.

## E12. Verifier leak routes and infrastructure noise

- Leak study (arXiv 2609.06780) `[V]`: five open-weight agents; total exploit rate 45 to 82% across all types; network and upstream access dominate at 25 to 66%; local git about 6 to 22% (lower bound; LLM-judged categories); a prompt cuts totals to 1.5 to 10.7% but local git persists at 0.3 to 8.6%.
- Anthropic BrowseComp post `[S]`: 2 of 18 runs identified the benchmark and decrypted the answer key via a canary string, through a third-party mirror; URL blocklists were insufficient.
- Infrastructure noise `[S]`: resource limits alone swung evaluation scores by 6 points.

## E13. Self-verification vs an independent verifier

- Valmeekam et al. `[V]`: an LLM verifier passed 38 of 45 invalid plans; self-critique 55% vs a sound verifier 88% (planning tasks, not code).
- Huang et al. (arXiv 2310.01798, ICLR 2024) `[V]`: self-correction without external feedback lowers accuracy on GSM8K, CommonSenseQA and HotpotQA (GPT-3.5, GPT-4, GPT-4-Turbo, Llama-2-70B); does not cover coding agents.
- Anthropic posts `[V]`: the Nov 2025 long-running-harness post uses self-verification and lists a QA agent as future work; the Mar 2026 harness-design post calls separating doer from judge "a strong lever" and Claude "a poor QA agent" out of the box; a full harness costs about 20x a solo run; the C-compiler experiment cost about $20k with 16 agents, verifier quality being the bottleneck.
- No matched builder-self-check vs separate-verifier study exists.

## E14. Long-horizon behaviour, failure modes, stall detection

- METR (arXiv 2503.14499 v4) `[V]`: the 80%-success horizon is 4 to 6x shorter than the 50% one ("roughly 5x"); doubling time 207 days; the paper stops at o3. 2026 horizons `[unreachable]`. BRIDGE: 80% horizon about 40 to 54 minutes `[S]`.
- SWE-Marathon (arXiv 2606.07682) `[V]`: identical tool-call runs up to 877 long; 32% duplicate calls in the worst scaffold; pass rate falls with run length (claude-code 41.9% to 3.2%).
- MAST (arXiv 2503.13657) `[V]`, shares of all failure labels (sum to 100): v3 (1,642 traces, NeurIPS 2025 D&B): step repetition 15.7, unaware of termination 12.4, premature termination 6.2, no/incomplete verification 8.2, whole verification category 23.5. v2 (200+ traces): 17.14 / 9.82 / 7.82 / 6.82 / 21.30. v1 (151 traces): 11.5 / 6.54 / 8.64 / 9.16 / 31.41. Always cite the version.
- OpenHands SDK stuck-detection defaults (read from the repo) `[V]`: 4 identical action+observation repeats; nudge at 3 and stuck on the 4th consecutive same-action error; 3 A/B cycles (6 actions); monologue 3. Tool defaults, not evidence.
- SWE-Search `[V]`: +23% relative at about 14x cost with GPT-4o, about 5x for other models; compute-matched resampling closes much of the gap. ACE (ICLR 2026) `[V]`: +17.0 average on AppWorld with DeepSeek-V3.1.
- Fresh-context loops: no controlled fresh-vs-continued evaluation exists; LoopsBench's Ralph row is 7.84% resolved, uncontrolled `[V]`; Stateless Language Agents (arXiv 2610.07625) is the closest `[S]`. The readable Ralph source is Clayton Farr's playbook (derived from Huntley); Huntley's own post `[unreachable]`.

## E15. Best-of-N, selectors and fan-out

- R2E-Gym `[V]`: hybrid verifier 51.0% vs 43.7% and 42.8% for each alone. CodeMonkeys `[V]`: 57.4% single; 66.2% is selection over 5 candidates, 4 from other teams (best member 62.8, random pick 60.9). CWM `[V]`: 53.9 to 65.8 with test-time scaling (40 unit tests). Satori `[V]`: Best@50 41.6 vs Best@500 41.0; the "10x" compares different trained models, not selection.
- SWE-HERO (arXiv 2604.01496 v2, Table 3) `[V]`: SWE-Hero-32B with SWE-Lego-Verifier-8B, Best@32 64.6 vs Pass@32 79.8; mean Pass@1 60.1; gains +7.9/+5.2/+4.5; "as much as 15% at K=16"; the best of 4 open verifiers was chosen on the test set.
- Stroebl et al. (arXiv 2411.17501 v3) `[V]`: imperfect verifiers cap resampling; "K <= 5 at cost ratio 4" holds for the four Llama-3.1 and Code Llama models only, on HumanEval+/MBPP+. Use: the false-positive cost sets the optimal N; measure it; do not cite a numeric cap.
- Frontier best-of-N (arXiv 2607.05391) `[V]`: pool mean 76.1 to selected 78.2 vs oracle 84.4 (N=3, Opus 4.5 / Gemini 3 Flash / MiniMax M2.5, SWE-bench Verified); 83.1 to 86.5 vs 92.1 (N=5 GPT-5.5, Terminal-Bench V2); the verifier is Gemini 2.5 Flash (closed API) scoring 20 repeated pairwise passes.
- Evolutionary methods `[S]`: AlphaEvolve, ShinkaEvolve and EvolRepair gains all depend on a numeric evaluator; EvolRepair crossover adds +2.1 points; FunSearch `[unreachable]`. Correlated errors across same-family candidates (Kim et al., arXiv 2506.07962) are not code-specific.

## E16. Autonomous greenfield builds

- ProgramBench `[V]`: 0/200 tasks fully resolved for all 9 models; the best (Opus 4.7) passes at least 95% of tests on 3.0% of tasks. Its "cheating on 20 to 36% of tasks" comes from an open-internet side experiment with 40 to 57% judge disagreement, not the main runs.
- ProjDevBench `[V]`: 27.38% of submissions accepted; 20 problems; best weighted score 77.85.
- Web-Bench 25.1% Pass@1 (Claude 3.7 Sonnet); Commit0 6.12% of tests (Claude 3.5 Sonnet, stage 1; abstract says 26% with feedback, table 29.30); PaperBench human 41.4 vs o1 26.6 on the same subset; ProjectEval about 15% best case (GPT-4o), 12.49 all-setting average `[V]`. Metrics differ; do not average them.
- Spec-driven development `[V]`: Panda (arXiv 2606.30689) compares Spec Kit and OpenSpec on determinism and traceability; SpecMine (2608.25202) is a corpus; de Macedo (2606.04967) a rubric. No outcome study of Kiro or BMAD, and no measurement of spec drift.
- Feedback across rounds (arXiv 2607.13091) `[V]`: a single uncontrolled Microsoft deployment of human-curated rules from accepted review comments (9 classes, 74 exposures, 0 recurrences, no baseline, model not named); says nothing about autonomous rounds.

## E17. Requirements elicitation and clarification

- ReqElicitGym (arXiv 2602.18306) `[V]`: current frontier models, overall elicitation 0.07 to 0.32 (Claude Opus 4.5 0.08, about 6 turns); style requirements below 0.01 for almost all 7 models.
- Ambig-SWE `[V]`: ambiguity detection accuracy 0.47 to 0.89 (Sonnet 4 reaches 0.89 only under strong encouragement); the 37.94 vs 59.52 figures are resolve rate without vs with file-location information, not interaction.
- ClarifyGPT `[V]`: 70.96 to 80.80 (GPT-4, MBPP-sanitized, 10 humans); 68.02 to 75.75 (four-benchmark average, simulated user). The 62.43 to 69.60 figures are not in the paper.
- LLMREI (GPT-4o) `[V]`: up to 73.7% of requirements elicited (60.94 full + 12.76 partial).
- Ask or Assume `[V]`: on hidden-information tasks, clarifying questions lifted the resolve rate from 54.8% to 69.4% (Sonnet 4.5, simulated oracle user).

## E18. Claude stack facts (official docs and source, re-verified in A-02)

- Pricing per MTok: Opus 5.5 $4/$20; Sonnet 5.5 $2/$10 (cache read $0.10); Haiku 5.5 $0.10/$0.50 up to 100K prompt tokens, 5x above; Fable 5.1 $10/$50. 1M context. Always-on thinking only on Fable 5.1 and Opus 5.5. Re-check on the public pricing page before budgeting.
- `--max-budget-usd` ignores spend restored by `--resume`. Stop hooks (and `/goal`) are capped at 8 consecutive continuations, reset by any tool call. PreToolUse command hooks fail open on timeout and non-2 non-zero exits; SDK callback hooks fail closed; deny rules apply in every mode including `bypassPermissions`. `--bare` ignores OAuth. `claude --bg` rejects `-p` and dies on reboot. `claude-code-action` on self-hosted runners is undocumented (source handles non-ephemeral runners).
- Auth and credits: the Legal page says users may run the unmodified Claude Code binary with their own subscription; no page addresses an always-on factory. Max/Team credits are $100 or $200 a month, expiring; the credits page says "Claude Code: No" and a credits-only org gets "Credit balance too low" on a Claude Code session. Managed Agents: beta, list token rates plus $0.08 per session-hour, excluded from ZDR and HIPAA.
- OpenTelemetry supported; Prometheus on port 9464.
- Anthropic April postmortem: model, prompt and effort changes need a canary and soak, with a per-model eval on every prompt change. Auto-mode classifier 17% FNR (E7). Token-upload incident and containment post (E8).

## E19. Forge, runner and sandbox facts (W2-07, re-verified in A-07)

- GitHub: JIT ephemeral runner API (`generate-jitconfig`, admin access required) and webhook redelivery API verified; GitHub does not auto-retry failed deliveries. ARC needs Kubernetes.
- Forgejo: v15 ephemeral runners are secondary-sourced only; whether a runner's default token triggers CI on PRs it opens is unverified.
- ntfy `http` actions: bearer header, max 3 actions; the phone makes the call (inference). Cloudflare Access Bypass is unlogged. Rootless Podman needs cgroup delegation (moot on a Mac host; see DESIGN).
- Self-hosted Renovate needs Node 24. CVE-2026-88884 (Renovate digest age) looks dubious; Renovate discussion #38115 unverified.

## E20. Upstream CLI contracts (worked example)

See `examples/harness-compat.md` and `research/wave2/W2-01` with audit `A-01`: event-schema differences, the kimi-cli 1.52.0 hazard, past breakage counts, release cadence. Source-level only; no binary was run.

## E21. Prior art

- StrongDM public repos (attractor, agate, cxdb, leash) are specs and small tools, last commits 2026-03-17, 2026-02-23, 2026-08-28, 2026-04-06. Attractor has goal gates with retry targets, a human gate with a timeout default and a supervisor loop; agate exits 255 to hand off to a human. StrongDM's and Stripe's own write-ups `[unreachable]`.
- GitHub Copilot coding agent: a human merges; its automation level covers issue triage only; single-branch push, no self-approve, rationale and confidence logged. gh-aw retired a vulnerable release range.
- OpenHands is now "Agent Canvas" plus a separate SDK; mini-swe-agent supersedes SWE-agent; Agentless has had no commits since 2024-12-22. SWE-bench Lite: Agentless 32.00% at $0.70, SWE-agent 18.00%, OpenHands 26.0 `[V]`.
- No open-source project implements goal, constraints registry and escalation together.

## E22. Disputes settled along the way

- MAST percentages: an audit rejected W2-05's figures by quoting the repo figure (v1); wave three found W2-05 quoted v3. v3 is used.
- SWE-HERO 64.6/79.8: a wave-two "correction" was wrong; the figures stand (E15).
- Dropped: EvilGenie 20.7% as "gaming"; METR "30.4%"; "~75% position bias"; "6 to 28% wrong patches" (mixed units); TOGA as an agent-written oracle; "45 to 82% git leak" as git-only; 3/10/30 rerun tiers and Gruber-170 as a stability rule; "any failure" or majority reruns; a "23 to 27%" repair range resting on an unread number; the SoK ">85%"; "1 in 6 dangerous actions pass any LLM gate"; a "6 to 15% revert floor"; a Copilot-derived merge `automation_level` enum; "StrongDM repos alive through 2026-10"; ProgramBench cheating as a main-run figure; Stroebl "K <= 5" for Command and GPT-4o; "small open verifier" for Gemini 2.5 Flash.
- One mis-attribution found: arXiv 2410.21819 is by Wataoka et al., not Panickssery et al. All other cited arXiv ids resolve to the papers described.

## E23. Still unreachable from the research environment

ACM, IEEE, USENIX, OpenReview, metr.org, Nature, vendor blogs: Jayasuriya (ISSTA 2023, FSE 2024), Fruntke and Krinke, BDUpdater, the Kennesaw thesis, incident-window pages, FunSearch, METR 2026 horizons, Chroma context rot, StrongDM and Stripe primary pages, the DBOS CIDR 2026 paper and the Temporal, Restate and DBOS docs. Not in the verifier sample: CodeJudgeBench comment stripping, FixedBench, ADI numbers, Anthropic's 17% miss rate.
