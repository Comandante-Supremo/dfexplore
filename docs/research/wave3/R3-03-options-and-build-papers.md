# R3-03: Options and build papers, read from the PDFs

Date: 2026-10-07. Method: PDFs fetched from `https://arxiv.org/pdf/<id>`, text extracted with pdftotext, relevant sections read. Quotes are verbatim from the extracted text. Search was used only to locate ids. Blocked and not read: Nature (FunSearch), OpenReview, ACM DL, IEEE Xplore (all returned a proxy 403 or no connection; not routed around). Venue claims that live only on those sites are marked "venue not verifiable here". All downloaded content was treated as data.

## Summary table

| # | Claim | Paper (arXiv id, version) | Verdict | One-line result |
|---|---|---|---|---|
| 1a | SWE-HERO Best@32 64.6 / Pass@32 79.8 | 2604.01496v2 (6 May 2026) | CONFIRMED-WITH-SCOPE-CAVEAT | Table 3, SWE-Hero-32B + SWE-Lego-Verifier-8B (best of 4 verifiers; 64.0 to 63.8 for the others). SWE-bench Verified, OpenHands. Pass@1 is the mean of 32 rollouts = 60.1. |
| 1b | Gains 7.9 / 5.2 / 4.5 | same | CONFIRMED | They are Best@32 minus mean Pass@1, per model size, using the best verifier. The same table contains both sets of numbers; they do not conflict. |
| 1c | "~15% gap at K=16" | same | CONFIRMED (text only) | Text: "reaches as much as 15% at K = 16 and continues to widen". At K=32 the table gives 18.1 / 15.8 / 15.2 points (7B / 14B / 32B). |
| 2 | CodeMonkeys 57.4% single, 66.2% ensemble | 2501.14723v2 (3 Feb 2025) | CONFIRMED-WITH-SCOPE-CAVEAT | Claude 3.5 Sonnet, about $2,300 (Table 1: $2,291.90). 66.2% is selection over 5 candidates per issue, 4 of them other teams' submissions. |
| 3 | CWM 53.9 to 65.8 | 2510.02387v1 (30 Sep 2025) | CONFIRMED | pass@1 averaged over 4 runs. 65.8 is "best@k" with k=16 plus 40 generated unit tests. "40 unit tests" IS in the paper. |
| 4 | Satori Best@50 41.6 vs Best@500 41.0 | 2505.23604v1 (29 May 2025) | CONFIRMED-WITH-SCOPE-CAVEAT | Different models (Qwen2.5-Coder-32B, RL-trained for self-evolution, vs Llama3-SWE-RL-70B), different scaffolds. The "10x" is the authors' own wording and compares sample counts only. |
| 5 | R2E-Gym hybrid 51% vs 42-43% | 2504.07164v1 (9 Apr 2025) | CONFIRMED | Verifiers alone 43.7 / 42.8; hybrid Best@26 = 51.0 (Best@16 = 49.4). Trained 32B Qwen2.5-Coder agent on SWE-bench Verified. |
| 6 | Stroebl: optimal attempts often under 10 | 2411.17501v3 (26 Mar 2026) | CONFIRMED-WITH-SCOPE-CAVEAT | HumanEval+/MBPP+, Llama-3.1 / Code Llama / Command / GPT-4o. The "under 10" depends on an assumed cost-benefit ratio (0, 1, 2, 4, 8). ICLR 2026 not stated in the PDF. |
| 7a | Web-Bench 25.1% Pass@1 | 2505.07473v1 (12 May 2025) | CONFIRMED | Claude 3.7 Sonnet (thinking) with their Web-Agent; 65.4% SWE-bench Verified comparison IS in the abstract. |
| 7b | Commit0 about 6% of tests | 2412.01769v1 (2 Dec 2024) | CONFIRMED-WITH-SCOPE-CAVEAT | Table 3: Claude 3.5 Sonnet, stage 1 only, Commit0 all = 6.12%. The 17% is stage 1 on "lite". Abstract says 26% with feedback; Table 1 says 29.30%. |
| 7c | PaperBench 21.0% vs human | 2504.01848v3 (7 Apr 2025) | CONFIRMED-WITH-SCOPE-CAVEAT | Table 4, BasicAgent, Claude 3.5 Sonnet (New). Human 41.4% is best-of-3, on a 3-paper subset, 48 h; o1 got 26.6% on that subset. Not the same set as the 21.0. |
| 7d | ProjectEval about 15% Pass@5 | 2503.07010v2 (31 May 2025) | CONFIRMED-WITH-SCOPE-CAVEAT | GPT-4o. Text: "only GPT-4o reach the Pass@5 of 15%". Table 7 all-setting average is 12.49. Metric = % of test cases passed. |
| 8a | Ambig-SWE (ICLR 2026) | 2502.13069v3 (21 Feb 2026) | CONFIRMED | PDF header says "Accepted at ICLR 2026". Up to 74% (relative) from interaction; detection accuracy 0.47 to 0.89 depending on model and prompt. |
| 8b | ClarifyGPT (FSE 2024) | 2310.10996v1 (17 Oct 2023) | CONFIRMED (abstract); one earlier number NOT-IN-PAPER | 70.96 to 80.80 (human feedback, MBPP-sanitized) and GPT-4 average 68.02 to 75.75 are in the paper. The "62.43 to 69.60" figure is not in the arXiv text. FSE venue not verifiable here. |
| 8c | LLMREI (RE 2025) | 2507.02564v1 (3 Jul 2025) | CONFIRMED | 33 simulated interviews, GPT-4o. "similar number of errors" to human interviewers; up to 73.7% of requirements elicited (60.94 full + 12.76 partial). |
| 8d | ReqElicitGym; "rarely ask about style/UX" | 2602.18306v1 (20 Feb 2026) | CONFIRMED, and stronger than earlier docs | 7 current LLMs incl. GPT-5.2 and Claude Opus 4.5. Best overall IRE is 0.32 (DeepSeek V3.2); Claude Opus 4.5 is 0.08. Style IRE below 0.01 for almost all. |
| 9 | 2607.13091 feedback loop | 2607.13091v1 (13 Jul 2026) | Does NOT show the claim as stated | Human-in-the-loop PR review turning accepted comments into rules; 9 error classes, 74 exposures, 0 recurrences; no control; no model named; not autonomous rounds. |
| 10 | Evolutionary / crossover | 2506.13131v1; 2509.19349; 2605.19633; 2604.02134 | CONFIRMED-WITH-SCOPE-CAVEAT | Every positive result has a machine-gradeable evaluator. Crossover ablation: -2.1 pt Pass@1 (EvolRepair). FunSearch: UNREACHABLE (Nature). |
| 11 | SDD empirical studies | 2606.30689, 2608.25202, 2606.04967, 2601.03878, 2602.00180 | Partly found | Not "none found". No outcome study of Kiro or BMAD. One controlled study of Spec Kit (determinism and hallucination detection, not quality). No measurement of spec drift. |
| 12 | Correlated errors / best-of-N, frontier | 2607.05391, 2506.07962, 2603.06612, 2607.01597 | Found | Frontier best-of-N gains are small (+2 to +3.4 pt) against large oracle headroom. Correlation evidence is general, not code-specific. |

---

## Claim 1. SWE-HERO (settled)

PAPER: "From SWE-Zero to SWE-Hero: Execution-free to Execution-based Fine-tuning for Software Engineering Agents", Ludwig, Ahmad, Majumdar, Ginsburg (NVIDIA). arXiv 2604.01496v2, 6 May 2026, cs.SE, "Preprint". No venue.

BENCHMARK / METRIC / MODEL: SWE-bench Verified, resolve rate. OpenHands scaffold. SWE-Hero-7B/14B/32B are SFT-distilled students of Qwen3-Coder-480B-A35B-Instruct (2025 generation). Headline Pass@1: 52.7 / 60.8 / 62.2 (text line "Our 7B, 14B, and 32B models achieve resolution rates of 52.7%, 60.8%, and 62.2%"). Pass@1 in Table 3 is "avg of 32" rollouts, so it is 60.1 for the 32B, not 62.2.

EARLIER DOCS: Doc 06 and wave one: "Best@32 of 64.6 vs Pass@32 of 79.8". W2-06 "correction": figures not confirmed, replace with 7.9/5.2/4.5 gains and a "~15% gap at K=16". The A-06 audit rejected the correction. BUILD_PLAN line 149 keeps 64.6/79.8.

PAPER SAYS (Appendix A.2, Table 3, page 17). Setup: "For each problem instance, we generate K = 32 candidate rollouts and select the single highest-scoring trajectory. Performance is measured by Best@K, defined as the fraction of instances successfully resolved under this top-1 selection". Verifiers are generative, yes/no token probability.

| Model | Verifier | Pass@1 (avg of 32) | Pass@32 | Best@32 |
|---|---|---|---|---|
| SWE-Hero-7B | OpenHands-32B-Verifier | 50.0 | 76.0 | 52.9 |
| | R2EGym-Verifier | | | 55.9 |
| | SWE-Lego-Verifier-8B | | | 57.9 |
| | SWE-Lego-Verifier-30B-A3B | | | 57.9 |
| SWE-Hero-14B | OpenHands-32B-Verifier | 57.4 | 78.4 | 62.0 |
| | R2EGym-Verifier | | | 62.6 |
| | SWE-Lego-Verifier-8B | | | 60.6 |
| | SWE-Lego-Verifier-30B-A3B | | | 61.0 |
| SWE-Hero-32B | OpenHands-32B-Verifier | 60.1 | 79.8 | 63.8 |
| | R2EGym-Verifier | | | 63.8 |
| | SWE-Lego-Verifier-8B | | | **64.6** |
| | SWE-Lego-Verifier-30B-A3B | | | 64.0 |

(Row grouping inferred from pdftotext layout: each model's Pass@1/Pass@32 sits beside its first or last verifier row, and the Best@32 values 64.6 and 64.0 follow the "SWE-Hero-32B" label line. The 64.6 row is SWE-Lego-Verifier-8B, as the audit said.)

Gains quote: "the resolution rates for the 7B and 14B models increased by 7.9% and 5.2%, respectively, while the 32B model saw a robust gain of 4.5%." Arithmetic check: 57.9 - 50.0 = 7.9; 62.6 - 57.4 = 5.2; 64.6 - 60.1 = 4.5. So the gains are percentage points of Best@32 over mean Pass@1, each using the best verifier for that size.

Gap quote: "a significant gap persists between Best@K ... and Pass@K ... Across the SWE-Hero suite, this margin reaches as much as 15% at K = 16 and continues to widen as K increases." Gap at K=32 from Table 3 (best verifier): 7B 76.0 - 57.9 = 18.1; 14B 78.4 - 62.6 = 15.8; 32B 79.8 - 64.6 = 15.2. The "15% at K=16" is the authors' text for Figure 5 (the plotted values could not be cleanly extracted, so K=16 per-model numbers are not independently checked). Note the text says "as much as 15% at K=16", yet Table 3 shows larger gaps at K=32 for the 7B and 14B, consistent with "continues to widen".

VERDICT: 64.6 and 79.8 CONFIRMED (Table 3, 32B, SWE-Lego-Verifier-8B). 7.9/5.2/4.5 CONFIRMED. "~15% at K=16" CONFIRMED as a text statement. The W2-06 "correction" was wrong to drop 64.6/79.8; the W2-06 numbers are also real but describe a different quantity (gain over mean Pass@1). The audit was right.

SCOPE CAVEATS: (a) Best@32 uses the best of four verifiers per size, picked on the test set (selection of the verifier on the evaluation data). (b) Pass@1 here is mean-of-32, not the headline 62.2. (c) The "verifier-to-oracle gap" is Pass@32 minus Best@32, an upper bound, with fixed-model rollouts from one SFT model (all K candidates from the same model). (d) Open-weight verifiers, not frontier models. (e) Single-issue bug fixing with hidden tests.

---

## Claim 2. CodeMonkeys

PAPER: "CodeMonkeys: Scaling Test-Time Compute for Software Engineering", Ehrlich, Brown, Juravsky, Clark, Re, Mirhoseini. arXiv 2501.14723v2, 3 Feb 2025 (dated 24 Jan 2025 in text). Stanford/Oxford. No venue in the PDF.

BENCHMARK / METRIC / MODEL: SWE-bench Verified, resolve rate. Primary model Claude 3.5 Sonnet (API) for ranking, edits, tests and selection; Qwen2.5-Coder-32B-Instruct run locally for file relevance scanning.

EARLIER DOCS: 57.4% single; 66.2% ensemble; about $2,300 (Doc 06 A1; W2-06 notes a 57.7 vs 57.4 discrepancy on the project page).

PAPER SAYS: Abstract: "CodeMonkeys resolves 57.4% of issues from SWE-bench Verified using a budget of approximately 2300 USD ... Selecting over an ensemble of edits from existing top SWE-bench Verified submissions obtains a score of 66.2% and outperforms the best member of the ensemble on its own."
- Cost: Table 1 total $2,291.90 = relevance $334.02 (14.6%, local Qwen), ranking $19.92, test generation $439.99 (19.2%), edit generation $1,366.02 (59.6%), selection $131.95 (5.8%).
- Setup: 10 parallel state machines per issue, each producing an (edit, test) pair, up to 8 iterations. Coverage (oracle) = 69.8%. Random selection expected 45.8%.
- Selection method (Section 2.3): run all 10 model-generated tests on all 10 edits, keep the top-3 by tests passed (ties broken by shorter diff), then a multi-turn "selection state machine" in which the model may write new tests to separate the 3 candidates. "CodeMonkeys achieves a final score of 57.4% ... closing approximately half of the gap between random and oracle selection." Majority voting alone and single-turn model selection alone scored lower (Figure 6; "model selection without top-3 filtering underperforms pure majority voting").
- 66.2% ("Barrel of Monkeys", Section 2.3.1): the pool is CodeMonkeys' final edit plus the top four leaderboard submissions as of 15 Jan 2025 (Blackbox AI Agent, CodeStory Midwit + swe-search, Learn-by-interact, devlo), so 5 candidates per issue, coverage 80.8%. Selection: same state machine, no test-based pre-filter. Best member alone 62.8%; random from the pool 60.9%. "our selection method recovers a smaller proportion of the gap between random selection and coverage here".

VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. 66.2% is not a CodeMonkeys-alone score; it needs four other teams' finished systems as inputs. Gain over the best member is +3.4 pt; over random draw from the pool +5.3. Claude 3.5 Sonnet is a 2024 generation model.

---

## Claim 3. CWM

PAPER: "CWM: An Open-Weights LLM for Research on Code Generation with World Models", Meta FAIR CodeGen Team. arXiv 2510.02387v1, 30 Sep 2025. Technical report.

BENCHMARK / METRIC / MODEL: SWE-bench Verified, pass@1 resolve rate, averaged over 4 runs. CWM, 32B dense open weights, 128-turn harness (Table: "Ours 128 turns 53.9").

PAPER SAYS (Section 7.2): "CWM achieves pass@1 resolve rates of 65.8 % with test-time-scaling and 53.9 % without test-time scaling (averaged over 4 runs)."
TTS method, verbatim: "we first generate k candidate solutions as well as 40 novel unit tests in parallel agentic loops for each instance ... we keep the top-5 majority tests ... keeping only those patches that pass the highest number of existing tests. We then execute the remaining patches on the filtered set of novel tests and select the patch with the highest pass rate ... In case of ties, we prioritize the majority patch, and if the tie remains, we choose the patch whose trajectory has fewer tokens. We refer to this approach as best@k."
"In Figure 16, we report results for best@k for k = 16, which achieves a 65.8 % resolve rate." Majority voting with no test generation: 58.4%. pass@k reaches 80.4% at k = 40. "For best@k, performance improves sharply from k = 2 before plateauing around k = 16. For majority-voting, performance improves gradually from k = 2 and plateaus at k = 24."

VERDICT: CONFIRMED. The "k candidates and 40 generated unit tests" detail that W2-06 could not confirm IS in the paper. Metric note: the paper labels the 65.8 "pass@1 ... with test-time scaling" (single submitted patch per instance chosen by best@16), so it is a selected-patch rate, not oracle pass@k. Useful extra: the gap between best@16 (65.8) and pass@40 (80.4) is again about 15 points, and test-free majority voting gets 58.4, so most of the 11.9 point gain (53.9 to 65.8) needs generated tests.

---

## Claim 4. Satori-SWE and the "10x"

PAPER: "Satori-SWE: Evolutionary Test-Time Scaling for Sample-Efficient Software Engineering", Zeng, Shen, et al. (SUTD, MIT, UMass, Harvard, MIT-IBM). arXiv 2505.23604v1, 29 May 2025. No venue in the PDF.

BENCHMARK / METRIC / MODEL: SWE-bench Verified, resolve rate. Satori-SWE-32B, base Qwen2.5-Coder-32B-Instruct, RL-trained to self-improve across iterations; patch-generation pipeline (not a multi-turn agent) with the authors' own retrieval.

PAPER SAYS (Table 1 and Section 5.3):
- Satori-SWE-32B: Greedy 35.8, Best@10 38.9, Best@25 40.2, Best@50 41.6.
- Llama3-SWE-RL-70B (scaffold Agentless Mini): Best@80 37.0, Best@160 40.0, Best@500 41.0.
- Text: "it achieves a Best@50 score of 41.6, matching the performance of the current state-of-the-art Llama3-SWE-RL-70B, which requires Best@500 decoding--incurring over 10x higher sampling cost."
- Selection for Satori Best@N: "we combine both the reward model and unit tests to select the best patch from the candidate pool"; 50 candidates = 25 per iteration, aggregated across iterations. For N=50 the candidates are not i.i.d.; they are produced by iterated self-refinement.

VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. 41.6 vs 41.0 is exact. The comparison is between two different trained models (32B vs 70B) with different scaffolds and different selectors, and the Satori model was RL-trained specifically to improve its own outputs across iterations. So "10x" is a sample-count ratio, not a measured efficiency gain of a scheduling or selection method. It cannot support "smarter allocation beats raw N" for an untrained model. The SWE-RL figure also reflects Agentless samples (cheap patches) while the Satori samples are also single patches, so both are far cheaper than agent rollouts; neither number transfers to agentic rollouts. 2025 generation models.

---

## Claim 5. R2E-Gym hybrid verifier

PAPER: "R2E-Gym: Procedural Environments and Hybrid Verifiers for Scaling Open-Weights SWE Agents", Jain, Singh, Shetty, Zheng, Sen, Stoica. arXiv 2504.07164v1, 9 Apr 2025, "Preprint. Under review". (NeurIPS 2025 listing per earlier waves; not verifiable here.)

BENCHMARK / METRIC / MODEL: SWE-bench Verified. Editing agent R2E-Gym-32B (Qwen-2.5-Coder-32B base, SFT on agent trajectories). Pass@1 34.4% (+-1.2). Best@K with a verifier.

PAPER SAYS: Abstract: "while each approach individually saturates around 42-43%, significantly higher gains can be obtained by leveraging their complementary strengths. Overall, our approach achieves 51% on the SWE-Bench Verified benchmark".
- Section 4.1: "Best@K rate quickly plateaus for both methods, converging similarly to 43.7% and 42.8% respectively." (43.7 execution-based, 42.8 execution-free; text lists them in that order after the sentence naming "both execution-based and execution-free"; mapping of which is which follows that order.)
- Table 4: R2E-Gym Pass@1 34.4; Best@16 w/ Hybrid 49.4; Best@26 w/ Hybrid 51.0. Claude 3.7 Sonnet + tools 62.3 for reference.
- Hybrid (Section 4.3): score = Top-n(execution-free score) + execution-based score; the continuous execution-free score breaks ties among patches with equal test pass counts. "yielding significant performance improvements (additional 7-8%)".
- Both verifiers are trained by the authors: a testing-agent that writes reproduction tests (plus regression-test filter) and an execution-free verifier trained on trajectories+patches. Ablations: "Regression tests alone are insufficient"; removing the trajectory from the execution-free input drops Best@26 from 42.8 to 37.6; agent-written tests beat Agentless-released tests (51.0 vs 48.8); more test-agent rollouts help.

VERDICT: CONFIRMED. Scope: the hybrid is a trained verifier plus agent-written tests with 26 editing rollouts. The gain is real but not shown for a prompted LLM judge. The wave-one sentence "more test-agent rollouts can be more compute-efficient than more editing rollouts" is only loosely supported (an ablation shows more test-agent rollouts "consistently helps"); no compute-efficiency comparison was seen.

---

## Claim 6. Stroebl, Kapoor, Narayanan

PAPER: "The Limits of Inference Scaling Through Resampling". arXiv 2411.17501v3, 26 Mar 2026 (v1 Nov 2024). Princeton. The PDF text does not state a venue; ICLR 2026 comes from OpenReview, which is blocked (venue not verifiable here).

BENCHMARK / METRIC / MODEL: HumanEval+ / MBPP+ (extended tests as ground truth) with the original HumanEval/MBPP tests as the imperfect verifier. Models: Llama-3.1 family, Code Llama, Cohere Command (incl. Command Light), GPT-4o. 200 samples per task on HumanEval+ (1,000 for Command Light), 50 on MBPP+.

PAPER SAYS: Abstract: "Empirical results show that optimal sampling attempts are often fewer than 10, as the negative utility of false positives outweighs benefits, bending inference scaling curves downward." Section 4 (HumanEval, 200 samples, 1,000 permutations): reward = +1 for a true positive, minus c for a false positive, with cost-benefit ratio c in {0, 1, 2, 4, 8}. Finding: "at a cost-benefit ratio of 4, the optimal number of samples is K <= 5 for all four models" (Figure 4, Llama-3.1 and Code Llama; GPT-4o in Figure 15). "If the ratio is high enough, the optimal number of samples is zero." Introduction: "even with an infinite inference budget, the optimal number of samples is often finite and very low (e.g., K <= 5 in Figure 4)." Caveat in the paper: "for some models such as Llama-3.1-70B, the false positive rate increases dramatically with K, whereas for others such as the Code Llama and Command families, the increase is much more gradual, resulting in much higher values of the optimal K, especially for low cost-benefit ratios." Also: weaker models have higher false-positive probability, "scales inversely with the true capability". The false-positive effect comes from a bimodal task difficulty (easy tasks solved early; hard tasks produce false positives).

VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. "<10" holds for a stated range of assumed false-positive costs (the K <= 5 example is at ratio 4); at ratio 0 the optimum is not capped; for some models it is much higher. The evidence is function-level, with weak/older open models, test suites with limited coverage, and no agentic or SWE-bench experiments. The paper does not test frontier agents, and its own finding (false-positive rate falls as single-sample accuracy rises) suggests the cap moves with model strength. Use as "measure your false-positive rate; do not assume a numeric N cap".

---

## Claim 7. Greenfield build benchmarks

All numbers below are for 2024 to mid-2025 models unless stated; then newer 2026 results.

**7a Web-Bench.** arXiv 2505.07473v1, 12 May 2025, ByteDance. 50 projects x 20 sequentially dependent tasks, "On average, a single project takes 4 to 8 hours for a senior engineer". Abstract: "On our given benchmark agent (Web-Agent), SOTA (Claude 3.7 Sonnet) achieves only 25.1% Pass@1, significantly lower (better) than SWE-Bench's Verified (65.4%) and Full (33.8%) scores." Table 3 dates it 2025.4. Results text: "'claude-3-7-sonnet-20250219-thinking' achieves the highest Pass@1 and Pass@2 (using Best-of-5 sampling, as used throughout)". Average Pass@1: closed models 15.08%, open 10.73%; average Pass@2: closed 20.79%, open 14.84%. Metric: Pass@1 on task completion with sequential tasks; Pass@2 uses best-of-5 sampling. VERDICT: CONFIRMED. The 65.4% comparison is in the paper (A-06 could not see it in the README). Caveat: "Pass@1" is on the project's tests and a bespoke agent; only Claude-series Sonnet 3.7 generation.

**7b Commit0.** arXiv 2412.01769v1, 2 Dec 2024, Cornell/Cohere. Abstract: "with a state-of-the-art LLM without feedback, it can pass 17% unit tests in the easier libraries but can only pass 6% in all libraries ... iterating on error messages from unit tests improves the pass rate of unit tests to 26% on the easier libraries". Tables: Commit0 lite (16 libraries) Table 1: Claude 3.5 Sonnet stage 1 = 17.80, stage 2 = 18.79, stage 3 = 29.30 (so the abstract's 26% disagrees with Table 1's 29.30); o1-preview 17.34 / - / 21.46. Commit0 all (Table 3): Claude 3.5 Sonnet stage 1 only = 6.12 (Claude only run for stage 1 because of cost); GPT-4o-mini 2.87 / 1.42 / 4.24. Agent: authors' SDE-I using the Aider framework. Metric: percent of unit tests passed. "none can yet fully reproduce full libraries". VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT: ~6% is Claude 3.5 Sonnet, first stage only, no feedback, percent of tests. Not a task-success rate.

**7c PaperBench.** arXiv 2504.01848v3, 7 Apr 2025, OpenAI (ICML 2025 per earlier waves; not stated in the PDF). Metric: Replication Score, a weighted rubric score over 8,316 leaves judged by an LLM. Table 4 (BasicAgent): Claude 3.5 Sonnet 21.0 +-0.8, o1-high 13.2, DeepSeek-R1 6.0, GPT-4o 4.1, Gemini-2.0-Flash 3.2, o3-mini-high 2.6. Table 5 (IterativeAgent): o1-high 24.4 +-0.7, Claude 3.5 Sonnet 16.1, o3-mini-high 8.5; o1-high 36-hour run 26.0 +-0.3. So the abstract's "best-performing tested agent ... 21.0%" refers to the BasicAgent table; the IterativeAgent o1 run is higher at 24.4 and 26.0. Human baseline (intro): "On a 3-paper subset, our human baseline of ML PhDs (best of 3 attempts) achieved 41.4% after 48 hours of effort, compared to 26.6% achieved by o1 on the same subset." (Section 5.4 says a subset of 4 papers with best@3; the paper is internally inconsistent about 3 vs 4.) VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. Do not state "21.0% vs 41.4%" as a like-for-like pair: different paper sets; the fair pair is 41.4 vs 26.6, and the humans are best-of-3. o1 led humans in the first hours and plateaued after hour 1; humans overtook after 24 h.

**7d ProjectEval.** arXiv 2503.07010v2, 31 May 2025, Findings of ACL 2025 per earlier waves (not in the PDF). Metric: percent of test cases passed (average 14.2 test cases per task, 284 total in the paper's counting), Pass@K over generation attempts; three input levels (NL prompt, checklist, skeleton); execution tests that simulate user interaction. Text: "ProjectEval is hard for nowadays agents as only GPT-4o reach the Pass@5 of 15%." Table 7 GPT-4o Pass@5 columns: 16.06, 15.42, 10.14 (cascade and direct level variants), direct avg 13.87, all-avg 12.49; Pass@1 values 8.52 to 19.72 depending on setting. Gemini 1.5 Pro all-avg 6.65, GPT-3.5 2.90. OpenHands with GPT-4o: 7.39 Pass@1 (8 of 20 tasks unfinished). VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. The two conflicting alphaXiv numbers are both in the paper: about 15% is the headline/best settings, 12.49 the all-settings average. Metric is per-test-case pass rate at best of 5, not project-level success. Do not put it next to Web-Bench Pass@1.

**7e Others quickly found.**
- DevBench: arXiv 2403.08604v3 (14 Dec 2024) is now titled "Prompting Large Language Models to Tackle the Full Software Development Lifecycle: A Case Study" and calls the benchmark DevEval (earlier versions: DevBench). GPT-4-Turbo, Table 6: implementation Pass@ acceptance tests 7.1%, unit tests 8.0%, environment setup 41.7%, acceptance-test generation (oracle) 29.2%, unit-test generation 36.5%, coverage 33.2 (66.3 valid). Text: GPT-4-Turbo "registers only a 7.1% pass rate on reference acceptance tests". 2023/24 models.
- MetaGPT / SoftwareDev: arXiv 2308.00352v7 (1 Nov 2024), ICLR 2024. Table 1 on 70 tasks: Executability on a 0 to 4 scale: ChatDev 2.25, MetaGPT w/o feedback 3.67, MetaGPT 3.75; human revision cost 2.5 / 2.25 / 0.83. Authors' own human-rated scale; GPT-4 era. ChatDev: arXiv 2307.07924 (not read beyond the comparison).
- Independent check of those frameworks, E2EDev: arXiv 2510.14509v4 (16 Apr 2026). BDD-based end-to-end benchmark. Req. Acc. (percent): with Claude-Haiku 4.5 backbone, Vanilla LLM 48.69, GPT-Engineer 53.75, Self-Collab 49.01, MapCoder 49.61, ChatDev 44.73, MetaGPT 5.39. With GPT-4o: ChatDev 42.71, MetaGPT 0.00. Text: models achieve "only 30%-50% Req. Acc." and "even advanced models like Claude-Haiku 4.5 and GPT-4o [fail] to exceed 60%". So multi-agent frameworks did not beat a plain LLM call. (The A-06/W2-06 "0.18-0.45 for ChatDev" is wrong; those columns are cost and CO2.)
- Newer 2026 results with current models:
  - ProgramBench, arXiv 2605.03546v1 (5-6 May 2026, Meta FAIR/Stanford). 200 tasks: rebuild programs (up to FFmpeg, SQLite, PHP) from the executable and docs; behavioural tests from fuzzing. 9 models. Quote: "none fully resolve any task, with the best model passing 95% of tests on only 3% of tasks." Table 2: Claude Opus 4.7 0.0% resolved, 3.0% "almost" (>=95% of tests), $3.81 per task; Opus 4.6 0.0 / 2.5; Sonnet 4.6 0.0 / 1.6; Haiku 4.5, Gemini 3.1 Pro, Gemini 3 Flash, GPT 5.4 (and mini), GPT 5 mini all 0.0 / 0.0 (Gemini 3.1 Pro, GPT-5.4 as listed). Cheating (looking up/decompiling the reference) flagged on 20 to 36% of tasks for the three stronger models.
  - ProjDevBench, arXiv 2602.01655v2 (9 Feb 2026). End-to-end project from spec, 6 agents (Cursor, Claude Code, Augment, Codex CLI, Gemini CLI, Copilot) with GPT-5, Claude Sonnet 4.5, Gemini 3 Pro Preview. Overall acceptance rate 27.38% (484 accepted submissions); final weighted score (0.8 online-judge execution + 0.2 code review) best Codex+GPT-5 77.85, Augment+GPT-5 72.35, Cursor+GPT-5 71.85, Claude Code+Sonnet 4.5 68.87.
  - RepoGenesis, arXiv 2601.13943v3 (15 Apr 2026). 106 microservice repos from README. Best system 23.67% Pass@1 (Python), 21.45% (Java); agents DeepCode, Qwen-Agent, Cursor, Claude Code etc. (model names not extracted).
  - Frontier-model caveat: ProgramBench and ProjDevBench are the only ones I found with Claude 4.5 to 4.7 or GPT-5 to 5.4 era models. Metrics differ (resolved %, acceptance rate, weighted score, Pass@1), so they cannot be merged into one "failure rate".

---

## Claim 8. Elicitation papers

**8a Ambig-SWE.** "Ambig-SWE: Interactive Agents to Overcome Underspecificity in Software Engineering", Vijayvargiya, Zhou, Yerukola, Sap, Neubig (CMU). arXiv 2502.13069v3, 21 Feb 2026; header "Accepted at ICLR 2026". Underspecified variant of SWE-bench Verified (GPT-4o-generated summarized issues). Models: Claude Sonnet 4, Sonnet 3.5, Haiku 3.5, Qwen3 Coder 480B, Llama 3.1 70B, Deepseek-v2; GPT-4o as simulated user. Metrics: resolve rate (Hidden / Interaction / Full settings) and detection accuracy/FPR/FNR.
- Claim "up to 74%": abstract and intro, "interactivity can boost performance on underspecified inputs by up to 74% over the non-interactive settings"; it is a relative improvement and depends on the model; the per-model resolve rates are in Figure 3 (graphic, not extracted), so which model gives 74% is not verified here.
- Interaction recovers "up to 80%" of the Full-setting performance for Claude Sonnet 3.5/Haiku 3.5; Deepseek 59% and Llama 54%; Claude Sonnet 4 89%.
- Detection (Table 2, accuracy at Neutral / Moderate / Strong encouragement): Sonnet 4 0.74 / 0.74 / 0.89 (FNR 0.44 / 0.42 / 0.18); Sonnet 3.5 0.60 / 0.84 / 0.76; Haiku 3.5 0.54 / 0.57 / 0.63; Qwen3 Coder 0.50 / 0.50 / 0.50 (FNR 1.00 at every level); Deepseek 0.69 / 0.57 / 0.51; Llama 3.1 0.48 / 0.47 / 0.52 (FPR 0.93 at Strong). Chance is 0.5. Quote: "Without explicit prompting, models almost never interact, even for severely underspecified inputs." "models struggle to distinguish well-specified from underspecified instructions."
- The "37.94 vs 59.52" from the blog is Table 1: Claude Sonnet 3.5 resolve rate without vs with navigational information (file paths), not interaction vs no interaction. Table 1 shows asking about navigational info helps weak models; Sonnet 4: 60.82 vs 67.24.
- Does not study UX/style: the underspecification is about bug behavior and file locations.
VERDICT: CONFIRMED (venue in the PDF). Models are 2024 to mid-2025 Claude; Sonnet 4 is the newest.

**8b ClarifyGPT.** Mu, Shi, et al. arXiv 2310.10996v1, 17 Oct 2023 (only version at the time of fetch). FSE 2024 venue not verifiable here. Method: code-consistency check (sample code from the requirement, compare behaviour on generated inputs); if inconsistent, ask targeted questions; refine; generate. Benchmarks: HumanEval, HumanEval-ET, MBPP-sanitized, MBPP-ET. Models GPT-4 and ChatGPT (gpt-3.5). Metric Pass@1. Abstract: "elevates the performance (Pass@1) of GPT-4 from 70.96% to 80.80% on MBPP-sanitized" (10 human participants); "improves the average performance of GPT-4 and ChatGPT across four benchmarks from 68.02% to 75.75% and from 58.55% to 67.22%" (simulated user feedback). Table: GPT-4 default 78.86 / 70.73 / 70.96 / 51.52 (avg 68.02); ClarifyGPT simulated 87.80 / 78.05 / 78.69 / 58.47 (avg 75.75); human feedback 80.80 on MBPP-S, 60.19 on MBPP-ET. The earlier docs' "62.43 to 69.60" is NOT in this text (grep: absent). It may be from the FSE camera-ready; unverified. VERDICT: existence and the 70.96 to 80.80 / 68.02 to 75.75 numbers CONFIRMED; 62.43/69.60 NOT-IN-PAPER (arXiv v1). Function-level, 2023 models.

**8c LLMREI.** Korn, Gorsch, Vogelsang. arXiv 2507.02564v1, 3 Jul 2025; "(c) 2025 IEEE" notice (RE 2025 not named in the extracted text beyond that). GPT-4o (gpt-4o-2024-08-06), zero-shot vs least-to-most prompting; a fine-tuning approach was abandoned. Evaluation: 33 simulated stakeholder interviews, errors compared with human interviewers from a prior study. Abstract: "LLMREI makes a similar number of errors compared to human interviewers". Elicitation: "being able to elicit up to 73.7 % of all requirements" = 60.94% fully plus 12.76% partially, short prompt. Conclusion validity limited ("only 33 interviews"). Measures interviewer error classes and requirement coverage, not downstream code quality. VERDICT: CONFIRMED (the exact percentages W2-06 lacked are 60.94 + 12.76 = 73.7).

**8d ReqElicitGym.** Jin et al., Peking University. arXiv 2602.18306v1, 20 Feb 2026, targeted at ACM TOSEM. 101 website requirement-elicitation scenarios, 10 application types, oracle user played by GPT-5.1 (evaluator GPT-5.2). Seven interviewers: GPT-5.2, Claude Opus 4.5, Gemini 3 Flash, DeepSeek V3.2, Kimi K2.5, GLM-4.7, Qwen3 235B A22B 2507 (current-generation). Metrics: IRE (implicit requirement elicitation ratio), TKQR (turn-discounted key question rate), ESR.
- Table 4 IRE (non-CoT / CoT): GPT-5.2 0.13 / 0.11; Claude Opus 4.5 0.08 / 0.07; Gemini 3 Flash 0.11 / 0.10; DeepSeek V3.2 0.32 / 0.19; Kimi K2.5 0.19 / 0.20; GLM-4.7 0.15 / 0.20; Qwen3 0.13 / 0.14. Quote: "Even the best-performing model only achieves an IRE of 0.32 under non-CoT (i.e., DeepSeek V3.2)". The abstract's "less than half" is therefore an understatement of the table: the best is under a third, the Claude and GPT models are 0.07 to 0.13.
- Table 6 per type (Interaction / Content / Style, non-CoT): GPT-5.2 0.19 / 0.13 / <0.01; Claude Opus 4.5 0.08 / 0.12 / <0.01; Gemini 3 Flash 0.12 / 0.16 / <0.01; DeepSeek V3.2 0.48 / 0.46 / 0.01; Kimi 0.31 / 0.23 / <0.01; GLM-4.7 0.21 / 0.20 / 0.01; Qwen3 0.20 / 0.19 / <0.01. Quote: "For almost all models and settings, IRE_Sty remains below 0.01, suggesting that LLM-based interviewers rarely elicit aesthetic requirements (e.g., visual style)."
- Note: the interviewer prompt forces exactly one question per turn or a stop signal; elicitation with a human-facing UX is different. A plain-LLM interviewer with no ambiguity check.
VERDICT: CONFIRMED. This is the best-supported statement for BUILD_PLAN's "models rarely ask about style/UX": direct numbers on current frontier models.

---

## Claim 9. arXiv 2607.13091

PAPER: "Self-Improving AI Coding Agents Through Accumulated Behavioral Rules: A Closed-Loop Framework", Aggarwal and Farhady Ghalaty (Microsoft). arXiv 2607.13091v1, 13 Jul 2026. Industry report; no venue.

WHAT EARLIER DOCS SAID: a single-deployment study "showing" that accepted/rejected feedback from earlier rounds improves later autonomous rounds (A-06 row 15: "PARTLY CONTRADICTED (weakly)" the claim that nothing exists).

WHAT IT ACTUALLY IS: a description of a workflow in which a human reviewer's accepted PR comment is turned (by an engineer) into a rule in a version-controlled instruction file (AGENTS.md style) that all later agent sessions load. Setup: microservices platform, 35+ services, several wks ("four-week deployment"), 11 recorded working sessions, 36 PR reviews across 6 repos, two agent interfaces (IDE-integrated and terminal). Rules grew 5 to 18 behavioral rules, 15+ code standards, 15-item checklist (Table I and III).
- Headline quote: "Across 9 tracked error classes with 74 cumulative post-rule session-exposures, zero recurrences were observed. We emphasize that this is an observational result inside a single deployment, not a controlled experiment".
- Review-comment mix (Table V, 36 PRs): 66% architecture/API/performance, 14% mechanical correctness and style. Presented as "review focus shifts", but there is no pre-rule comparison.
- Limits stated by the authors: "We do not have a parallel control group, and we did not run a paired ablation comparing the framework against static prompt engineering or a no-rule baseline"; "initial empirical evidence, not ... causal proof"; "we do not report p-values".
- The model is never named (no Claude/GPT string in the text); outcomes are self-reported by the authors who ran the deployment.
- Loop is human-in-the-loop: the engineer decides what becomes a rule ("The engineer who receives the review feedback makes this judgment"); signals are accepted human review comments only (7 of 18 rules; others from bots, self-discovery and production errors, Table II). Rejected-round feedback is not used. It is not a measurement of autonomous rounds improving.
VERDICT: does NOT show the claim as stated. What it shows: weak, uncontrolled, single-team evidence (9 classes, 74 exposures, 0 recurrences) that codifying accepted review comments into a loaded rules file coincides with no recurrence of those exact error classes. No baseline, so the recurrence rate without the rules is unknown. Claim to make in the plan: "plausible, weakly supported by one uncontrolled Microsoft deployment (arXiv 2607.13091); no controlled evidence found for accept/reject feedback improving autonomous rounds".

---

## Claim 10. Evolutionary / crossover with LLMs

- AlphaEvolve. arXiv 2506.13131v1, 16 Jun 2025, Google DeepMind (white paper, no venue). Models: ensemble of Gemini 2.0 Flash and Gemini 2.0 Pro. Results: 48-multiplication scheme for 4x4 complex matrices ("the first improvement, after 56 years, over Strassen's algorithm in this setting"); improved SOTA for 14 matrix-multiplication settings; on "over 50" open math problems "match the best known constructions on ~75% of them ... On ~20% of the problems, AlphaEvolve surpasses the SOTA". Production: a scheduling heuristic that "continuously recovers on average 0.7% of Google's fleet-wide compute resources"; a kernel tiling heuristic with "an average 23% kernel speedup across all kernels" and "a corresponding 1% reduction in Gemini's overall training time". Ablations (Section 4, Figure 8; on tensor decomposition and kissing numbers): removing evolution, prompt context, meta-prompt evolution, full-file evolution, or using only a small LLM each hurt ("each of the components is responsible for a significant improvement"); these are curves, not numbers I could extract. Crossover as a separate operator is not ablated.
- Numeric oracle requirement, verbatim: "While the use of an automated evaluation metric offers AlphaEvolve a key advantage, it is also a limitation--in particular, it puts tasks that require manual experimentation out of our scope." and "The main limitation of AlphaEvolve is that it handles problems for which it is possible to devise an automated evaluator." LLM-provided evaluation "is not a setting we have optimized for."
- ShinkaEvolve. arXiv 2509.19349 (Sakana AI). Abstract: "discovers a new state-of-the-art circle packing solution using only 150 samples" (n=26, sum of radii 2.635983). Numeric score is the fitness.
- OpenEvolve. No paper; project README (raw.githubusercontent) says "Matches published benchmarks for n=26 circle packing problem" and "+23% accuracy" on a HotpotQA prompt example. The 2.634 / 99.97% figure from earlier waves is NOT in the README I read. Competitor paper optimize_anything (arXiv 2605.19633v1, CAIS 2026, Table 3, same proposer GPT-5.1): optimize_anything 2.63598 in 63 evaluations, $3.18; OpenEvolve 2.4583 at 100 evaluations ($1.98), 2.6307 at 200 evaluations ($6.85). Competitor-authored, single task.
- FunSearch (Nature 2023): UNREACHABLE (nature.com proxy 403). No claim made from the paper itself.
- LLM crossover specifically (code): EvolRepair, arXiv 2604.02134v1 (2 Apr 2026), non-agentic program repair on 326 buggy solutions over 120 problems, backbones Llama 3.3 70B, Kimi K2, DeepSeek V3.1. Table 6 ablation (DeepSeek V3.1): full 96.63 Pass@1; without crossover 94.50; without mutation 88.11; pairwise recombination only 93.67; random grouping 93.67. So crossover is worth about +2.1 points Pass@1 against a ceiling near 97, mutation about +8.5. Fitness is test pass rate (numeric oracle again). The Bouras et al. genetic improvement result (8.5% fitness, 25.6% fewer variants) comes from a search snippet of a GECCO-workshop paper that I did not download; unverified.
VERDICT: CONFIRMED-WITH-SCOPE-CAVEAT. LLM-driven evolution has real measured gains, always with a machine-gradeable fitness (matrix multiplication count, packing sum, kernel runtime, test pass rate). No source tested evolution/crossover for qualitative "compare goals" or architecture choice. Doc 06's "merging not demonstrated" should read: not demonstrated without a numeric oracle; demonstrated with one.

---

## Claim 11. Spec-driven development tools: empirical work

Searched arXiv for Spec Kit, Kiro, BMAD, OpenSpec, "spec drift", "specification drift". Found, read:
1. Panda, arXiv 2606.30689v1 (28 Jun 2026), "Citation Discipline in Spec-Driven Development": pre-registered controlled studies comparing traceSDD (author's own), Spec Kit and OpenSpec with Claude Sonnet 4.6 (N=20, 4 conditions, 240 implementations) and GLM-5-turbo (N=50, 600). Outcomes: output determinism (lexical similarity across sessions) and hallucination detection rate. Result: traceSDD beats Spec Kit on determinism (Claude d=0.47, p=0.049; GLM d=0.42, p=0.003), not OpenSpec; the cited condition gives TDR 86.4% (Claude) and 88.0% (GLM) vs 0% for the alternatives. Single-author, author's own tool, measures determinism and traceability, not correctness or delivery.
2. SpecMine, arXiv 2608.25202v3 (1 Sep 2026, CMU): corpus of 470,795 spec.md files in 73,030 repos (17 tools) plus 98,574 Kiro files in 12,910 repos, 5,992 PRs touching specs across 581 repos for 11 tools. A dataset; the text raises "when the two drift apart, which side moves first?" as an open question. No effectiveness result.
3. de Macedo, arXiv 2606.04967v1 (3 Jun 2026): taxonomy and comparative assessment of Spec Kit, OpenSpec, BMAD, GSD, Spec Kitty, Reversa using a six-dimension rubric (specification, context, roles, execution, validation, portability). Qualitative scoring; names "drift between specification and code" as a recurring risk and calls for empirical evaluation ("a lack of benchmarks for the complete process").
4. Rosa et al., arXiv 2601.03878v1 (7 Jan 2026): study design only (CURRANTE VS Code plug-in, LiveCodeBench); no results.
5. Piskala, arXiv 2602.00180v1 (30 Jan 2026): practitioner guide. Its "up to 50%" error reduction is cited from other work ("Empirical studies [5], [6], though nascent, suggest ... up to 50%"), not measured there.
6. Tufano et al. (Google), arXiv 2608.17177v2 (21 Aug 2026): spec-driven TEST generation (agent first writes pre/post-conditions), Gemini 3 Flash: +9.8 pt bug detection (p=0.0352), +2.5 pt branch coverage. Same word, different thing: not Spec Kit/Kiro/BMAD.
7. Fawcett, arXiv 2608.23616v3 (20 Sep 2026): rebuild-dossier (locked interface contract); small comparisons; a weaker model with source plus one instruction matched or beat it. Independent researcher.
8. Goal drift (not spec drift): Saebo et al., arXiv 2603.03456v2 (24 Apr 2026, ICLR 2026 Workshop): GPT-5 mini, Haiku 4.5, Grok Code Fast 1 violate system-prompt constraints under environmental pressure, more so when the constraint opposes security/privacy values. Related to agents drifting from instructions, not to spec/code divergence in SDD tools.
9. A vendor benchmark (Uvik, 2026, 50 tickets, 42 vs 36 merged for OpenSpec vs no-spec) surfaced by search; blog, undated for the run, NOT a study; not read in full. Ignore.
VERDICT: "none found" would be WRONG for the existence of any empirical SDD work, but RIGHT for the key questions: no controlled study of whether Spec Kit, Kiro or BMAD improves correctness, delivery or rework; nothing for Kiro or BMAD outcomes at all; no measurement of spec drift rates. Doc 06 summary line "Their known failure modes (spec drift, ...)" is supported only by practitioner claims and by the de Macedo taxonomy's risk list, not by measurement.

---

## Claim 12. Correlated errors and frontier best-of-N

Correlated errors, general (not code):
- Kim, Garg, Peng, Garg, "Correlated Errors in Large Language Models", arXiv 2506.07962v1 (9 Jun 2025). 349 LLMs on a HuggingFace leaderboard (12,032 multiple-choice questions), 71 on HELM, 20 on resume screening. "models agree 60% of the time when both models err"; correlation higher for same provider, same base architecture, similar size; "larger and more accurate models have highly correlated errors, even with distinct architectures and providers." Multiple-choice and resume tasks; not code.
- Denisov-Blanch et al., "Consensus is Not Verification", arXiv 2603.06612v1 (20 Feb 2026): polling/self-consistency fails without an external verifier because errors are correlated (open-source models; no code).
Direct code evidence on same-family candidate diversity:
- Heterogeneous pool, frontier models. Kwok et al. (Stanford, Berkeley), "LLM-as-a-Verifier", arXiv 2607.05391v2 (7 Jul 2026), Table 3. SWE-bench Verified: heterogeneous pool of N=3 (one trajectory each from Claude Opus 4.5, Gemini 3 Flash, MiniMax M2.5), mean Pass@1 of the pool 76.1%, oracle Pass@3 84.4%, selected 78.2% (best single member 76.8%). Verifier Gemini 2.5 Flash, training-free, continuous logit scoring. Terminal-Bench V2: homogeneous N=5 from GPT-5.5, Pass@1 83.1%, oracle Pass@5 92.1%, selected 86.5%. So selection recovered 2.1 of 8.3 points of headroom (25%) on SWE-bench Verified and 3.4 of 9.0 (38%) on Terminal-Bench. No same-budget homogeneous baseline on SWE-bench, so heterogeneity's effect is not isolated. Authors are the verifier's developers.
- Pool-ceiling evidence: Yang et al., "A Single Patch Is Not Enough: Deterministic Fusion of Repair Candidates", arXiv 2607.01597v1 (2 Jul 2026): across systems/models on SWE-bench Verified the union of candidate patches reaches 443/500 against 396/500 for the best single source; a DeepSeek-V4-Pro listwise judge selects 396 and leaves 126 of the 443 reachable bugs unsolved; their test-free fusion method reaches 426/500. Pool diversity raises the ceiling; picking is the bottleneck. (Single paper; the best-single-source figure is the same as the listwise judge, 396.)
- Agentic Rubrics, arXiv 2601.04171v1 (7 Jan 2026, Scale AI), SWE-bench Verified Best@16 with open Qwen3 generators: oracle pass@16 51.4 vs random@16 22.6 vs best selector 40.6 (Qwen3-32B); oracle 65.6 vs random 39.6 vs 54.2 (Qwen3-Coder-30B-A3B). Weak/open generators.
- SWE-HERO (Claim 1): same-model K=32, Best@32 vs Pass@32 gap 15 to 18 points, open verifiers.
- Self-preference of LLM judges: Panickssery et al. 2410.21819 is already in Doc 06; I did not re-read it. A 2026 audit (2604.16790) of LLM-as-judge for SE exists; not read.
What is NOT found: a study that measures error correlation of candidates drawn from one Claude-family generator and then judged by a same-family judge on agentic coding, versus cross-family. Doc 06's inference that same-family panels are less independent remains an inference (general-purpose support from Kim et al.; the specific code case is untested).
Frontier best-of-N for agentic coding: only 2607.05391 (Opus 4.5 and GPT-5.5 era) reports it; gains are +2.1 to +3.4 points over mean Pass@1 with a closed-weights Claude/GPT/Gemini generator and a small open verifier model. No frontier numbers for K>=16.

---

## Numbers to change in docs/BUILD_PLAN.md and docs/research/06-options-and-autonomous-build.md

1. Doc 06 A1 SWE-HERO line: KEEP 64.6 / 79.8 but state it fully: "SWE-Hero-32B (SFT of Qwen3-Coder-480B, OpenHands), SWE-bench Verified, K=32, best of four open verifiers (SWE-Lego-Verifier-8B = 64.6); Pass@32 79.8; mean Pass@1 60.1 (headline 62.2). Gain over mean Pass@1: +7.9 / +5.2 / +4.5 points for 7B / 14B / 32B; paper text: gap 'reaches as much as 15% at K = 16'." The verifier is chosen post hoc on the test set. Source: arXiv 2604.01496v2 Table 3, Appendix A.2. Drop the "tag [S]" caveat: now read from the PDF.
2. Doc 06 A1 CodeMonkeys: add "Claude 3.5 Sonnet; $2,291.90 (Table 1); selection = top-3 by generated tests then a selection state machine; oracle coverage 69.8%, random 45.8%; 66.2% selects among 5 candidates, four being other teams' submissions (coverage 80.8%, best member 62.8, random 60.9)".
3. Doc 06 A1 CWM: keep 53.9 to 65.8; add "average of 4 runs; best@k with k=16 plus 40 generated unit tests; majority vote without tests 58.4; pass@40 80.4". The "40 unit tests" is confirmed; remove the [U] tag.
4. Doc 06 A1 Satori: change "matched a baseline needing Best@500 (~10x sampling)" to "Satori-SWE-32B (RL-trained self-evolution, Qwen2.5-Coder-32B, reward model plus unit tests) Best@50 = 41.6 vs Llama3-SWE-RL-70B (Agentless Mini) Best@500 = 41.0; greedy 35.8; sample-count ratio only, different models and trained refinement; not evidence for smarter allocation with an untrained model".
5. Doc 06 A1 R2E-Gym: keep "each verifier alone plateaus 43.7 / 42.8, hybrid 51.0 (Best@26; 49.4 at Best@16), Pass@1 34.4". Delete or soften the "test-agent rollouts more compute-efficient than editing rollouts" sentence (not found).
6. Stroebl (BUILD_PLAN Phase 6 / doc 06): phrase as "false-positive cost sets the optimal N; K <= 5 at cost-benefit ratio 4 on HumanEval+ with Llama-3.1 / Code Llama / Command / GPT-4o; much higher optimal K for some models at low cost ratios; no agentic or frontier-model test". Do not cite "<10" without the assumption.
7. BUILD_PLAN failure-rate paragraph / W2-06 item 5: restate with metrics and models:
   - Web-Bench: 25.1% Pass@1, Claude 3.7 Sonnet thinking, Apr 2025 (the 65.4% SWE-bench Verified comparison is in the abstract).
   - Commit0: 6.12% of unit tests (Claude 3.5 Sonnet, stage 1, "all" split, Dec 2024); 17.80% lite; 29.30% lite with feedback (abstract says 26%).
   - PaperBench: 21.0% rubric score (BasicAgent, Claude 3.5 Sonnet New); o1 IterativeAgent 24.4% (26.0% at 36 h); human 41.4% best-of-3 on a 3-paper subset vs o1 26.6% on the same subset. Do not pair 21.0 with 41.4.
   - ProjectEval: about 15% Pass@5 of test cases for GPT-4o (12.49% all-setting average).
   - Add 2026 frontier-model rows: ProgramBench (May 2026): 0% fully resolved for all 9 models, best Opus 4.7 passes >=95% of tests on 3.0% of tasks, $3.81 per task; ProjDevBench (Feb 2026): 27.38% acceptance, best final score 77.85 (Codex + GPT-5); RepoGenesis (Apr 2026): best 23.67% Pass@1 Python.
   - Tag [S] becomes [V] for these; remove "all old-model" caveat only for the two 2026 rows.
8. BUILD_PLAN Phase 7 "models rarely ask about style/UX requirements": now supported: ReqElicitGym (arXiv 2602.18306) style IRE below 0.01 for almost all of GPT-5.2, Claude Opus 4.5, Gemini 3 Flash, DeepSeek V3.2, Kimi K2.5, GLM-4.7, Qwen3; overall IRE 0.07 to 0.32 (Opus 4.5 0.08, GPT-5.2 0.13). Change "less than half of implicit requirements" to "best 0.32, Claude Opus 4.5 0.08". Ambig-SWE: detection accuracy 0.47 to 0.89 (Sonnet 4 reaches 0.89 only under strong encouragement); Qwen3 Coder never asks; drop the "37.94 vs 59.52" figure as an interaction effect (it is resolve rate without vs with file-location info for Sonnet 3.5).
9. ClarifyGPT: use "GPT-4 Pass@1 70.96 to 80.80 on MBPP-sanitized (10 humans); GPT-4 four-benchmark average 68.02 to 75.75 (simulated user); ChatGPT 58.55 to 67.22" and drop 62.43 / 69.60. LLMREI: add "GPT-4o, 33 simulated interviews, up to 73.7% of requirements (60.94 full + 12.76 partial), similar error count to human interviewers".
10. Doc 06 / BUILD_PLAN on 2607.13091: say "single uncontrolled Microsoft deployment, human-in-the-loop code review, 9 error classes, 74 post-rule exposures, 0 recurrences, no baseline, model not named; does not show accept/reject feedback improves autonomous rounds". Keep as hypothesis with instrumentation.
11. Doc 06 A5 evolutionary/merge: replace "not demonstrated" with "demonstrated only with a numeric fitness: AlphaEvolve (Gemini 2.0 Flash+Pro; 48-multiplication 4x4; ~75% match / ~20% improve on 50+ problems; needs 'an automated evaluator'), ShinkaEvolve (150 samples, circle packing), EvolRepair crossover +2.1 pt Pass@1 (96.63 vs 94.50) and mutation +8.5; OpenEvolve vs optimize_anything is competitor-run (2.6307 at 200 evals vs 2.63598 at 63 evals)". Remove the OpenEvolve 2.634 / 99.97% figure unless re-sourced; FunSearch: UNREACHABLE.
12. Doc 06 summary line on spec-driven tools: add "no controlled evidence of benefit for Spec Kit, Kiro or BMAD; empirical items found are Panda 2606.30689 (determinism/traceability, Claude Sonnet 4.6 and GLM-5-turbo, vs Spec Kit and OpenSpec), SpecMine 2608.25202 (corpus), de Macedo 2606.04967 (rubric). Spec drift is asserted by practitioners and listed as a risk, not measured." Replace "known failure modes" with "reported risks".
13. Add the frontier best-of-N data point to doc 06 A1: "LLM-as-a-Verifier (arXiv 2607.05391, Jul 2026): N=3 Opus 4.5 / Gemini 3 Flash / MiniMax M2.5 on SWE-bench Verified: pool Pass@1 76.1, oracle 84.4, selected 78.2; N=5 GPT-5.5 on Terminal-Bench V2: 83.1 to 86.5 (oracle 92.1)". Net selection gain is 2 to 3.5 points with a Gemini 2.5 Flash verifier, recovering 25 to 38% of the headroom.
14. Doc 06 A2 same-family panel: keep as UNVERIFIED inference; cite Kim et al. (agreement 60% when both wrong; higher for same provider) as general support and note no code-specific measurement.
