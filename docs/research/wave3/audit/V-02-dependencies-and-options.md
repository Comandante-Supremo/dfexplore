# V-02: Verification of R3-02 (dependency-update papers) and R3-03 (options and build papers)

Date: 2026-10-08. Verifier: independent model, not one of the readers.

Method. I downloaded 19 PDFs myself from `https://arxiv.org/pdf/<id>` into a new empty scratchpad directory (`scratchpad/v02pdf/`) and extracted the text with `pdftotext` (layout and raw modes) into a separate directory (`scratchpad/v02txt/`). For each number I grepped the text and read the surrounding section or table. I also checked the arXiv abstract-page title for all 59 arXiv ids cited in the two reader files. I did not route around any blocked host. The ACM, Zenodo and FSE items the readers marked UNREACHABLE stay UNREACHABLE, and I did not re-check them. All downloaded content was treated as data.

## Verdict

Both reader files are accurate on almost every number I checked. I reproduced all 18 claim groups at the stated table or figure, denominator, models and benchmark. No arXiv id is fabricated: all 59 resolve to the paper the readers describe.

Six claims need a correction. None of them reverses a reader verdict.

1. **Flaky-rerun rule.** The "any new failure, old all pass" rates of 10.9% (3 runs) and 3.0% (5 runs) hold only for a test that fails 50% of the time. Across all failure probabilities, the worst case for that rule is 25% at any number of runs. Only the strict all-fail/all-pass rule gets better as you add runs.
2. **Stroebl.** The "K <= 5 at cost-benefit ratio 4 for all four models" statement covers the four Llama-3.1 and Code Llama models in Figure 4 only. It does not cover Command or GPT-4o.
3. **LLM-as-a-Verifier (2607.05391).** The verifier, Gemini 2.5 Flash, is a closed API model. It is not a "small open verifier model". It scores with repeated pairwise passes (G=20).
4. **ProgramBench.** The 20-36% cheating rate comes from an open-internet ablation, not from the main runs. The 9 judges disagreed on 40-57% of tasks.
5. **ProjDevBench.** The benchmark has only 20 problems, and 27.38% is the share of submissions accepted.
6. **Byam range in the BUILD_PLAN edit.** R3-02's proposed "23-27%" range rests on an unread Fruntke-Krinke number. Byam itself supports "best cell 27% (28/103) of 40 cells; o3-mini 18-27%".

There is also one mis-attribution: arXiv 2410.21819 is Wataoka et al., not Panickssery et al.

## Claim table

Ratings: REPRODUCED, REPRODUCED-WITH-CORRECTION (with the corrected statement), NOT-REPRODUCED, UNREACHABLE.

| # | Claim (paper, arXiv id) | Reader said | Paper says (where) | Rating |
|---|---|---|---|---|
| 1 | Hejderup & Gousios, 2109.11921 | Tests detect 47% of direct and 35% of transitive injected faults. These are means of per-project detection scores (medians 51%/36%), on 1,122,420 artificial updates in 311 modules of 262 Java/Maven projects. Coverage of 58%/20% is a separate metric. | Abstract and Sec. 5.2: "median detection rate score is 51% (mean: 47%) ... 36% (mean: 35%)". Detection Score = detected mutants / all mutants, built on PITest. Coverage: abstract 58%/20%, text median 58% (mean 55%), transitive median 21% (mean 26%), sample of 521 projects. Threats section: "mutation operators do not substitute actual regression changes". Uppdatera: median 97%/88%, mean 74%/64%. | REPRODUCED |
| 2 | Venturini et al., 2301.04563 | 44% is the share for each of minor and patch (28/64 each). Minor plus patch is 87.5% (56/64). Sample: 384 npm clients. | Abstract: "44% were introduced both in minor and patch releases". Table 4: Major 3, Minor 28 (43.75), Patch 28 (43.75), Pre-release 5, total 64. Note: the Finding 4 text says "26 in minor", which disagrees with the table. The table sums to 64; the text version sums to 62. | REPRODUCED (add: the Finding 4 text gives 26, the table 28; use the table) |
| 3 | Go / GoSVI, 2309.02894 | 28.6% = 17,009 of 59,527 non-major upgrades, measured by static API diff. 3.1% is per breaking change. 33.3% is per client of a breaking upgrade ("may be affected"). | Table III: 17,009 / 59,527 = 28.57%. 363,428 breaking changes. Table IV gives 11,304 used by clients (3.11%); the text gives 11,034 (3.04%). 50,485 / 151,589 = 33.30% of clients of the 17,009 non-major breaking upgrades. | REPRODUCED |
| 4 | BreakGuard, 2608.20167 | 30.3% = 27/89 BUMP test-failure instances, GPT-4o with class context, best of 9 configurations. 95.1% is the share of detecting tests that crash, not precision. | Abstract and Table V: GPT-4o Class 27 (30.3%). Other cells 4-20 of 89. Filter funnel 571 → 188 → 179 → 164 → 104 → 89. Section on test outcomes: "Among the 3,566 detecting tests, 3,390 (95.1%) terminate with a Maven Surefire ERROR ... only 176 (4.9%) ... FAILURE". The only use of "precision" (line 75) is a citation of other work. The paper reports no precision or false-alarm rate. | REPRODUCED |
| 5 | DepBench, 2608.30300 (paper title "Update from Hell: Can Coding Agents Survive Hidden Breakage in Dependency Upgrades?") | 104/203 is the best of 10 harness-model configurations. Claude Code + Opus 4.8 solved 79/203. Python has 10 tasks (best 6). The four-state oracle certifies benchmark tasks; it is not a gate on agent output. | Abstract: "best completed configuration solves only 104/203 tasks (51.2%)". Table II: GPT-5.5 row 71/104/99/80, Opus 4.8 row 79/–/83/80, Gemini 3.5 Flash row 53/–/52/51, so 10 configurations. Definition 1, "Oracle-clean ... task": Ω(b)=pass ∧ Ω(b⊕m⊕t)=fail ∧ Ω(b⊕m⊕c⊕t)=pass ∧ Ω(b⊕c⊕t)=fail. It is applied "before release" to candidate tasks built from the developer repair c and held-out tests t. Agents are scored by a pass/non-pass verifier. Table III: Python best 6/10. | REPRODUCED |
| 6 | Byam, 2505.07522v3 | 27% = 28/103, o3-mini with prompt P8, best of 40 cells. o3-mini ranges 18-27%. 741/955 = 77.6% of errors fixed. The 103 cases are a compile-failure subset of BUMP. | Table 4: 5 models × 8 prompts = 40 cells. o3-mini P8 28/103 (27%), described as "completely repaired, with compilation and test execution succeeding". o3-mini column 19-28 of 103 (18-27%). Other cells range down to 7/103. Error level: 741/955 (78%). The introduction says 79%. | REPRODUCED |
| 7 | Cooldown, Tanaka et al., 2609.16605 | 64.3% = 157 of 244 ecosystem-level configs that set default-days. 135 adopters among 1,462 popular repos. No effectiveness evidence. | Sec. 4: "most common value is 7 days, which occurs in 157 of them, or 64.3%", out of 244 that set it (97.2% of 251). Values range from 2 to 42 days. Text: "135 adopters and 1,327 non-adopters". Conclusion: "not evidence that cooldown prevents supply chain incidents". | REPRODUCED |
| 8 | nf-core, Alam et al., 2607.10839 | Renovate PRs merged 98.46% vs Dependabot 72.13%, across 35,411 nf-core PRs. Observational and confounded. | Text: Renovate 98.46% (median 0.43 d), Dependabot 72.13%, other bots 51.94%, human 96.93%. 32,518 of 35,411 merged (91.83%). Table 10, bivariate: Renovate present vs absent 98.46% vs 92.49%, OR 5.01. Auto-merge 99.51% vs 92.53%, OR 13.93. Also, not in the reader file: the multivariable model (Table 13) gives Renovate an adjusted OR of 6.59. | REPRODUCED. Add: adjusted OR 6.59. Still observational within one ecosystem; the paper does not attribute the gap to tool quality. |
| 9 | Flaky arithmetic and Gruber, 2101.09077 | 0.5^6 = 1.56%; 0.5^10 = 0.098%. At-least-one-fail rule 10.9%/3.0%. Majority rule 25%. Gruber's 170 reruns is a flakiness-exposure figure, not a stability rule. | Gruber, abstract: "95 % confidence that a passing test case is not flaky on average would require 170 reruns". RQ3: 31 runs for order-dependent tests, 472 for 95% confidence. My recomputation (table below) matches every number at p=0.5. | REPRODUCED-WITH-CORRECTION: 10.9%/3.0% are p=0.5 values. The worst case of the at-least-one-fail rule over p is 25% for every n (e.g. p≈0.13 at n=5). For odd n the majority rule is also 25% at worst. Only the strict rule falls with n (worst case 0.25^n). |
| 10 | SWE-HERO, 2604.01496v2 | Table 3: Best@32 64.6 (SWE-Hero-32B + SWE-Lego-Verifier-8B, best of 4 verifiers). Pass@32 79.8. Mean Pass@1 of 32 rollouts 60.1. Gains 7.9/5.2/4.5 are Best@32 minus mean Pass@1. The text says the gap "reaches as much as 15% at K = 16". | Appendix A.2, Table 3 matches the reader's table. Arithmetic: 57.9-50.0=7.9, 62.6-57.4=5.2, 64.6-60.1=4.5. Text quotes found verbatim. The headline 62.2% is a different, single-run number. The row grouping of the Pass@1 and Pass@32 columns is inferred from layout but is consistent with the authors' own gain figures. | REPRODUCED |
| 11 | CodeMonkeys, 2501.14723v2 | 57.4% single system, about $2,300 (Table 1: $2,291.90). 66.2% is selection over 5 candidates per issue, 4 of them from other teams' leaderboard submissions. Coverage 80.8%, best member 62.8%, random pick 60.9%. | Abstract and Sec. 2.3.1 "Barrel of Monkeys": the pool is CodeMonkeys' final edit plus "submissions from the top four entries" (Blackbox AI, CodeStory Midwit + swe-search, Learn-by-interact, devlo), "five samples per problem", coverage 80.8%, 66.2% vs 62.8% best member and 60.9% random. The test-based pre-filter is skipped. | REPRODUCED |
| 12 | Stroebl et al., 2411.17501v3 | "<10" depends on an assumed cost-benefit ratio. K <= 5 at ratio 4. Models: Llama-3.1, Code Llama, Command, GPT-4o. | Abstract: "optimal sampling attempts are often fewer than 10". Sec. 4: ratios 0/1/2/4/8; "at a cost-benefit ratio of 4, the optimal number of samples is K ≤ 5 for all four models". The "four models" are the Llama-3.1 and Code Llama families in Figure 4; GPT-4o is in Figure 15. Caveat (Command and Code Llama): "much higher values of the optimal K, especially for low cost-benefit ratios". Setup: resample until a sample passes the original HumanEval tests; a false positive is a sample that passes those but fails HumanEval+. | REPRODUCED-WITH-CORRECTION: K <= 5 at ratio 4 holds for the four Llama-3.1 and Code Llama models in Fig. 4. The Command family can have a much higher optimal K. The setting is "resample until weak tests pass" at function level, with no agentic or frontier tests. |
| 13 | LLM-as-a-Verifier, Kwok et al., 2607.05391v2 | SWE-bench Verified: N=3 heterogeneous pool (Opus 4.5, Gemini 3 Flash, MiniMax M2.5), pool mean Pass@1 76.1, oracle 84.4, selected 78.2, best single 76.8. Terminal-Bench V2: N=5 from GPT-5.5, 83.1 → 86.5, oracle 92.1. Gemini 2.5 Flash verifier, training-free. R3-03 line 247 calls it "a small open verifier model". | Sec. 5.1 and 5.2: all numbers match. Scaffolds: mini-swe-agent for SWE-bench, Capy for Terminal-Bench. Gemini 2.5 Flash is a proprietary API model, used for its exposed logprobs, with repeated scoring (G=20) and pairwise comparison. The paper's appendix loosely calls it an "open verifier" in contrast to GPT-5.5. | REPRODUCED-WITH-CORRECTION: the verifier is closed-weights Gemini 2.5 Flash, accessed through its API logprobs with repeated pairwise passes (G=20), so its cost is not one call. Selection gain: +2.1 points over the pool mean and +1.4 over the best single model on SWE-bench Verified; +3.4 points on Terminal-Bench. The authors built the verifier. |
| 14a | ProgramBench, 2605.03546v1 | 200 tasks, 9 models, 0% fully resolved. Opus 4.7 passes >=95% of tests on 3.0% of tasks, $3.81 per task. Cheating flagged on 20-36% of tasks for the three stronger models. | Table 2: 0.0% resolved for all 9 models. "Almost" (>=95%): Opus 4.7 3.0, Opus 4.6 2.5, Sonnet 4.6 1.6, all others 0.0. $3.81 per task for Opus 4.7. The scaffold is mini-SWE-agent with internet blocked. The 20-36% cheating rate comes from a separate "open internet with cheating detection" run, one model per provider family, in which "judges disagree on 40–57% of tasks". | REPRODUCED-WITH-CORRECTION: the 20-36% cheating rate is from the open-internet ablation, not from the main results. Judge disagreement there was 40-57%. "Cheating" means looking up the source or wrapping the reference executable; the paper does not mention decompiling. |
| 14b | ProjDevBench, 2602.01655v2 | 27.38% acceptance (484 accepted). Best final score 77.85 (Codex + GPT-5); Claude Code + Sonnet 4.5 68.87. | Abstract: "overall acceptance rate of 27.38%". Text: "Only 27.38% of submissions were accepted". Final score = 80% execution + 20% code review. The benchmark has "20 programming problems across 8 categories". | REPRODUCED-WITH-CORRECTION: only 20 problems, and 27.38% is the share of online-judge submissions accepted, not a project success rate. |
| 15 | ReqElicitGym, 2602.18306v1 | Style IRE below 0.01 for almost all of 7 current models. Overall IRE 0.07-0.32 (Opus 4.5 0.08, GPT-5.2 0.13, DeepSeek V3.2 0.32). | Table 4 and Table 6 match exactly. The RQ3 quote "IRE_Sty remains below 0.01" is verbatim. The simulated user is GPT-5.1, and the 101 scenarios are all websites. Not in the reader file: Opus 4.5 ran only 6.4 turns on average (non-CoT) vs GPT-5.2's 20.0, so its low IRE partly reflects stopping early. | REPRODUCED (add the turn-count caveat) |
| 16 | Ambig-SWE, 2502.13069v3 | Detection accuracy 0.47-0.89 across models and prompts. Sonnet 4 reaches 0.89 only under strong encouragement. Qwen3 Coder's false-negative rate is 1.00. Venue ICLR 2026. | PDF header: "Accepted at ICLR 2026". Table 2 matches. 0.47 is Llama 3.1 under moderate encouragement, below chance (0.5). Quote: "Without explicit prompting, models almost never interact". | REPRODUCED |
| 17 | ClarifyGPT, 2310.10996v1 | GPT-4 four-benchmark average 68.02 → 75.75 with a simulated user. 70.96 → 80.80 on MBPP-sanitized with human feedback. The earlier docs' 62.43/69.60 is not in the paper. | Abstract and tables confirm. v1 is the only arXiv version; 62.43 and 69.60 do not appear anywhere in the text. Minor inconsistency in the paper: the introduction attributes the MBPP-ET human result (51.52 → 60.19) to ChatGPT, while RQ1 and the table attribute it to GPT-4. | REPRODUCED |
| 18 | Aggarwal & Farhady Ghalaty, 2607.13091v1 | One uncontrolled Microsoft deployment. Engineer-curated rules built from accepted human review comments. 9 error classes, 74 post-rule exposures, 0 recurrences. No control group, model not named, not autonomous rounds. | All counts verbatim: 35+ services, 11 sessions, 36 PRs in 6 repos, four weeks. "We do not have a parallel control group". "The engineer who receives the review feedback" decides what becomes a rule. No model name appears in the body. | REPRODUCED |

UNREACHABLE (not re-attempted, agreeing with R3-02): Jayasuriya ISSTA 2023 (11.58%) and FSE 2024 (2.30%), Fruntke-Krinke (19%/23%), BDUpdater (90.5%), the Ferdous thesis (53%). None of these numbers should go into BUILD_PLAN as verified.

## arXiv id integrity

- **The 19 PDFs I opened** all match the described paper and arXiv version: 2109.11921, 2301.04563, 2309.02894, 2608.20167, 2608.30300, 2505.07522, 2609.16605, 2607.10839, 2101.09077, 2604.01496, 2501.14723, 2411.17501, 2607.05391, 2605.03546, 2602.01655, 2602.18306, 2502.13069, 2310.10996, 2607.13091.
- **The other 40 cited ids** have abstract-page titles that match the reader descriptions: 2605.24397, 2401.09906, 2510.03480, 2206.07230, 2609.25911, 2510.08609, 2407.03880, 2408.05129, 2609.25587 (Galaxy), 2505.23604, 2504.07164, 2510.02387, 2505.07473, 2412.01769, 2504.01848, 2503.07010, 2507.02564, 2506.13131, 2509.19349, 2605.19633, 2604.02134 (EvolRepair), 2606.30689, 2608.25202, 2606.04967, 2601.03878, 2602.00180, 2608.17177, 2608.23616, 2603.03456, 2506.07962, 2603.06612, 2607.01597, 2601.04171, 2604.16790, 2403.08604, 2308.00352, 2510.14509, 2601.13943, 2307.07924, and 2410.21819 (see the next item).
- **None looks fabricated.**
- **Mis-attribution:** R3-03 Claim 12 cites "Panickssery et al. 2410.21819". arXiv 2410.21819 is "Self-Preference Bias in LLM-as-a-Judge" by Wataoka, Takahashi and Ri. Panickssery et al. ("LLM Evaluators Recognize and Favor Their Own Generations") is a different paper. Fix the author or the id wherever Doc 06 uses it.
- **Naming note:** the 2608.30300 paper is titled "Update from Hell". DepBench is the benchmark's name inside it. The id is correct.

## Flaky-test rerun table (recomputed)

Model: the test is flaky with the same per-run failure probability p on the old and new versions (there is no real regression), and runs are independent. Each cell is the probability that the rule wrongly blames the new version. n is the number of runs on each side.

| Rule | p | n=2 | n=3 | n=4 | n=5 | n=10 | Worst case over p |
|---|---|---|---|---|---|---|---|
| Strict: new fails all n, old passes all n | 0.5 | 6.25% | 1.56% | 0.39% | 0.098% | 0.0001% | 0.25^n (at p=0.5) |
| | 0.3 | 4.41% | 0.93% | 0.19% | 0.041% | ~0 | |
| | 0.1 | 0.81% | 0.073% | 0.007% | 0.0006% | ~0 | |
| Any: new fails at least once, old passes all n | 0.5 | 18.75% | 10.94% | 5.86% | 3.03% | 0.10% | 25% for every n (p≈0.29/0.21/0.16/0.13/0.07) |
| | 0.3 | 25.0% | 22.5% | 18.3% | 14.0% | 2.7% | |
| | 0.1 | 15.4% | 19.8% | 22.6% | 24.2% | 22.7% | |
| Majority: majority of new fail, majority of old pass (odd n) | 0.5 | – | 25.0% | – | 25.0% | – | 25% (at p=0.5) |
| | 0.1 | – | 2.7% | – | 0.85% | – | |

Every one of R3-02's figures is correct at p=0.5: 1.6%, 0.1%, 10.9%, 3.0% and 25%. The missing point is the worst case. Under the "any failure" and majority rules, more reruns do not bound the false-attribution rate. A test that fails about 13% of the time defeats the "any failure" rule at n=5 exactly as often (25%) as a 50/50 test defeats the majority rule.

**Recommended rule:** strict paired attribution. Run the failing tests k times on each side, interleaved. Blame the update only if every new run fails and every old run passes. Any mixed result is "flaky or unattributed": escalate, or record it in the flake history, and do not auto-attribute. With sequential stopping, stop at the first new pass or first old fail. Early stopping cannot raise the false-attribution rate, because attribution still needs all k results on both sides.

Choose k from the false-positive budget per flaky test, using the worst case 0.25^k:

| Budget per flaky failing test | Smallest k | Worst-case false attribution |
|---|---|---|
| ≤ 10% | 2 | 6.25% |
| ≤ 2% | 3 | 1.56% |
| ≤ 0.5% | 4 | 0.39% |
| ≤ 0.1% | 5 | 0.098% |

The budget scales with the number of flaky tests that fail at once. With F such tests, the chance of at least one false attribution is about F × 0.25^k. BUILD_PLAN's current "k = 5 with sequential stopping ... mixed results escalate" (line 70) already is this rule, at a 0.1% budget. Keep it, and state that attribution needs 5/5 failures on new and 5/5 passes on old.

Three costs to state in the plan:

- The strict rule misses an intermittent real regression; it escalates the case instead.
- The independence assumption fails for order-dependent and environment flakiness. Interleaving and randomising order help; they do not remove the problem.
- Do not quote "3.0% at 5 runs" for any looser rule.

## Edits to BUILD_PLAN drawn from the readers' lists

### Safe to apply

1. **Line 22, four-state check.** Re-source it as "adapted from DepBench's task-validation oracle (arXiv 2608.30300, Definition 1). DepBench applies it to benchmark tasks before release, not to agent output." Add: "DepBench: best of 10 agent configurations 104/203 (51.2%); Claude Code + Opus 4.8 79/203; only 10 Python tasks." (R3-02 #2)
2. **Line 22, repair rate.** Replace "19 to 27%" with: "best measured: 27% (28/103) of builds fully repaired (compile and tests), o3-mini, best of 40 model × prompt cells, on a Java/Maven compile-failure subset of BUMP (Byam, arXiv 2505.07522); o3-mini ranges 18-27% across prompts." Do not adopt R3-02's "23-27%": the 23% is Fruntke-Krinke, from a search snippet only. (R3-02 #1, corrected)
3. **Line 67, cooldown.** Use: "64.3% (157 of 244) of ecosystem-level cooldown configs that set a default delay use 7 days; 135 adopters among 1,462 popular repos; the paper offers no effectiveness evidence." (R3-02 #3)
4. **Line 74.** Keep "no study covers CLI contract breaks", and note that Python evidence is narrow (Montandon, 2408.05129). (R3-02 #4)
5. **Line 153, nf-core.** Restore 98.46%/72.13% as verified (arXiv 2607.10839), described only as "observed nf-core merge rates, observational, confounded by adoption and auto-merge config". Keep the Gruber-170 rejection and give the reason: 170 is the number of reruns needed to expose flakiness, not a stability rule. (R3-02 #5)
6. **Line 136, blocked list.** arXiv is now readable. Mark the arXiv items as read and keep the ACM-only items (Jayasuriya ISSTA/FSE, Fruntke-Krinke, BDUpdater) as UNREACHABLE. (R3-02 #6)
7. **Line 70, reruns.** Add the attribution rule and the worst-case table above. If the plan quotes 1.6%/0.1%, say these are for the strict rule; for "any failure" or majority rules the worst case is 25% regardless of the number of runs. (R3-02 #7, corrected)
8. **Line 149, SWE-HERO.** Keep 64.6/79.8 and state it fully: SWE-Hero-32B, SWE-Lego-Verifier-8B, best of 4 open verifiers chosen on the test set; mean Pass@1 60.1; gains +7.9/+5.2/+4.5 points. (R3-03 #1)
9. **CodeMonkeys, CWM, R2E-Gym, Satori.** Add the scope text exactly as in R3-03 #2-#5. I re-checked the CodeMonkeys numbers; the others were not in my sample but are consistent with A-06.
10. **Phase 7, "models rarely ask about style/UX".** Cite ReqElicitGym (2602.18306): style IRE below 0.01 for almost all of 7 current models; overall IRE 0.07-0.32 (Opus 4.5 0.08, about 6 turns). Cite Ambig-SWE for detection accuracy 0.47-0.89 (Sonnet 4 reaches 0.89 only under strong encouragement), and drop "37.94 vs 59.52" as an interaction effect. (R3-03 #8)
11. **ClarifyGPT.** Use 70.96 → 80.80 (GPT-4, MBPP-sanitized, 10 humans) and 68.02 → 75.75 (GPT-4, four-benchmark average, simulated user). Drop 62.43/69.60. (R3-03 #9)
12. **Phase 7, DoD lessons file.** Cite 2607.13091 as "single uncontrolled Microsoft deployment; human-curated rules from accepted review comments; 9 classes, 74 exposures, 0 recurrences; no baseline; model not named; not evidence about autonomous rounds". (R3-03 #10)
13. **Failure-rate paragraph.** Add the 2026 frontier rows. ProgramBench: 0/200 resolved for all 9 models; Opus 4.7 passes >=95% of tests on 3.0% of tasks. ProjDevBench: 27.38% of submissions accepted, 20 problems, best weighted score 77.85. Keep the metrics separate. (R3-03 #7, corrected)
14. **Phase 6, best-of-N.** Add 2607.05391: pool mean 76.1 → selected 78.2 vs oracle 84.4 (N=3, mixed models, SWE-bench Verified); 83.1 → 86.5 vs 92.1 (N=5 GPT-5.5, Terminal-Bench V2); verifier Gemini 2.5 Flash (closed API, repeated pairwise scoring, built by the authors). This supports keeping N small and measuring the selector. (R3-03 #13, corrected)

### Do not apply, or apply only as corrected

1. R3-02 #1's "about 23-27%" range. It depends on unread Fruntke-Krinke figures; use edit 2 above.
2. R3-02 #7's "any-failure 10.9%/3.0%" as general rates. These are p=0.5 values only; the worst case is 25%.
3. R3-03 #6's statement that K <= 5 at ratio 4 holds for "Llama-3.1 / Code Llama / Command / GPT-4o". It holds for the four Llama-3.1 and Code Llama models only. Phrase the Stroebl result as "false-positive cost sets the optimal N; measure it" and do not cite a numeric cap.
4. R3-03 Claim 12's wording "small open verifier model" for Gemini 2.5 Flash.
5. ProgramBench's "cheating on 20-36% of tasks" as a main-run figure. It is an open-internet ablation with 40-57% judge disagreement.
6. Any figure the readers marked UNREACHABLE, as verified: 11.58%, 4.35%, 2.30%, 19%/23%, 90.5%, 53% (Ferdous), and the incident windows (vendor snippets only).
7. Any citation of "Panickssery et al." with id 2410.21819, until the author or id is fixed.
8. Do not mark the Doc 06 same-family-judge correlation claim as measured. It remains an inference (R3-03 #14 already says so; keep it that way).
