# A-06 Audit: W2-06 options and build literature

Auditor: independent model, 2026-10-07. Target: `docs/research/wave2/W2-06-options-and-build-literature.md`. Context: `docs/BUILD_PLAN.md`, `docs/research/06-options-and-autonomous-build.md`.

Method: the same network block applied (WebFetch to arxiv.org gave `ENOTFOUND`). I re-checked claims with fresh `WebSearch` queries (also abstract-level and model-summarised, so not independent of the researcher's channel) and with project READMEs fetched from raw.githubusercontent.com (bytedance/web-bench, R2E-Gym/R2E-Gym, commit-0/commit0, sani903/InteractiveSWEAgents). I read no paper body. "SUPPORTED" below therefore means "re-obtained from a primary listing or the authors' own README", not "checked against a paper table".

## Verdict

**TRUST WITH CAVEATS.** Most headline numbers hold at abstract or README level: CodeMonkeys, CWM, Satori, R2E-Gym, Stroebl, Web-Bench, PaperBench 21.0, and the existence and venues of Ambig-SWE, ClarifyGPT, LLMREI and ReqElicitGym. The doc is careful to label its guesses. It has one material error. Correction #1, which says wave one's SWE-HERO figures could not be confirmed, is **contradicted**. A search extract of the paper's Table 3 shows Best@32 = 64.6 and Pass@32 = 79.8 for SWE-Hero-32B with the SWE-Lego-8B verifier. Wave one was right, and W2-06 treated "my search found nothing" as if it were evidence against the claim. Three further weaknesses:
- The **[V]** tag is applied to some numbers that only came from secondary summaries: Commit0, ProjectEval, and the Web-Bench 65.4% comparison.
- The build-benchmark paragraph lumps four different metrics together (task Pass@1, % of unit tests passed, rubric score, Pass@5 of test cases) under "fail most of the time".
- All the build and selection numbers come from 2024 to mid-2025 models (Claude 3.5/3.7 Sonnet, GPT-4o, open 32B models). The doc flags this, but its N defaults still lean on them.

Of the proposed edits, the elicitation and selector-measurement edits are well grounded. "Hybrid verifier required" and "evolutionary mode" go beyond the evidence.

## Claims table

| # | Claim (as stated in W2-06) | Rating | Evidence |
|---|---|---|---|
| 1 | CodeMonkeys 57.4% SWE-bench Verified (Sonnet 3.5); 66.2% from selecting over an ensemble of other submissions' edits; ~$2,300 | SUPPORTED | Search of arXiv 2501.14723 and the Stanford blog; SWE-bench experiments submission 20250121_codemonkeys = 287/500 = 57.4%. 66.2% is selection across *other systems'* candidates, not CodeMonkeys alone. |
| 2 | CWM 53.9% → 65.8% with test-time scaling | SUPPORTED | Meta publication page, DeepLearning.AI and other summaries. Metric caveat: the paper calls 65.8 "pass@1 with TTS", while a DAIR summary calls it best@k. "40 unit tests" stays [U]. |
| 3 | Satori-SWE 41.6 Best@50 matches Llama3-SWE-RL-70B Best@500; greedy 35.8; different model and RL-trained refinement | SUPPORTED | The paper's table (via search) gives 41.6 vs 41.0 (Agentless Mini), greedy 35.8. W2-06's caveat (different model, trained-in loop) is correct and important. |
| 4 | R2E-Gym: each verifier type plateaus ~42-43%, hybrid 51%; 34.4% pass@1; 8.1K tasks | SUPPORTED | R2E-Gym README (raw.githubusercontent) gives 51%, 34.4% and 8.1K. The 42-43% plateau comes from the abstract via search (43.7 / 42.8). Both verifiers are *trained* models (R2E-Gym's own), not prompted LLM judges. |
| 5 | Stroebl, Kapoor, Narayanan (ICLR 2026): optimal attempts often <10 under imperfect verifiers; HumanEval/MBPP | SUPPORTED (abstract) | ICLR 2026 poster page and mlanthology; abstract wording matches. The "<10" depends on an assumed cost of false positives, and the evidence is from weak models on saturated function-level benchmarks. It is not a constant for agentic Claude 5.x runs. |
| 6 | SWE-HERO correction: "64.6 / 79.8 not confirmed"; replace with [S] gains 7.9/5.2/4.5 and a "~15% gap at K=16" | **CONTRADICTED** | Search extract of 2604.01496 Table 3: 32B with the SWE-Lego-8B verifier gives Best@32 64.6 and Pass@32 79.8 (gap 15.2). Gaps across verifiers: 7B 18-23, 14B 16-18, 32B 15-16 points at **K=32**. W2-06's "K=16" and its point-gain figures do not match this extract. Existence, 62.2% (32B) and 55.7% (no stage one) are SUPPORTED. |
| 7 | Web-Bench: 50x20 sequential tasks, 4-8 senior-hours, Claude 3.7 Sonnet 25.1% Pass@1 | SUPPORTED | bytedance/web-bench README, verbatim. The "versus 65.4% on SWE-bench Verified" comparison is not in the README (it says only "lower than SWE-bench Full and Verified"): PLAUSIBLE-UNCHECKED. |
| 8 | Commit0: ~17% of tests on easier libraries, ~6% on all, 26% with feedback; no agent fully reproduced a library | PLAUSIBLE-UNCHECKED (tag overstated) | The abstract (via search) confirms only "none fully reproduce" and "interactive feedback helps". 17%/26% come from a Moonlight review and 6.12%/29.30% from alphaXiv, both secondary. Should be [S]. The metric is % of unit tests passed, not % of builds. |
| 9 | PaperBench: best agent 21.0% (Claude 3.5 Sonnet New, open scaffold) | SUPPORTED | arXiv, OpenAI and ICML abstracts. The metric is a rubric replication score, not task success. The human 41.4% is not found (the ICML lay summary says 41%, and also gives "27%" for the best agent); W2-06 already marks it [S]. Versions differ (an earlier talk gives 14.1% / 22.0%). |
| 10 | ProjectEval: GPT-4o ~15% Pass@5 on complex projects | PLAUSIBLE-UNCHECKED | Two alphaXiv pages conflict: 15% ("complex") vs 12.49% (overall). Pass@5 is the best of 5 and the % of test cases, not comparable with Web-Bench Pass@1. Tagged [V]/[S]; it should be [S]. |
| 11 | Ambig-SWE exists (ICLR 2026, CMU); models cannot detect underspecification and rarely ask; interaction up to 74% better | SUPPORTED | mlanthology ICLR 2026 entry, arXiv v3 abstract, author repo README. "74%" is abstract wording; relative vs absolute is unconfirmed. The blog's 37.94 vs 59.52 is [S]. |
| 12 | ClarifyGPT (FSE 2024): ask only when a consistency check detects ambiguity; GPT-4 70.96 → 80.80 MBPP-S; averages 62.43 → 69.60 | Existence SUPPORTED; numbers PLAUSIBLE-UNCHECKED | FSE 2024 profile confirms the venue (different title from the arXiv one). Search shows a *different* GPT-4 average of 68.02 → 75.75, and no source shows 70.96/80.80. W2-06 notes that versions differ; the [V] tag is too strong for the specific numbers. |
| 13 | LLMREI (RE 2025): 33 simulated interviews, error count comparable to humans | SUPPORTED | RE 2025 researchr page and arXiv 2507.02564 abstract. |
| 14 | ReqElicitGym: fewer than half of implicit requirements uncovered across 7 LLMs | Existence SUPPORTED; "less than half" PLAUSIBLE-UNCHECKED | arXiv 2602.18306 (101 scenarios, oracle user). Also found: style requirements elicited at below 0.01 (W2-06 omits this; it is relevant to UX acceptance). |
| 15 | "Nothing found showing that accept/reject feedback from earlier rounds improves later autonomous rounds" | PARTLY CONTRADICTED (weakly) | arXiv 2607.13091 (July 2026) turns accepted review comments into persistent rules, giving 0% recurrence of ruled-against error classes over 11 sessions (single deployment, authors are the deployers). arXiv 2608.10319 studies personalized skills from interaction histories (results not retrieved). Counter-evidence: memory or over-personalisation can hurt (OP-Bench, secondary). The correct statement is "weak, single-deployment evidence only", not "nothing". |

**Generation and metric mismatches to flag:**
- Every build-benchmark and selection number is from Claude 3.5/3.7 Sonnet, GPT-4/4o, o1, or open 7-32B models. None is from the Claude 5.x family that BUILD_PLAN uses. Stroebl's own finding says false-positive rate falls as single-sample accuracy rises, so the optimal N for a strong model may differ in either direction. The "<10" number should not anchor N for Opus/Sonnet 5.5.
- Summary item 5 mixes Pass@1 (Web-Bench, task-level), % of unit tests (Commit0), rubric score (PaperBench) and Pass@5 over test cases (ProjectEval). Only "no agent fully reproduced a library" (Commit0) and Web-Bench 25.1% are build-level failure rates.
- The SWE-HERO, R2E-Gym, Satori and CodeMonkeys gaps are measured on single-issue bug fixing with hidden ground-truth tests. That is not compare goals and not greenfield builds.

## Tag integrity

1. **[V] is defined honestly but named misleadingly.** Section 0 defines [V] as "returned from a primary-source listing, mediated by the search tool". The search tool returns a model-written summary, not the abstract text, so it can blend sources. [V] reads as "verified"; it should read as "abstract-level, via search summary" (for example [A]). A reader skimming the defaults table will take [V] as checked.
2. **[V] applied to secondary-only numbers:**
   - Commit0 17/6/26 (Moonlight/alphaXiv).
   - ProjectEval 15% (alphaXiv overview; conflicts with 12.49%).
   - Web-Bench "65.4% SWE-bench Verified" comparison (not in the README).
   - ClarifyGPT specific numbers (not re-found; another version differs).
   - Summary item 5's blanket [V].
3. **Absence of evidence was used as a correction.** Correction #1 (SWE-HERO) downgraded a correct wave-one claim because the researcher's searches "returned nothing". A "not confirmed" result should never be written as "replace the figures".
4. **The W2-06 [S] figures for SWE-HERO are themselves wrong:**
   - "Up to about 15% at K=16, widens with K" does not match a K=32 table showing gaps of 15-23.
   - The 7.9/5.2/4.5 gains could not be traced.
5. **Guesses are labelled well.** N=3-4, halving, tournament shape, 3x reruns, 5 questions and 3 rejection rounds are all marked **Guess**. Walking-skeleton and separate-test-author are marked [U].

## Overreach findings

1. **"Hybrid verifier: required, evidence-backed".** Overreach.
   - In R2E-Gym the hybrid combines a *trained* execution-free verifier with agent-written tests. In Satori it is a trained reward model plus unit tests. The factory would use a prompted Claude judge, which is not shown to transfer.
   - Downgrade to "default design, measure in Phase 6".
2. **Stroebl as the basis for an N ceiling of 8-10.** Mild overreach. The ceiling is a function of an assumed false-positive cost, and the paper's models are weak. Keep it as a ceiling with a measurement, not as a sourced number.
3. **Internal inconsistency in the defaults.**
   - §4.2 says "default 3, max 4-5", while the §5 table says "3, max 4".
   - Successive halving at N=3-4 with "2 checkpoints, halve each" leaves 1 candidate after the first checkpoint, so the second checkpoint is vacuous.
   - "Knockout for N>4" can never apply when max N is 4.
   - These defaults do not fit together.
4. **Evolutionary mode.** The scoping is correct (numeric fitness only), but cost is not addressed. ShinkaEvolve's ~150 samples and AlphaEvolve's thousands are each full agent runs on a home server. It belongs in "optional later", not as a Phase 6 item.
5. **Ambiguity detection.** The evidence supports the need (Ambig-SWE, ClarifyGPT, ClarifyCodeBench). It does not support the *method*: ClarifyGPT's code-consistency check is function-level, and its transfer to a conversational product vision is untested. Label the method as design.
6. **Traceability drift metric.** It is consistent with doc 06 item 8 and BUILD_PLAN's "traceability gate", so it adds measurement rather than new design. Risk not mentioned: requirement-ID citations are written by the builder, so the metric can be gamed. Panda (cited) shows forced citations change behaviour. The verifier, not the builder, must judge traceability.
7. **"Fail most of the time".** This is supported for 2024-25 models only. The doc says so, and its recommendation to use the numbers as a reason for skeleton-first rather than as a forecast is appropriately modest.

## Consistency with BUILD_PLAN.md and other docs

- BUILD_PLAN's Phase 6 exit ("known-better option wins reliably") is compatible with the selector-accuracy edit, which sharpens it.
- BUILD_PLAN's verifier ordering ("deterministic first, LLM judges last") must be reconciled with "hybrid" scoring. The hybrid should rank gate survivors only and never override a gate (doc 06 A4, item 4).
- W2-05 §88 recommends "best-of-N with an execution-based selector" for solve goals. W2-06 says execution-only selection plateaus. The two need one shared selector design, at minimum a cross-reference.
- Doc 06's correction list is otherwise accurate (Satori inference, R2E-Gym compute-efficiency sentence, A5 merging, B3 elicitation, halving as design).

## Missing risks and alternatives

- **Correlated candidates.** An all-Claude fan-out produces correlated errors. Zhu et al. (cited) show mixed-model rollouts beat single-model ones. Diversity of prompts, plans and models deserves a default, especially given open decision 6.
- **The planted-set test can be too easy.** Selector accuracy should be measured on near-tie pairs and on pairs where the worse option is longer or more polished, not just on obviously-better options.
- **Elicitation cost.** Over-asking burdens the user (ClarifyCodeBench: more questions are not better questions). Measure user question load alongside the question count.
- **Style and UX requirements.** ReqElicitGym shows LLM interviewers almost never elicit aesthetic or style requirements (below 0.01). The elicitation template should force a UX/style probe, or use the preview checkpoint that doc 06 lists as an open question.
- **DoD-lessons risk.** Accumulated lessons can over-personalise or entrench a one-off complaint. They need an expiry or review step, plus the instrumentation W2-06 proposes.
- **Contamination and age.** The benchmarks are public and from 2024-25, so 5.x-family results on them may be inflated by training-data overlap. The factory's own Phase 7 runs are the only valid calibration.

## Recommended edits

### To doc 06

| Proposed edit (W2-06 §3/§4.1) | Decision | Reason |
|---|---|---|
| Correction 1: replace the SWE-HERO 64.6/79.8 figures | **REJECT** | Contradicted. Keep the wave-one figures; add "32B rollouts, SWE-Lego-8B verifier (best of 3 verifiers), K=32, from a search extract of Table 3; re-check against the PDF". Drop the 7.9/5.2/4.5 and "K=16" figures. |
| Corrections 2-4 (Satori caveat, R2E-Gym sentence, CWM 40 tests) | ACCEPT | Re-confirmed. Add 41.6 vs 41.0 and the Agentless Mini baseline scaffold. |
| Correction 5: A5 merging (merge only with numeric fitness) | ACCEPT | Correct narrowing. Cite the crossover numbers as [S]. |
| Correction 6: B3 elicitation rewrite (Ambig-SWE, ClarifyGPT, ReqElicitGym) | ACCEPT | Existence and venues re-confirmed. Mark ClarifyGPT numbers as version-dependent. Add the ReqElicitGym style-requirement gap. |
| A1: add Stroebl and "cap N where false positives dominate" | MODIFY | State the scope (weak models, HumanEval/MBPP, assumed false-positive cost). Phrase it as "measure the false-positive rate; do not raise N past where it dominates", not as a numeric cap. |
| Evolutionary mode subsection | MODIFY | Keep as "optional later, numeric fitness only, budget-capped". Do not add it to Phase 6 scope. |
| Benchmarks calibration paragraph | MODIFY | List the metric per benchmark (Pass@1 task; % of unit tests; rubric score; Pass@5 of test cases) and the model generation. Do not use the single phrase "fail most of the time". |
| Relabel "max ~5 questions" and "max 3 rounds" as guesses | ACCEPT | Unsourced. |
| "DoD lessons" loop as an untested hypothesis | MODIFY | Say "weak single-deployment evidence (arXiv 2607.13091), plus a risk of over-personalisation"; keep the instrumentation. |

### To BUILD_PLAN.md

| Proposed edit (W2-06 §4.2) | Decision | Reason |
|---|---|---|
| Phase 6 exit: selector accuracy on a planted set and gate false-positive rate | ACCEPT, with an addition | Directly supported (Stroebl; SWE-HERO and R2E-Gym gaps). Include near-tie and polish-bias pairs. |
| Phase 6: hybrid verification "required" | MODIFY | Make it the default for ranking gate survivors, behind a measurement. The evidence uses trained verifiers, not prompted judges. Reconcile with W2-05's execution-only selector. |
| Phase 6: Copeland aggregation as a candidate design | ACCEPT as a candidate | Labelled [S]; cheap at N≤4 (≤6 pairs, ×2 for order swaps). |
| Phase 6: evolutionary mode | MODIFY | Move to "Optional later". |
| Phase 7: ambiguity-detection step before elicitation | ACCEPT | Strongly supported need. Label the detection method as design. |
| Phase 7: traceability drift metric (fraction of hunks traced to a requirement ID) | ACCEPT, with a guard | The verifier, not the builder, must assess traceability. Self-cited IDs can be gamed. |
| Phase 7: record first-pass acceptance rate | ACCEPT | Needed to evaluate the rejection loop. |
| §4 item 7: failure rates now sourced; Spec Kit/Kiro/BMAD have no evidence of benefit | MODIFY | Say "sourced for 2024-25 models at abstract level" and keep "re-check against full text" open. The no-evidence statement for SDD tools is fine. |
| Open decision 3: default N=3, max 4-5 (guess) | MODIFY | Use one value (3, max 4) to match the §5 table and doc 06's `max_candidates: 4`. Drop successive halving at N≤4, or define it as a single cut. Drop the "knockout for N>4" rule. Keep it marked as a guess. |
