# W2-06 Options exploration and autonomous build: literature verification

Researched 2026-10-07 (wave two). Scope: parallel option exploration (best-of-N, selection, evolutionary search) and autonomous greenfield builds (benchmarks, elicitation, spec-driven evidence, feedback loops).

## 0. Method limits (read first)

- Full-text access failed. `WebFetch` returned `getaddrinfo ENOTFOUND` for arxiv.org, export.arxiv.org, huggingface.co and scalingintelligence.stanford.edu. `curl` through the agent proxy returned `403 CONNECT tunnel failed` (organization policy) for arxiv.org, export.arxiv.org, ar5iv.org, ar5iv.labs.arxiv.org, api.semanticscholar.org, huggingface.co, alphaxiv.org, openreview.net, aclanthology.org, dl.acm.org, nature.com and scalingintelligence.stanford.edu. Only raw.githubusercontent.com answered (200), and I did not need it. I did not try to bypass the policy.
- All evidence below therefore comes from `WebSearch`, which returns summaries of arXiv abstracts, proceedings pages and index pages, plus secondary write-ups. I read no paper body. Tags:
  - **[V]** = the claim, with its numbers, was returned from a primary-source listing (arXiv/proceedings/abstract text) with an identifiable id and venue. Abstract-level only, and mediated by the search tool.
  - **[S]** = reported only by a secondary site (blog, review aggregator) citing the paper. Treat as unverified.
  - **[U]** = no source found, or my own inference or design guess.
- Nothing here has been checked against a paper table. Anything used as a hard default should be re-checked against the PDF once a network path to arxiv.org exists.

## 1. Summary

1. Fan-out is real but verifier-limited. Coverage (pass@N) rises roughly log-linearly over orders of magnitude of samples, but realized gains depend on selection. With imperfect verifiers the best N is often below 10, and the curve can bend downward. [V]
2. Wave one's headline figures for CodeMonkeys, CWM, Satori-SWE and R2E-Gym mostly check out at abstract level. Its SWE-HERO figures ("Best@32 64.6 vs Pass@32 79.8") could not be confirmed, and the paper is a fine-tuning paper, not a selection study.
3. Doc 06 says merging across candidates is "not demonstrated". That is too strong: LLM crossover has measured gains in genetic-improvement and program-repair work, and AlphaEvolve/FunSearch/ShinkaEvolve are population-based searches with measured results. But all of these have a numeric fitness function, which a general compare goal lacks.
4. Doc 06 says no strong primary source exists for conversational elicitation. That is wrong. Ambig-SWE (ICLR 2026), ClarifyGPT (FSE 2024), LLMREI (RE 2025) and ReqElicitGym exist. Key finding: agents under-ask and cannot detect underspecification, and interaction helps strongly when they do ask.
5. Autonomous greenfield builds still fail most of the time on public benchmarks: Web-Bench 25.1% Pass@1 for the best model at the time, Commit0 6% of unit tests on the full set, PaperBench 21.0% (agent) vs a human baseline roughly double, ProjectEval about 15% Pass@5 for GPT-4o. All numbers are from 2024 to mid-2025 models. They are lower bounds for today, not predictions. [V]
6. No controlled empirical study of Spec Kit, Kiro or BMAD effectiveness was found. Closest: a registered report (SANER 2026, no results) and a single-author comparison of SDD frameworks on determinism and traceability, not quality. [V]
7. No numeric defaults for halving schedules or clarifying-question counts have primary evidence. Defaults in section 5 are labelled evidence-backed or guess.

## 2. Verified findings

### 2.1 Best-of-N, selection and the verifier bottleneck

- **Large Language Monkeys** (Brown et al., arXiv 2407.21787, 2024). On SWE-bench Lite, DeepSeek-Coder-V2-Instruct goes from 15.9% (1 sample) to 56% (250 samples), above the 43% single-attempt SOTA then. Coverage grows log-linearly over four orders of magnitude. Where no automatic verifier exists, majority voting and reward models plateau beyond several hundred samples. Cost point: 5 samples of a cheap model beat one sample of GPT-4o or Claude 3.5 Sonnet at API prices of the time. [V]
- **CodeMonkeys** (Ehrlich et al., arXiv 2501.14723, 2025). 57.4% on SWE-bench Verified with Claude Sonnet 3.5 (the Stanford project page abstract says 57.7%; the arXiv abstract says 57.4%), at roughly $2,300 for the benchmark. Selecting over an ensemble of edits from top existing submissions gives 66.2%, above the best single member. Selection = voting using model-written tests, then a final multi-turn selection trajectory. Parallel scaling (more trajectories) and serial scaling (more iterations) are both used. [V]
- **CWM** (Meta, arXiv 2510.02387, Sept 2025). 32B open-weights model; 53.9% SWE-bench Verified without test-time scaling, 65.8% with it. [V] Wave one's detail "k candidates and 40 generated unit tests" was not confirmed. [U]
- **Satori-SWE / EvoScale** (arXiv 2505.23604, 2025). Evolutionary test-time scaling: outputs are iteratively refined by selection and mutation, with RL training the model to improve its own scores, reducing reliance on external verifiers at inference. 41.6% Best@50 (50 total samples accumulated over iterations, selection by reward model plus unit tests) matches Llama3-SWE-RL-70B's Best@500. Greedy accuracy 35.8%. Caveat: the 10x saving compares different models (32B EvoScale vs 70B baseline) and is a trained-in refinement loop, not a scheduling trick for generic best-of-N. [V]
- **R2E-Gym** (arXiv 2504.07164; NeurIPS 2025 listing). 8.1K+ procedurally built tasks (SWE-GEN), 34.4% pass@1 for a 32B model. Execution-based test verifiers (low distinguishability) and execution-free verifiers (biased toward stylistic features) each plateau near 42-43%; the hybrid reaches 51% on SWE-bench Verified. [V] Wave one's sentence "more test-agent rollouts can be more compute-efficient than more editing rollouts" was not confirmed in what I retrieved. [U]
- **SWE-ZERO to SWE-HERO** (NVIDIA, arXiv 2604.01496, April 2026). A two-stage SFT recipe (about 300k execution-free trajectories, then 13k execution-based, distilled from Qwen3-Coder-480B). 32B reaches 62.2% on SWE-bench Verified; skipping stage one gives 55.7%. Under parallel scaling with K=32 rollouts and a generative verifier, gains are 7.9 and 5.2 points for 7B and 14B and 4.5 points for 32B, and the gap between verifier selection and the oracle ceiling reaches up to about 15% at K=16 and widens with K. [S] for the parallel-scaling details (secondary summary of the paper); [V] for the existence, authorship and SFT results.
- **Limits of inference scaling through resampling** (Stroebl, Kapoor, Narayanan; arXiv 2411.17501; ICLR 2026). With an imperfect verifier (coding unit tests of limited coverage), false positives cap accuracy regardless of compute. Weaker models have higher false-positive rates; they report that optimal sampling attempts are often fewer than 10, because false-positive costs outweigh benefits and the scaling curve bends downward. Evidence is from HumanEval and MBPP. [V]
- **ROC analysis of verifier-based scaling** (arXiv 2507.12399). Rejection sampling beats best-of-N at equal compute for concave verifier ROC curves; both converge as compute goes to infinity; high-compute performance cannot in general be extrapolated from low-compute runs. Evidence from GSM8K/MATH500, not code. [S]
- **Compute-optimal TTS** (Snell et al., arXiv 2408.03314, ICLR 2025). Adaptive per-prompt allocation beats fixed best-of-N by over 4x efficiency on MATH; a smaller model with test-time compute beats a 14x larger model only where the small model's base success is non-trivial; beam search beats best-of-N at low budget but flattens. Math, not agents. [V]
- **Scaling test-time compute for LLM agents** (Zhu et al., arXiv 2506.12928, 2025). Systematic study on GAIA validation (165 tasks). Despite simplicity, best-of-N is the best parallel method overall (63.03 reported by a review site vs beam search 56.97 and DVTS 55.76); list-wise verification/merging beats other verifiers; more diverse rollouts (mixing models) help: GPT-4.1 alone 55.76 pass@1, four different models 74.55 pass@4. Numbers are from a review aggregator. [V] for conclusions in the abstract, [S] for numbers. Note: GAIA is web research, not code.
- **General AgentBench** (arXiv 2602.18998, 2026). Counterpoint: neither sequential nor parallel scaling gave effective gains in its general-agent setting, citing a context ceiling (sequential) and a verification gap (parallel). Details not read. [S]
- **SWE-Search** (Antoniades et al., arXiv 2410.20285, ICLR 2025). MCTS with a value agent and a discriminator agent (multi-agent debate): 23% relative improvement averaged over five models vs agents without search. Depth-scaling curves not retrieved. [V]
- **Pairwise/tournament selection.** A 2026 preprint (arXiv 2605.14163) uses a pairwise comparator aggregated with a Copeland rule; ExPairT-LLM (arXiv 2511.10855) selects code via pairwise tournament queries; a survey (arXiv 2512.22256) states the trade-off: scoring is cheap and parallel but needs calibrated absolute scores, comparison avoids calibration at more verifier calls. No matched-cost comparison of knockout vs round-robin vs scoring on SWE tasks was found. The call counts (round-robin n(n-1)/2, knockout n-1) are my arithmetic. [S]/[U]

**Read-across:** the verifier is the bottleneck in every source (R2E-Gym plateaus, Satori's need for combined RM+tests, SWE-HERO's 15% gap, Stroebl's cap). Hybrid verification (execution plus learned/LLM judge) is the one design choice supported by two independent results (R2E-Gym 42-43% to 51%; Satori's RM+UT selection).

### 2.2 Evolutionary and merge approaches

- **FunSearch** (Romera-Paredes et al., Nature 625, 468-475, Dec 2023, doi 10.1038/s41586-023-06924-6). LLM plus systematic evaluator in an island-based evolutionary loop; new cap-set constructions (largest increase in the size of cap sets in about 20 years per DeepMind's blog) and better online bin-packing heuristics. A course note says multi-parent crossover gave "relatively small benefit" over single-parent; I could not confirm that against the paper. [V] for results; [S] for the crossover remark.
- **AlphaEvolve** (Novikov et al., arXiv 2506.13131, June 2025). Evolutionary coding agent with automated evaluators. 48-scalar-multiplication algorithm for 4x4 complex matrices (previous 49). Secondary summary: over 50 open math problems, matched the best known on about 75% and improved about 20%. Its ablations (evolution, prompt context, full-file evolution, strong LLMs each help) are known only from a secondary summary. The "0.7% of Google compute recovered" figure that I was asked to check was not confirmed. [V] for the paper's existence and the 4x4 result; [S] for percentages and ablations; [U] for 0.7%.
- **OpenEvolve** (open source). The author reports 2.634 for the 26-circle packing sum of radii (99.97% of AlphaEvolve's 2.635). A competitor paper (arXiv 2605.19633, optimize_anything) reports OpenEvolve with GPT-5.1 reaching only 2.6307 after 200 iterations (about $6.85) vs its own 2.63598 in 63 evaluations (about $3.18). Both are interested parties; no independent peer-reviewed reproduction found. [S]
- **ShinkaEvolve** (Sakana, arXiv 2509.19349; ICLR 2026). New best-known 26-circle solution with about 150 samples, using novelty-based rejection of near-duplicate programs, parent sampling balancing exploration and exploitation, and bandit selection among LLMs. Ablation numbers not retrieved. [V] headline; [U] ablations.
- **DeltaEvolve** (arXiv 2602.02919). In an AlphaEvolve-style ablation, removing numeric scores from context changed little, but removing the selection policy (random context) made performance collapse. [S]
- **LLM crossover.** Bouras et al. (GI workshop at ICSE 2025): LLM-assisted crossover improved fitness (runtime) by an average 8.5% over the best traditional-crossover variant, with 25.6% fewer variants to reach the same milestone. EvolRepair (arXiv 2604.02134, program repair): pass@1 96.63% full, 94.50% without crossover, 88.11% without mutation; replacing group recombination with pairwise degrades results. LLEGO (arXiv 2503.14217): crossover and mutation together beat either alone. Cross-paper comparisons are weak. [S] for numbers, since I saw them only via summaries.

**Read-across:** every population-based success has a cheap, automatic, numeric fitness function (cap-set size, packing radius, runtime, test pass rate). None addresses selecting among architectures or libraries with no oracle. Merging is therefore evidence-supported only when a re-runnable score exists to catch bad merges.

### 2.3 Autonomous build benchmarks and failure modes

- **Web-Bench** (ByteDance, arXiv 2505.07473, May 2025). 50 projects x 20 sequentially dependent tasks; each project is 4 to 8 senior-engineer hours. Best result (Claude 3.7 Sonnet with their agent) was 25.1% Pass@1, versus 65.4% on SWE-bench Verified. [V]
- **Commit0** (arXiv 2412.01769, 2024). Generate Python libraries from scratch against unit tests: a state-of-the-art LLM without feedback passes 17% of tests on the easier libraries and about 6% across all; with iteration on test feedback 26% on the easier subset. No agent fully reproduced a library. [V] (a secondary site gives 29.30% and 6.12% for different subsets; versions differ).
- **PaperBench** (OpenAI, arXiv 2504.01848, ICML 2025). 20 ICML 2024 papers, 8,316 gradable rubric leaves. Best agent (Claude 3.5 Sonnet New, open scaffold) 21.0%; o1 with an iterative agent 24.4% at 12 h and 26.0% at 36 h (secondary); the human baseline of ML PhDs scored 41.4% on a 3-paper subset in 48 h (sources disagree whether best-of-three or mean). Best LLM judge F1 about 0.83. [V] for 21.0% and design; [S] for the rest.
- **ProjectEval** (Findings of ACL 2025, arXiv 2503.07010). Three input levels (prompt, checklist, skeleton); execution tests that simulate user interaction. GPT-4o about 15% Pass@5 on complex projects (secondary summary of the abstract). [V]/[S]
- **DevBench** (arXiv 2403.08604; published ICML). 22 repositories, 4 languages; stages: design, environment setup, implementation, acceptance testing, unit testing. GPT-4-Turbo and others fail the challenges; weaknesses: repository structure, compilation management, advanced concepts. Per-stage numbers not retrieved. [V]
- **MetaGPT / ChatDev.** SoftwareDev (70 tasks) executability on a 0-4 human-rated scale: ChatDev 2.25, MetaGPT 3.75 (3.67 without executable feedback); the authors' own numbers. The independent E2EDev (arXiv 2510.14509) reports requirement accuracy near zero for MetaGPT with most models and 0.18-0.45 for ChatDev. Different metric from SoftwareDev, so not a refutation, but self-reported scores should not be quoted as success rates. [V]/[S]
- **MAST** (Cemri et al., arXiv 2503.13657). 14 failure modes in 3 groups (system design issues, inter-agent misalignment, task verification), built from 150 annotated traces, dataset of 1600+ traces across 7 frameworks, human kappa 0.88 in the latest version. Listed modes include disobeying task specification, step repetition, loss of conversation history, failing to ask for clarification, premature termination, missing or incorrect verification. Conclusion: failures are mostly design and coordination, not model limits. [V]
- **Spec/goal drift.** Wink (arXiv 2602.17037) classes real IDE-agent misbehavior into specification drift, reasoning problems and tool-call failures (no rates retrieved); "Evaluating Goal Drift" (arXiv 2505.02709) and a constraint-drift paper (arXiv 2605.10481) show agents widening scope; the Spec Growth Engine (arXiv 2606.27045) proposes a drift gate as a blocking merge condition (a design proposal, not a measurement). No benchmark counting unrequested features or untraced code was found. [V] for existence; [U] for any rate.

### 2.4 Elicitation and clarifying questions

- **Ambig-SWE** (Vijayvargiya et al., CMU, arXiv 2502.13069; ICLR 2026). Underspecified variant of SWE-bench Verified. Models cannot reliably tell well-specified from underspecified instructions, and without strong prompting rarely ask. When they do interact, performance improves by up to 74% over the non-interactive setting (relative; per-model tables not retrieved). A blog reports 37.94% vs 59.52% resolve for Claude Sonnet 3.5 without vs with questions. [V] abstract; [S] blog numbers.
- **Ask or Assume?** (Edwards and Schuster, arXiv 2603.26233, 2026). Same setup; a scaffold that decouples underspecification detection from execution reaches 69.40% resolve, closing the gap to fully specified inputs. [V]/[S]
- **ClarifyGPT** (FSE 2024, doi 10.1145/3660810, arXiv 2310.10996). Ask only when a code-consistency check detects ambiguity. GPT-4 Pass@1 70.96% to 80.80% on MBPP-sanitized with 10 human participants; simulated-user averages across benchmarks 62.43% to 69.60% (GPT-4) and 54.32% to 62.37% (ChatGPT) in v1 (numbers differ between versions). [V]
- **ClarifyCoder** (arXiv 2504.16331): fine-tuning raised communication rate and good-question rate by 40 and 30 points absolute. It cites prior work that code LLMs produce code in over 63% of ambiguous cases without asking. [V]
- **ClarifyCodeBench** (arXiv 2607.00711, July 2026): nearly all models ask fewer questions than annotated as required; enabling thinking does not increase asking; asking many questions does not imply asking the right ones. Pass@1 drops under ambiguity (blog numbers GPT-4o 35.0 to 27.2). [V] qualitative; [S] numbers.
- **LLMREI** (Korn, Gorsch, Vogelsang; RE 2025; arXiv 2507.02564). Chatbot interviewer, 33 simulated stakeholder interviews: error counts comparable to human interviewers, extracts a large portion of requirements. [V] (no exact percentages retrieved).
- **ReqElicitGym** (arXiv 2602.18306, preprint): across seven LLMs, less than half of implicit requirements are uncovered, and effective questions tend to come in later turns. [V]
- **Optimal number of questions or turns:** no study found measuring diminishing returns. [U]

### 2.5 Spec-driven development and acceptance tests

- **SDD empirical evidence.** (a) Rosa et al., SANER 2026 registered report (arXiv 2601.03878): design only, human-in-the-loop spec, tests, function stages with CURRANTE on LiveCodeBench; no results. (b) Panda (arXiv 2606.30689, June 2026): traceSDD vs Spec Kit vs OpenSpec on Claude Sonnet 4.6 and GLM-5-turbo; forcing citations lowers output determinism but enables automated detection of spec deviations. Measures fidelity, not whether SDD beats no-SDD. [V]
- No study isolating "spec quality to correctness" for autonomous agents, and none for Kiro or BMAD. [U] ProjectEval's three input levels are the nearest controlled variation of spec richness, but I did not retrieve its per-level results.
- **Acceptance test generation.** An AST 2025 industrial case study (arXiv 2504.07244): user stories to Gherkin to Cypress with GPT-4 Turbo; 95% of scenarios judged helpful; 60% of scripts usable as generated, 8% minor fixes, 24% regenerate, 8% discarded. Earlier work: 1 syntax error in 50 generated feature files. [V] No study of test-first versus test-after ordering for LLM implementation found. [U]

### 2.6 Human feedback loops

- RECODE-H (ICLR 2026, arXiv 2510.06186): richer simulated feedback gives substantial gains on research code. [V]/[S]
- ProSoftArena (arXiv 2601.02399): human takeover raised a weaker model's L2 success from 6.7% to 66.7%; agent-initiated help was rare. [S]
- SWE-Interact: best models solve about 50% of single-turn tasks but about 25% of multi-turn revealed-requirement versions. Primary paper not read. [S]
- Field data on agentic PRs: 79.1% of merged agentic PRs had no observed feedback loop; another study reports users push back in 44% of turns. [S]
- Nothing found showing that feedback from an earlier accepted/rejected round improves later autonomous rounds (the "DoD lessons" mechanism). [U] That mechanism has no empirical support yet; it is a hypothesis to measure.

## 3. Corrections to wave-one docs

1. Doc 06 A1: "SWE-HERO: with K=32, Best@32 of 64.6 vs Pass@32 of 79.8, showing selection quality is the bottleneck." Not confirmed. Searches for those figures returned nothing. The paper (arXiv 2604.01496) is an SFT recipe whose headline number is 62.2% (32B). A secondary summary gives verifier gains of 7.9/5.2/4.5 points and a verifier-to-oracle gap up to about 15% at K=16. Replace the figures with these (marked [S]) or drop them.
2. Doc 06 A1: "Satori-SWE: Best@50 of 41.6 matched a baseline needing Best@500 (~10x sampling)." Number is right, but omitted that the compared baseline is a different, larger model (Llama3-SWE-RL-70B) and that Satori-SWE's gain comes from RL-trained iterative refinement, not from smarter scheduling. Doc 06 A6 then uses it to "support spending effort on smarter allocation over raw N". That inference does not follow.
3. Doc 06 A1: "R2E-Gym ... more test-agent rollouts can be more compute-efficient than more editing rollouts." Not confirmed. What was confirmed: each verifier type alone plateaus around 42-43% and the hybrid reaches 51%.
4. Doc 06 A1 CWM: "generates k candidate solutions and 40 generated unit tests in parallel loops." The 53.9% to 65.8% figures are confirmed; the "40 unit tests" detail was not.
5. Doc 06 A5: "Merging is not demonstrated in the sources found; treat as UNVERIFIED for quality." Too strong: LLM crossover has measured gains in GI (8.5% better fitness, 25.6% fewer variants) and program repair (EvolRepair ablation), and AlphaEvolve/FunSearch/ShinkaEvolve are population-based. The narrower claim that is still true: no evidence for merging without a numeric oracle.
6. Doc 06 B3: "No strong primary source found for agent-led elicitation strategy (UNVERIFIED as research)". Wrong; see 2.4. Also "Open questions: Empirical failure rates ... were not researched" is now partly answered (2.3).
7. Doc 06 summary: "Hybrid selection (execution tests plus learned/LLM verifier) beats either alone." Supported by R2E-Gym and Satori. Keep, and cite.
8. BUILD_PLAN section 2 Phase 6 ("successive halving") and doc 06 A6 ("Satori-SWE's 10x efficiency (A1) supports spending effort on smarter allocation over raw N"): no source supports a halving schedule for code agents. Snell et al. support adaptive allocation, on math. Mark halving as a design choice.
9. Doc 06 A2 says the 22.9% to 64.3% AlpacaEval and 65% order-swap figures are unverified; they remain so. This wave did not re-examine them.
10. BUILD_PLAN section 4 item 7 lists "evidence on autonomous-build failure rates" as unresearched. Update with section 2.3.

## 4. Implications and concrete changes

### 4.1 To doc 06

- Add to A1 the Stroebl et al. result and a rule: **cap N at the point where verifier false positives dominate.** With tests that were themselves generated by the system (as in the Phase 7 plan), assume weak distinguishability and keep N small.
- Rewrite A5: "Merge only when a re-runnable numeric fitness exists (benchmarks, tests, perf); otherwise use winner-plus-adopt sub-goal." Cite Bouras/EvolRepair as [S].
- Add a subsection "Evolutionary mode": for goals with a numeric metric (perf tuning, compression, heuristics), allow an AlphaEvolve-style loop (population, islands, novelty rejection) instead of a single N-way fan-out. Cite ShinkaEvolve's roughly 150-sample figure as evidence that sample-efficient designs exist, but flag that OpenEvolve vs competitor numbers are from interested parties.
- Rewrite B3 with Ambig-SWE, ClarifyGPT and ReqElicitGym; specifically add: (a) the factory must run its own ambiguity check (code-consistency or multi-sample disagreement, as in ClarifyGPT) instead of trusting the model to notice; (b) force a question step, since models under-ask even when told they can; (c) prefer targeted questions (the same papers show many questions are not useful).
- Add a "benchmarks calibration" paragraph from 2.3: public greenfield success rates are low (Web-Bench 25.1%, Commit0 about 6%, PaperBench 21.0%) and old-model. Use as a reason for walking-skeleton-first milestones and bounded repair loops, not as a forecast.
- Replace doc 06's item 9 ("Max ~5 questions per turn") label with an explicit **[guess]** tag. Same for the "max 3 automatic rounds" rule in item 10.
- State that the "DoD lessons" loop is a hypothesis with no direct evidence; track its effect (see section 6).

### 4.2 To BUILD_PLAN.md

- Phase 6 exit criteria: add "measure selector accuracy on a planted set (known-better option) and report false-positive rate of gates"; the Stroebl result says this, not N, sets the ceiling.
- Phase 6 scoring: keep "gates, Pareto, judged"; add a hybrid verification requirement (execution plus an LLM judge) citing R2E-Gym/Satori. Keep order-swapped pairwise comparison; add Copeland-style aggregation for more than 2 candidates as a candidate design [S].
- Phase 6: add an "evolutionary mode" item for numeric-metric goals, deferred until after the basic fan-out works.
- Phase 7: add (a) an ambiguity-detection step before elicitation; (b) spec-drift measurement metric: fraction of diff hunks traceable to a requirement ID (the factory must define and own this metric; no external benchmark exists); (c) record, per build, the first-pass acceptance rate to evaluate the rejection loop.
- Section 4 item 7: rewrite to reflect that autonomous-build failure rates are now sourced, and that Spec Kit/Kiro/BMAD effectiveness has no empirical backing (adopt their file layouts for interop and familiarity, not on evidence of benefit).
- Section 3 open decision 3 ("how many parallel options per compare goal"): propose default 3, max 4-5, justified by host capacity plus Stroebl's plateau, flagged as a guess (see 5).

## 5. Defaults: evidence-backed versus guess

| Default | Value proposed | Status |
|---|---|---|
| N for best-of-N with test-based selection on a goal with a trustworthy oracle | up to 8-10 | Weak evidence. Stroebl: optimal attempts often <10 under imperfect verifiers [V]. Large Language Monkeys shows gains to 250 only when verification is exact [V]. Keep 10 as a ceiling, not a target. |
| N for compare goals with no oracle (architectures, libraries) | 3, max 4 | **Guess.** Home-server capacity argument; no source on diminishing returns for design choices. |
| Hybrid verifier (tests plus LLM judge) | required | **Evidence-backed** (R2E-Gym 42-43% to 51%; Satori RM+UT) at abstract level. |
| Pairwise judging with swapped order | required | Evidence-backed for bias (wave-one cites, not re-checked) plus 2605.14163 design [S]. |
| Successive halving checkpoints (compiles, smoke), drop bottom half | 2 checkpoints, halve each | **Guess.** No source on code agents. Snell supports adaptive allocation by difficulty (math). Treat as a cost heuristic and measure whether halving drops the eventual winner. |
| Tournament shape | knockout for N>4 with cheap comparator; Copeland for N<=4 | **Guess**; the cost arithmetic is mine. |
| Rerun metrics 3x | 3 | **Guess** (standard benchmarking practice, not sourced). |
| Clarifying questions per elicitation | ask only when ambiguity check fires; up to 5 per turn; stop when no new requirement | **Guess.** The only evidence-backed parts: models under-ask [V]; detection must be explicit; interaction helps (up to 74% relative, Ambig-SWE) [V]. No optimal count found. |
| Auto rounds on rejection before escalation | 3 | **Guess.** |
| Walking skeleton first | yes | Supported by Web-Bench's sequential-dependency design (error compounding across dependent tasks), but the causal claim is my inference. [U] |
| Separate author for acceptance tests | yes | Design inference from MAST "incorrect verification" and self-consistent-error reports; no controlled test found. [U] |

## 6. Remaining unverified items

- Full text of all papers cited (no paper body read; access blocked). Priority re-checks: CodeMonkeys selection-vs-coverage numbers and cost split; R2E-Gym verifier-scaling figures; SWE-HERO Best@K table (the 64.6/79.8 claim); Ambig-SWE per-model table; Stroebl's sample-count curves; AlphaEvolve ablation table and the 0.7% compute-recovery claim.
- Matched-cost comparison of knockout, round-robin and score-based selection for code candidates.
- Any evidence that human accept/reject feedback from earlier rounds improves later autonomous rounds; recommend instrumenting Phase 7 to test it.
- Controlled evidence for Spec Kit, Kiro or BMAD; also EARS, PRD and ADR primary sources (still not fetched).
- Benchmarks for scope creep or unrequested features by coding agents.
- Number of clarifying questions at which returns diminish.
- Judge-panel size for non-code criteria (unchanged from wave one).
- Newer (2026) greenfield or long-horizon app-build results with current models; all numbers here are 2024-mid-2025 models except the 2026 papers noted.

## 7. Sources

Retrieved via search-tool summaries only (see section 0).

- Large Language Monkeys, arXiv 2407.21787
- CodeMonkeys, arXiv 2501.14723; https://scalingintelligence.stanford.edu/blogs/codemonkeys
- CWM, arXiv 2510.02387; https://ai.meta.com/research/publications/cwm/
- Satori-SWE, arXiv 2505.23604
- R2E-Gym, arXiv 2504.07164; https://neurips.cc/virtual/2025/131676
- SWE-ZERO to SWE-HERO, arXiv 2604.01496
- Limits of Inference Scaling Through Resampling, arXiv 2411.17501 (ICLR 2026)
- ROC-n-reroll / verifier imperfection, arXiv 2507.12399
- Snell et al., arXiv 2408.03314 (ICLR 2025)
- Scaling Test-time Compute for LLM Agents, arXiv 2506.12928
- Benchmark Test-Time Scaling of General LLM Agents, arXiv 2602.18998
- SWE-Search, arXiv 2410.20285 (ICLR 2025)
- Pairwise selection: arXiv 2605.14163, arXiv 2511.10855, survey arXiv 2512.22256
- FunSearch, Nature 625:468-475, doi 10.1038/s41586-023-06924-6
- AlphaEvolve, arXiv 2506.13131
- OpenEvolve comparisons: arXiv 2605.19633 (optimize_anything), arXiv 2510.26144 (FM Agent), arXiv 2510.14150 (CodeEvolve)
- ShinkaEvolve, arXiv 2509.19349 (ICLR 2026)
- DeltaEvolve, arXiv 2602.02919; EvolRepair, arXiv 2604.02134; LLEGO, arXiv 2503.14217; Bouras et al., GI@ICSE 2025
- Web-Bench, arXiv 2505.07473
- Commit0, arXiv 2412.01769
- PaperBench, arXiv 2504.01848 (ICML 2025, PMLR 267)
- ProjectEval, arXiv 2503.07010 (Findings of ACL 2025)
- DevBench, arXiv 2403.08604
- MetaGPT, arXiv 2308.00352; ChatDev, arXiv 2307.07924; E2EDev, arXiv 2510.14509
- MAST, arXiv 2503.13657
- Wink, arXiv 2602.17037; Goal drift, arXiv 2505.02709; Constraint drift, arXiv 2605.10481; Spec Growth Engine, arXiv 2606.27045
- Ambig-SWE, arXiv 2502.13069 (ICLR 2026); Ask or Assume?, arXiv 2603.26233
- ClarifyGPT, arXiv 2310.10996 (FSE 2024); ClarifyCoder, arXiv 2504.16331; ClarifyCodeBench, arXiv 2607.00711
- LLMREI, arXiv 2507.02564 (RE 2025); ReqElicitGym, arXiv 2602.18306
- Rosa et al., arXiv 2601.03878 (SANER 2026 registered report); Panda, arXiv 2606.30689
- Acceptance test generation industrial case study, arXiv 2504.07244 (AST 2025); BDD acceptance tests, arXiv 2403.14965
- RECODE-H, arXiv 2510.06186; ProSoftArena, arXiv 2601.02399; HULA, arXiv 2411.12924
