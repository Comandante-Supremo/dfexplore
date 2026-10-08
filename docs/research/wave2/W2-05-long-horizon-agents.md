# W2-05 - Long-horizon agent reliability: verification and deepening of doc 04

Researched 2026-10-07 (wave two).

## Access limitation (read this first)

Egress policy blocked almost every primary-source host from this session. Failures recorded:

- WebFetch returned `getaddrinfo ENOTFOUND` for arxiv.org (abs, html), export.arxiv.org, alphaxiv.org, ghuntley.com, docs.openhands.dev.
- curl via the agent proxy got 403 on CONNECT for arxiv.org, export.arxiv.org, ar5iv.org, api.semanticscholar.org, ghuntley.com, openreview.net, aclanthology.org, proceedings.neurips.cc, www.usenix.org, dl.acm.org, ieeexplore.ieee.org, metr.org, tbench.ai, huggingface.co, docs.temporal.io, sqlite.org. Per the proxy README these are org policy denials, not retried or routed around.
- Only www.anthropic.com and raw.githubusercontent.com were readable.

Tag legend. **[V]** = I read the page itself. **[S]** = I saw the claim only in a WebSearch result summary of the primary source (arXiv id and venue are real search hits, numbers are second-hand); this is NOT verified under the task rules and counts as unverified until someone reads the paper. **[U]** = unsourced or reasoning only. Only three pages were read in full: two Anthropic posts and Huntley's playbook README. Everything else is [S] or [U]. The task asked for primary-source verification and this session could not deliver most of it; treat the numbers below as leads with identified citations, and re-run the fetches from a host with arXiv access.

## Summary

- Reliability falls steeply with task length. METR's 50% horizon is far longer than the 80% horizon (about 4x shorter at 80% in early data) [S]. For a factory that needs unattended success, plan against the 80% horizon, not the headline.
- The best failure taxonomy found (MAST, Berkeley, arXiv 2503.13657) says step repetition (15.7%), unaware-of-termination (12.4%) and reasoning-action mismatch (13.2%) are among the most frequent modes, and that wrong or missing verification is about a quarter of failures [S]. This supports a non-LLM supervisor that owns termination and verification.
- Intrinsic self-verification does not reliably help: Huang et al. (ICLR 2024) report accuracy dropping after self-correction without external feedback [S]. Verification gains come from external signals (tests, execution). This supports doc 04's "criteria are runnable commands" and cuts against the claim that an LLM verifier alone suffices.
- Stall detection has concrete, shipped defaults (OpenHands: 4 identical action-observation pairs, 3 repeated errors, 6 ping-pong cycles) [S]. Evidence that failed runs are merely longer is confounded by task difficulty, so a step-count cap alone is a weak detector [S].
- SQLite + systemd is justified for this scale, but on engineering grounds, not on measured evidence; no source compared it with DBOS or Temporal for LLM agents. Temporal's own guidance puts LLM calls in Activities, so wave one's "replay fits poorly" claim is overstated [S].
- Huntley's original post could not be read. The official playbook repo README was read [V] and shows Ralph's design is more guarded than wave one's secondary sources imply (backpressure, planning vs building modes, disposable plan).

## Verified findings

### A. Horizon length and degradation

1. METR, "Measuring AI Ability to Complete Long Software Tasks", arXiv 2503.14499. 50% time horizon doubling about every 7 months since 2019, possibly faster since 2024. v1 (Mar 2025): Claude 3.7 Sonnet about 50 min; later version: o3 about 110 min [S]. In early data the 80% horizon was about 15 min against 59 min at 50% for Claude 3.7 (roughly 4-5x shorter) [S]. A critique (LessWrong/greaterwrong) argues 80% horizons are an order of magnitude uncertain depending on the fit [S, weak].
2. 2026 state of the frontier, from Wikipedia and forecasting pages, not METR [S, weak]: Claude Mythos 50% at least 16 h (17.4 h on one page), 80% about 3 h 6 min; METR says measurements above 16 h are unreliable on the current suite; for GPT-5.6 Sol the 50% horizon is 11.3 h if cheating counts as failure vs over 270 h under a permissive convention. Lesson: reward hacking and cheating swing the headline by 25x, so verifier design matters more than model choice.
3. Ord, "Is there a half-life for the success rates of AI agents?", arXiv 2505.05115 appeared in results; it models success as decaying exponentially with task length (constant per-step hazard). I did not read it [S: title only]. If true it gives a usable planning rule: P(success) = 2^(-t/T50), so an 8-hour task at a 2-hour T50 succeeds about 6%, and a task at half the T50 about 71%. Mark this [U] as a heuristic until the paper is read.
4. SWE-Bench Pro (Scale AI, Sep 2025, revised Nov 2025): 1,865 tasks, 41 repos, average patch about 107 lines over 4.1 files [S]. A July 2026 claim that about 30% of the 731 public tasks are broken is from a guide site and unconfirmed [U]. Terminal-Bench 2.0, arXiv 2601.11868 (Jan 2026): 89 hard tasks, frontier models and agents under 65%, smaller about 15% [S].
5. Behavioral studies: SWE-agent failed trajectories are 12.6% longer on Lite and 18.5% longer on Verified, but a later study (arXiv 2604.02547 / 2511.00197) finds length tracks task difficulty and within contested tasks resolved runs were slightly longer (44.0 vs 39.6 steps) [S]. SWE-smith reports most failures of SWE-agent-LM-32B are hitting the cost or turn limit [S].

### B. Failure taxonomies

6. MAST, Cemri et al., "Why Do Multi-Agent LLM Systems Fail?", arXiv 2503.13657 (v3 Oct 2025). 14 modes in 3 categories; kappa 0.88 on expert annotation; 7 frameworks; v3 dataset 1,600+ annotated traces (v1/v2 text says 200+ tasks; versions differ, cite the version) [S]. Reported rates over 1,642 traces from a secondary write-up: Step Repetition 15.7%, Reasoning-Action Mismatch 13.2%, Unaware of Termination Conditions 12.4%, Disobey Task Specification 11.8%, Incorrect Verification 9.1%, No/Incomplete Verification 8.2%, Task Derailment 7.4%, Fail to Ask for Clarification 6.8%, Premature Termination 6.2%. Category shares: system design 44.2%, inter-agent misalignment 32.3%, task verification 23.5% [S, secondary reproduction; figures not read in the paper]. Authors' conclusion: failures are largely architectural, not just model weakness [S]. These are rates of occurrence in traces, not cost or causal weight.
7. IBM/Berkeley applied MAST to enterprise SRE traces: one model showed +46% Premature Termination and +43% Unaware of Termination [S]. Termination failures are model-specific, so budget for per-model calibration.
8. Anthropic harness post [V] (read directly): two observed patterns, one-shotting the whole app and running out of context mid-feature, and later agents declaring the project done early. Table of four problems and fixes: declaring victory early (feature list), leaving bugs or undocumented progress (git + progress notes), marking features done prematurely (self-verification), spending time figuring out how to run the app (`init.sh`). Feature list was over 200 items for a claude.ai clone, JSON chosen because the model is less likely to overwrite it than Markdown; agents may only edit `passes`. No quantitative reliability numbers are given.

### C. Context rot, compaction, memory

9. Chroma "Context Rot" technical report (14 Jul 2025): 18 LLMs including GPT-4.1, Claude 4, Gemini 2.5, Qwen3; performance gets less reliable as input length grows even on simple tasks, worse with topically similar distractors [S]. Widely quoted specific percentages come from blogs, not the report [S].
10. Liu et al., "Lost in the Middle", arXiv 2307.03172 (TACL 2024): not fetched; I could not confirm anything beyond the title. Cite as [U].
11. ACE, arXiv 2510.04618 (ICLR 2026): names "context collapse" (iterative rewriting erodes detail) and brevity bias; uses delta updates; AppWorld average 59.5%, +10.6 points over baselines, latency reduced up to 86.9% [S]. Practical reading: never have an LLM rewrite the whole memory file; append and curate by small deltas.
12. MemGPT, arXiv 2310.08560: virtual-memory paging of context; the numeric document-QA results were not retrievable [S]. A-MEM, arXiv 2502.12110 (NeurIPS 2025): Zettelkasten-style linked notes, evaluated on LoCoMo conversational recall [S]; no numbers obtained. Neither addresses coding-agent long runs, so they are weak evidence for the factory.
13. Anthropic context-engineering post: wave one already cited; I did not re-read it this wave.

### D. Self-verification, reflection, search

14. Huang et al., "LLMs Cannot Self-Correct Reasoning Yet", arXiv 2310.01798 (ICLR 2024). Without external feedback performance can degrade. A secondary log reports GPT-4 on GSM8K 95.5% -> 91.5% -> 89.0% over two rounds, and multi-agent debate with 9 responses 83.0% vs self-consistency 88.2% [S]. Prior positive results often used oracle labels [S].
15. Reflexion, arXiv 2303.11366: 91% pass@1 on HumanEval vs 80% GPT-4 baseline, using execution feedback in an episodic memory [S]. Self-Refine, arXiv 2303.17651: about 20 points absolute average over 7 tasks [S]. The synthesis (my inference): reflection helps when the feedback is grounded (tests, compiler), and is neutral to harmful when the model grades itself.
16. Test-time search over software trajectories: SWE-Search (MCTS, ICLR 2025, arXiv 2410.20285) 23% relative improvement across five models [S]; SWE-Replay (arXiv 2601.22129) cuts cost up to 17.4% and improves up to 3.8% on SWE-bench Verified by branching from archived trajectories at critical steps [S]; R2E-Gym (arXiv 2504.07164) hybrid execution + execution-free verifier gives better best-of-n scaling than either alone [S]; SWE-World reports 55.0 -> 68.2 going from K=1 to 8 candidates with a verifier [S]. Early failure prediction on trajectory prefixes plus restart: 66.6% -> 71.8% at 25% false positive rate, token use down 15-20% (single 2026 preprint) [S].

### E. Stall and loop detection numbers

17. OpenHands stuck detector (docs): repeated identical action-observation pair 4+ times; same action producing the same error 3+ times; two-action ping-pong 6+ cycles [S]. DeepEval loop metric: default similarity threshold 0.85 for stagnation between consecutive outputs [S]. A Notion research note reports action entropy drifts before looping begins [S, blog-grade]. Anthropic "Building effective agents" [V]: stopping conditions such as a maximum iteration count are common; checkpoints for human input at blockers; evaluator-optimizer is advised only when criteria are clear and iteration measurably helps.

### F. Durable execution

18. DBOS: library-only, checkpoints workflows and steps in Postgres, resumes from last completed step; exactly-once for steps that do their work inside the same DB transaction as the checkpoint; the "tens of thousands of workflows per second" figure is a vendor claim [S]. I found no peer-reviewed DBOS paper in search results [S]. Postgres, not SQLite, is the documented production backend (wave one already said this).
19. Temporal's own blog states the Workflow must be deterministic and non-deterministic work (LLM calls, tools) belongs in Activities, with replay not re-asking the model for earlier decisions [S, vendor]. Side-effect guidance converges on idempotency keys from stable operation identity, not timestamps [S, blogs].
20. Academic items touching agent side effects: Cordon, arXiv 2606.17573 (speculative shadow state plus an effect outbox, commit or abort after validation) and Agent libOS, arXiv 2606.03895 (cites Sagas and RIFL-style durable completion records) [S, titles and abstracts only; not read]. Both support the `effect` table design in doc 04.

### G. Ralph

21. Huntley's playbook repo README (raw.githubusercontent.com/ghuntley/how-to-ralph-wiggum) [V]: three phases, two prompts, one loop; planning mode writes `IMPLEMENTATION_PLAN.md` without implementing, building mode does one task, runs tests, updates the plan, commits; fresh context each iteration; "backpressure" via tests, type checks, lints; one task per loop; "Let Ralph Ralph" (treat the plan as disposable and regenerate it when stale); run in a sandbox with permissions bypassed; tune with "signs" (prompt text, `AGENTS.md`). Proposed enhancements still being evaluated include LLM-as-judge binary checks and deriving tests from acceptance criteria. No measured results appear in it.
22. Original post ghuntley.com/ralph (title per search: "Ralph Wiggum as a 'software engineer'"): unreachable. Secondary summaries say the loop is `while :; do cat PROMPT.md | claude-code; done`, that prompts of about 40-50 lines outperform 200+ line ones (anecdote), and that uncapped loops burn credits [S, blogs]. No controlled evaluation of Ralph-style loops was found anywhere, including in this wave. Treat all Ralph effectiveness claims as practitioner anecdote.

## Corrections to wave-one docs

1. Doc 04 section 3: "Community evidence above supports an independent verifier because workers mark their own work done prematurely (Anthropic, same post)." Unsupported attribution. The Anthropic post [V] fixes premature marking with *self*-verification by the same coding agent using end-to-end browser tests, and lists a dedicated QA agent as an open question. The independent-verifier recommendation is wave one's design choice, now supported indirectly by MAST (verification failures about 23.5% of traces) [S] and Huang et al. [S], not by Anthropic.
2. Doc 04 section 4: "LLM agent steps are non-deterministic, so replay-based engines (Temporal/Restate determinism model) fit poorly unless every LLM call is a recorded activity." The caveat in the sentence is in fact how Temporal prescribes agents be built (LLM and tool calls as Activities, whose results are recorded) [S, Temporal blog]. "Fits poorly" should become "fits, at the cost of running a server and structuring everything as workflow/activity code". The case against Temporal is operational weight, not determinism.
3. Doc 04 section 2 and Open Questions: Huntley's write-up "not reached" - still not reached. But wave one's characterization of Ralph as a bare loop whose risk is "moved into the task list" omits that the playbook already prescribes backpressure (tests/lints), a planning-mode/building-mode split and a regenerable plan [V]. The listed failure modes come from a vendor blog (futureagi.com) and remain unverified [U].
4. Doc 04 summary: "Detect stalls with progress signals ... not with agent self-reports." Sound, but no thresholds were given and the "diff churn" / "state fingerprint" heuristics are unsourced [U]. Only repeated action/error/ping-pong thresholds have a shipped precedent (OpenHands) [S].
5. Doc 04 section 7: "Memory without pruning degrades retrieval (secondary source, theneuron.ai, weak)". Keep as weak; better-supported statements are context rot (Chroma) and context collapse (ACE) [S]. Replace the citation.
6. Doc 04 section 4: "Go gained SQLite in June 2026" and DBOS SQLite production-readiness remain unchecked (vendor blog, not reachable this wave). Do not rely on them.
7. BUILD_PLAN line 24 "Temporal/Redis deferred" and line 46 "Planner/worker/verifier loop ... stalls are detected without self-reports" are fine in direction, but "fresh-context iterations" should not be presented as evidence-backed: no controlled evaluation of fresh-context vs continued-context loops was found [U].
8. Doc 04 describes the MAST-like "premature DONE" as a Ralph-specific failure; MAST measures it at only 6.2% (premature termination) vs 12.4% for not knowing when to stop and 15.7% for step repetition [S], so loops and non-termination are the more frequent problem than early DONE.

## Implications and concrete changes

### To /home/user/dfexplore/docs/BUILD_PLAN.md

1. Phase 5 exit criteria: add "verification is grounded (executable checks); LLM-only self-grading may never flip a criterion". Evidence: Huang et al. [S]; Reflexion gains used execution feedback [S].
2. Phase 5: add a "horizon budgeting" rule: decompose any goal whose estimated human-time exceeds roughly the 80% horizon of the configured model (unknown for Sonnet-class; METR shows 80% is about 4-5x shorter than 50%) [S] into sub-goals with intermediate criteria. Guess: target sub-goals under 1-2 hours of human-equivalent work [U].
3. Phase 5: add a "cheating/reward-hacking" check: criterion and test files hash-locked; a verifier run that detects edits to tests, skipped tests or network fetches of oracle solutions fails the attempt. Evidence: Terminal-Bench cheating caveats and the 25x swing on METR scoring [S].
4. Replace the "turn and USD caps" line with explicit defaults (table below) and make them config, with Phase 5 logging the data to calibrate them. Open item 8 (stall-threshold calibration) stays open.
5. Add an optional Phase 5b experiment: best-of-N with an execution-based selector for hard `solve` goals (R2E-Gym, SWE-World, Satori-SWE all report gains from execution-grounded selection) [S]. Do not build tree search first.
6. Keep SQLite + systemd. Add a decision record: revisit DBOS-on-Postgres only if the supervisor grows beyond ~10 workflow types or resume logic exceeds a few hundred lines [U].

### Suggested defaults (label each)

| Parameter | Default | Basis |
|---|---|---|
| Identical action+observation repeats before kill/replan | 4 | evidence-backed precedent (OpenHands) [S] |
| Same action, same error fingerprint | 3 | evidence-backed precedent (OpenHands) [S] |
| Two-action ping-pong cycles | 6 | evidence-backed precedent (OpenHands) [S] |
| Consecutive-output similarity treated as stagnation | 0.85 | tool default (DeepEval) [S]; weak for coding |
| Step-count-only kill | do not use alone | length is confounded with difficulty [S]; use only as cost backstop |
| Per-attempt turn cap | set at about 2x the median successful attempt's turns, learned per goal kind | principle from SWE-agent cost-cap analysis [S]; number is a guess |
| Iterations with no new criterion verified (outer loop) before replan | 3 | guess [U] |
| Replans with no progress before escalate | 2 | guess [U] |
| Max attempts per goal before abandon/escalate | 8 | guess [U] |
| Budget cap per goal | 3-5x the median cost of past successes of that kind | guess [U], informed by failures clustering at the cost ceiling [S] |
| Early-restart classifier | defer to later phase | one preprint: 66.6 -> 71.8% resolved at 25% FPR [S] |

### To /home/user/dfexplore/docs/research/04-state-and-long-running.md

- Section 3: rewrite the attribution (correction 1) and cite MAST category 3 (no/incorrect verification, premature termination) as the support.
- Section 4: reword the determinism sentence (correction 2); state the real objection (extra server, workflow-code constraints).
- Section 6: add the OpenHands thresholds as precedent; add "kill on identical repeated tool call with identical result" and "detect hidden cycles" (repeated action sequences with no explicit error) [S]; note that hidden-cycle and entropy detection are unproven for this setup.
- Section 7: replace the weak citation with Chroma and ACE; add an explicit rule that notes are append/delta-updated, never rewritten wholesale (context collapse) [S].
- Section 2: add the playbook details [V] (backpressure, planning/building modes, disposable plan, signs in `AGENTS.md`). Add: "no controlled evaluation of Ralph exists in the sources found".
- Schema: add `attempt.action_fingerprint` (rolling hash of last K tool calls) and `attempt.test_tamper_flag`. Both are design suggestions [U].
- Add the effect-outbox pattern (Cordon-style staged outward effects, commit after validation) to section 5 as an option beyond the dedupe-key table [S].

### Is SQLite + systemd justified?

- For: single host, single writer, a few concurrent attempts; no source reports needing more. Anthropic's own long-running harness uses plain files + git [V]. DBOS's library model also works by writing checkpoints to a DB, which is what the `attempt`/`effect` tables already do [S].
- Against/risk: hand-rolled resume and idempotency are where bugs hide. Mitigate with the reboot/kill -9 acceptance test already in the plan.
- Alternatives: DBOS on a local Postgres is the nearest drop-in with the same semantics, at the cost of running Postgres and adopting its decorators. Temporal/Restate add a server and workflow-code discipline; sensible only if workflows become many and long-lived. No benchmark or paper comparing these for agents was found. Verdict: justified as a starting point on simplicity [U], not demonstrated superior.

## Remaining unverified items

- Every [S] number above needs checking against the paper. Highest priority: MAST per-mode percentages and trace counts (v1/v2/v3 differ); METR 50% vs 80% horizon values for current Claude models from metr.org; Huang et al. GSM8K tables; Ord's half-life paper; Chroma report details.
- Huntley's original ghuntley.com/ralph text and any measured Ralph evaluation (none found).
- Exact OpenHands detector semantics (windowing; whether thresholds apply to consecutive or total occurrences).
- Lost in the Middle (2307.03172), MemGPT and A-MEM quantitative results, Letta file-based memory benchmarks.
- Any peer-reviewed DBOS paper and Restate design paper; DBOS SQLite production-readiness.
- Whether Claude-family models specifically show the termination failures MAST measured (its tests used other frameworks/models).
- Generalization of METR horizons (benchmarked tasks) to dependency-update and greenfield-build goals.

## Sources

Read directly [V]:
- https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
- https://www.anthropic.com/engineering/building-effective-agents
- https://raw.githubusercontent.com/ghuntley/how-to-ralph-wiggum/main/README.md
- https://raw.githubusercontent.com/multi-agent-systems-failure-taxonomy/MAST/main/README.md (only: over 1K annotated traces; no taxonomy numbers)

Seen via search summaries only [S] (arXiv ids are real search hits; not read):
- METR: arXiv 2503.14499; Ord arXiv 2505.05115; Wikipedia METR page; greaterwrong 80%-horizon critique
- MAST: arXiv 2503.13657; sky.cs.berkeley.edu/project/mast; huggingface.co/blog/ibm-research/itbenchandmast; niteagent.com MAST write-up (percentages)
- Huang et al. arXiv 2310.01798 (ICLR 2024); beancount.io research log (GSM8K numbers)
- Reflexion arXiv 2303.11366; Self-Refine arXiv 2303.17651
- ACE arXiv 2510.04618 (ICLR 2026); MemGPT arXiv 2310.08560; A-MEM arXiv 2502.12110 (NeurIPS 2025)
- Chroma https://research.trychroma.com/context-rot
- Terminal-Bench arXiv 2601.11868; SWE-Bench Pro (Scale AI leaderboard and secondary pages)
- SWE-Search arXiv 2410.20285 (ICLR 2025); SWE-Replay arXiv 2601.22129; R2E-Gym arXiv 2504.07164; Satori-SWE arXiv 2505.23604; SWE-World arXiv 2602.03419
- Code-agent behavior: arXiv 2604.02547, 2511.00197; SWE-agent arXiv 2405.15793; SWE-smith arXiv 2504.21798
- OpenHands stuck detector docs (docs.openhands.dev/sdk/guides/agent-stuck-detector); DeepEval loop metric; Notion looping research note
- DBOS blog and architecture docs (dbos.dev); Temporal blog "Of course you can build dynamic AI agents with Temporal"; Cordon arXiv 2606.17573; Agent libOS arXiv 2606.03895
- ghuntley.com/ralph (title/snippet only); i-scoop.eu, kartit.net Ralph write-ups (secondary)
