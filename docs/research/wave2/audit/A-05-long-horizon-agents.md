# A-05 - Audit of W2-05 (long-horizon agent reliability)

Auditor pass 2026-10-07. Sources I read myself: Anthropic "Effective harnesses for long-running agents" (WebFetch), Anthropic "Harness design for long-running application development" (2026-03-24, WebFetch), the Ralph playbook README (raw.githubusercontent.com, 1,226 lines), a shallow clone of `OpenHands/software-agent-sdk` (HEAD 69e2688, 2026-10-07: `openhands-sdk/openhands/sdk/conversation/stuck_detector.py`, `.../conversation/types.py`), and a shallow clone of `multi-agent-systems-failure-taxonomy/MAST` (HEAD a70542e, 2025-07-23: README and `assets/taxonomy_v11_cropped-1.png`). arXiv, metr.org and temporal.io were blocked, so METR, Huang and Temporal could be checked only through WebSearch summaries.

## Verdict: TRUST WITH CAVEATS

W2-05 is honest about what it could not reach, and its two main corrections to doc 04 hold up. The verifier attribution is wrong in doc 04: I re-read the November 2025 post, and it uses self-verification. The Temporal "fits poorly" wording is overstated. The direction of the proposals is sound: grounded verification, supervisor-owned termination, keeping SQLite. Four things need fixing before anything goes into BUILD_PLAN:

1. The MAST per-mode percentages do not match the taxonomy figure in MAST's own repo. Correction 8 depends on those percentages and flips under the repo's numbers.
2. The OpenHands thresholds are the right numbers, but the doc gets their meaning wrong. "6 ping-pong cycles" is really 3 A/B cycles. "Same error 3+" really fires on the 4th consecutive error; a nudge comes at 3. A monologue detector (3) is left out.
3. The "Huntley playbook README [V]" is Clayton Farr's third-party synthesis, not Huntley's text.
4. W2-05 missed Anthropic's March 2026 harness-design post. That post does support a separate evaluator: "Separating the agent doing the work from the agent judging it proves to be a strong lever". So the independent-verifier design has Anthropic backing after all, just not from the post doc 04 cited.

## Claims table

| # | Claim (W2-05) | Rating | Evidence |
|---|---|---|---|
| 1 | Anthropic Nov 2025 post fixes premature "passing" with *self*-verification; a QA agent is only future work (correction 1) | SUPPORTED | Re-read. The table row reads "Self-verify all features. Only mark features as 'passing' after careful testing." Future work names "a testing agent, a quality assurance agent". The post also says initializer and coding agents differ only by prompt. **W2-05 is right and doc 04 line 51 is wrong.** Caveat: the Mar 2026 Anthropic post (W2-02 section N) does back separate generator/evaluator ("strong lever"; "Out of the box, Claude is a poor QA agent"). W2-05 says Anthropic gives no support, which is too strong. |
| 2 | Post details: >200 features, JSON because less likely to be overwritten, only `passes` editable, no quantitative numbers | SUPPORTED | Re-read; all match. |
| 3 | METR 80% horizon about 4-5x shorter than 50% (3.7 Sonnet: 59 min vs about 15 min) | PLAUSIBLE-UNCHECKED | WebSearch confirms 59 min at 50% and "reducing ... by a factor of five" going to 80%. The 15 min figure was not found. METR's Mar 2026 note says 80% horizons depend heavily on modelling assumptions. metr.org blocked. |
| 4 | 2026 frontier horizons (Mythos 16 h, GPT-5.6 11.3 h vs 270 h, "25x swing") | UNSUPPORTED as a basis for edits | Wikipedia/forecasting pages, tagged weak by the researcher, yet used as evidence in BUILD_PLAN edit 3. |
| 5 | Ord half-life rule P = 2^(-t/T50) | PLAUSIBLE-UNCHECKED | Arithmetic is right (8 h at T50 = 2 h gives 6.25%; half of T50 gives 70.7%). Paper not read; correctly tagged [U]. |
| 6 | MAST per-mode rates: step repetition 15.7%, unaware of termination 12.4%, verification category 23.5% | CONTRADICTED (by the repo's own figure); version-dependent | MAST repo README headline figure: step repetition 11.5%, unaware of termination 6.54%, premature termination 8.64%, no/incomplete verification 9.16%, incorrect verification 13.61%. Categories 37.17 / 31.41 / 31.41. A third site gives 41.77 / 36.94 / 21.30. W2-05's numbers come from one secondary write-up said to describe v3 (1,642 traces, LLM-annotated). The ordering of modes is not stable across versions. |
| 7 | MAST: 14 modes, 3 categories, kappa 0.88, 7 frameworks, 1,600+ traces | SUPPORTED (README + search) | README: "over 1K annotated MAS traces". Search: 1,642 traces, GPT-4 *and Claude* families. The v3 labels come from an o1 annotator (kappa 0.77 vs humans), not experts. W2-05's open item "whether Claude-family models show these failures" is partly answered: MAST includes Claude traces. |
| 8 | Correction 8: premature termination only 6.2%, so non-termination dominates early DONE | CONTRADICTED | Repo figure: premature termination 8.64% > unaware of termination 6.54%. The conclusion flips with the paper version. |
| 9 | Huang et al.: intrinsic self-correction degrades (GPT-4 GSM8K 95.5 -> 91.5 -> 89.0); prior gains used oracle labels | PLAUSIBLE-UNCHECKED | Multiple secondary summaries agree (beancount, liner, lunadong). The direction is widely replicated. arXiv blocked. |
| 10 | OpenHands stuck-detector defaults 4 / 3 / 6 | SUPPORTED as numbers; semantics MISSTATED | Code `types.py`: `action_observation=4`, `action_error=3`, `monologue=3`, `alternating_pattern=6`. In `stuck_detector.py`: action-error fires when streak `> threshold` (4 consecutive; a nudge at exactly 3). The alternating check takes the last 6 actions and 6 observations (3 A/B cycles, not "6 cycles"). All checks are on trailing events since the last user message. Equality ignores ids and metrics. The numbers are tool defaults with no published calibration; "evidence-backed precedent" overstates them. |
| 11 | Temporal guidance: LLM/tool calls go in Activities, so "fits poorly" is overstated (correction 2) | SUPPORTED (vendor, via search) | Search returned the Temporal post (Egger and Androulakis, 2025-11-12): Workflow code must be deterministic, and LLM calls go through Activities. temporal.io was blocked for fetch. Doc 04's own sentence already conceded "unless every LLM call is a recorded activity", so this is a reframe, not a reversal. Correction accepted. |
| 12 | Ralph playbook: planning/building modes, backpressure, one task per loop, disposable plan, sandbox, no measured results | SUPPORTED (content); MISATTRIBUTED | README lines 58-64, 147-201, 173-180. The author is Clayton Farr ("I'm (Clayton) still determining..."). It is a synthesis of Huntley's post and videos hosted at `ghuntley/how-to-ralph-wiggum`, also listed as derived from `ClaytonFarr/ralph-playbook` in W2-08. Not "Huntley's" or "official". The README also concedes "Ralph can go in circles, ignore instructions". |
| 13 | No controlled evaluation of Ralph / fresh vs continued context exists | PLAUSIBLE-UNCHECKED (absence claim) | My WebSearch also found only practitioner write-ups (Thoughtworks Radar, codecentric). Related: Anthropic's Mar 2026 post says resets vs compaction is model-dependent (needed for Sonnet 4.5 "context anxiety", dropped for Opus 4.5). That is first-party anecdote, not a controlled evaluation. |
| 14 | Failed SWE-agent trajectories longer, but confounded by difficulty; a step-count cap alone is weak | PLAUSIBLE-UNCHECKED | Search-only. The conclusion is reasonable regardless. |
| 15 | ACE "context collapse"; append/delta-update memory | PLAUSIBLE-UNCHECKED | Search-only. The rule is also consistent with doc 04's append-only progress logs. |

## Tag integrity

- Mostly honest. The access-limitation section is explicit, and [S] is defined as unverified. Good.
- Mis-tag: "Huntley's playbook repo README [V]" and correction 3, "the playbook already prescribes...". The page was read [V], but its authority is a third party. That weakens correction 3: it shows what a well-informed practitioner recommends, not what Huntley wrote.
- [S] numbers promoted to hard defaults. The OpenHands 4/3/6 values are labelled "evidence-backed precedent". They are shipped defaults, and with wrong semantics for two of them. MAST 6.2/12.4/15.7 [S] drives correction 8, which is a factual rewrite of doc 04. The "25x swing" [S, weak] is cited as evidence for BUILD_PLAN edit 3.
- "Supported by MAST (verification failures about 23.5%)" in correction 1: the figure is version-dependent (repo shows 31.41%). The direction holds; cite it without the number.
- W2-06 cites MAST as [V] with the same counts. That is consistent on structure, and neither doc verified the percentages.

## Overreach

- **Defaults table.** Acceptable as config defaults. Fix the semantics: ping-pong = 3 A/B cycles; same-error kill on the 4th consecutive; add monologue 3. Relabel the basis as "OpenHands shipped default, uncalibrated". Drop the DeepEval 0.85 similarity row: no coding evidence, and it overlaps the fingerprint checks. The OpenHands detectors are intra-conversation. To use them, the supervisor must parse `stream-json` tool_use/tool_result events live and kill or interrupt the run, since it cannot nudge a `-p` run. W2-05 does not say this. Outer-loop rows (3 / 2 / 8, budget 3-5x) are correctly marked guesses.
- **Horizon budgeting.** The direction is sound, but it cannot be operational. The 80% horizon for the configured Claude models is unknown, METR tasks are not dependency-update goals, and "estimated human time" for a goal is itself an unvalidated estimate. Keep it as planner guidance, not a Phase 5 exit criterion.
- **Test-tamper checks.** Largely duplicates BUILD_PLAN line 28 (protected paths, test-diff policy) and W2-04. W2-04 has the better evidence (METR 0.7% vs 30.4% with a visible scorer; RHB hardening). The genuinely new item is blocking network fetches of solutions and upstream fixes. That belongs in the sandbox egress policy.
- **action_fingerprint column.** Reasonable. Doc 04 already has `error_fingerprint` and `state_fingerprint`; this adds the tool-call dimension needed for the OpenHands-style checks. `test_tamper_flag` as a boolean is too thin; record integrity violations as `event` rows with type and evidence.
- **Correction 8.** Not supported; see claims 6 and 8.
- **SQLite + systemd "justified on simplicity".** Reasonable, and consistent with W2-02. The SDK provides no goal-level state machine, lease, ledger, idempotent effects or reboot recovery, and its `--bg` daemon dies on reboot. So a hand-rolled supervisor is needed either way, and W2-02 shows it can be thinner (store `session_id`, read `result.total_cost_usd`, pass `--max-turns`/`--max-budget-usd` per invocation). W2-05 does not cite W2-02 and should. The "revisit DBOS at >10 workflow types / a few hundred lines of resume logic" trigger is a guess, which is fine if labelled.
- **Best-of-N (Phase 5b).** Fine as deferred and optional. It overlaps Phase 6 compare infrastructure, so it should reuse that rather than be a separate build.

## Consistency with BUILD_PLAN and other docs

- Consistent: non-LLM supervisor, verifier as sole writer of `satisfied`, deterministic checks first (BUILD_PLAN lines 24-27), open item 8 (stall calibration).
- Correction 7 ("fresh-context iterations" not evidence-backed) agrees with my search. Also note that W2-02/Anthropic Mar 2026 says it is model-dependent.
- W2-08 already identified the playbook repo's ClaytonFarr lineage; W2-05 contradicts it by calling it Huntley's.
- W2-02 section N already lists the Mar 2026 generator/evaluator post; W2-05 should reconcile correction 1 with it.

## Missing

- Anthropic Mar 2026 harness-design post: first-party support for a separate evaluator, model-dependent "context anxiety", and cost reference ($200 / 6 h full harness vs $9 / 20 min solo).
- Flaky tests and nondeterministic CI as a source of false stall and loop signals. Detectors need a flake-retry or quarantine rule before counting "same error" repeats.
- How detectors attach to `claude -p`: live stream-json parsing plus the SIGINT-then-SIGTERM policy (W2-02). Otherwise intra-run detection is impossible and only post-hoc fingerprints remain.
- Budget-stop classification (spend-cap 429 is not a retry, W2-02). This interacts with the "max attempts" and "budget cap" defaults.
- Generalization caveat: MAST is multi-agent chat frameworks, and METR covers benchmarked tasks. Neither measures a code-supervised planner/worker/verifier loop.
- Pinned/"held" goals and the stall ladder: a goal that legitimately waits (pin, upstream fix) must not trip stall detectors. The state machine already has `held`; the detectors should exclude it explicitly.

## Recommended edits

### To BUILD_PLAN.md

| W2-05 edit | Decision | Reason |
|---|---|---|
| 1. Phase 5 exit: verification grounded; LLM-only self-grading never flips a criterion | ACCEPT | Consistent with lines 26-27; supported by the direction of Huang (search-level) and by both Anthropic posts. |
| 2. Horizon budgeting rule | MODIFY | Make it planner guidance ("decompose goals estimated over about 1-2 h human-equivalent; log actual durations to calibrate"), not an exit criterion; drop specific METR multipliers. |
| 3. Cheating/reward-hacking check | MODIFY | Merge into the existing anti-gaming line 28. Add only "sandbox egress denies fetching upstream fixes or solutions during `solve`/`update` attempts". Cite W2-04, not the "25x swing". |
| 4. Explicit stall defaults as config | MODIFY | Use the corrected OpenHands semantics (4 identical action+observation; kill on 4th consecutive same-action error, nudge at 3; 3 A/B cycles; monologue 3). Drop DeepEval 0.85. Require live stream-json parsing in Phase 1. Keep open item 8. |
| 5. Optional Phase 5b best-of-N | ACCEPT (deferred) | Reuse Phase 6 N-sandbox machinery. |
| 6. Keep SQLite + systemd, plus decision record | ACCEPT | Agrees with W2-02's SDK gap table; label the revisit thresholds [U]. |
| Correction 7 (fresh-context not evidence-backed) | ACCEPT | Add "model-dependent per Anthropic Mar 2026"; make fresh vs resumed a per-model config. |

### To doc 04

| W2-05 edit | Decision | Reason |
|---|---|---|
| Section 3: rewrite attribution (correction 1) | ACCEPT, MODIFY citation | The Nov 2025 post uses self-verification. Cite the Mar 2026 Anthropic post ("strong lever", "poor QA agent") plus MAST verification modes without the 23.5% figure. |
| Section 4: reword the Temporal determinism sentence (correction 2) | ACCEPT | Real objection is the extra server and workflow-code discipline. |
| Section 6: add OpenHands thresholds and hidden-cycle note | MODIFY | Corrected semantics as above; mark them as uncalibrated tool defaults. |
| Section 7: replace theneuron citation with Chroma/ACE; delta-only notes rule | ACCEPT | Mark Chroma/ACE as [S]. |
| Section 2: add playbook details and "no controlled evaluation" | MODIFY | Attribute to Clayton Farr's Ralph Playbook (derived from Huntley), not Huntley. Huntley's own post is still unread. |
| Schema: `attempt.action_fingerprint` | ACCEPT | |
| Schema: `attempt.test_tamper_flag` | MODIFY | Record as `event(type='integrity_violation', payload=evidence)`, consistent with "quarantine-by-builder counts as integrity violation". |
| Section 5: effect-outbox (Cordon) | MODIFY | Doc 04's `effect` intent/done table is already an outbox. Add one sentence only; the source is abstract-level [S]. |
| Correction 8 (premature DONE vs non-termination frequencies) | REJECT | MAST's own repo figure gives premature termination 8.64% > unaware of termination 6.54%. Keep doc 04's failure list and add that both modes are common. |
| Corrections 4, 5, 6 | ACCEPT | Unsourced heuristics flagged as such; weak citation replaced; DBOS/Go SQLite claims left unchecked. |
