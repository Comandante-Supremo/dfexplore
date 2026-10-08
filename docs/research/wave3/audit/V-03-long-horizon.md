# V-03: Verification of R3-04 (long-horizon agent papers)

Verifier pass 2026-10-08, run on a different model from the readers. Method: I downloaded each PDF from `arxiv.org/pdf/<id>` (MAST at v1, v2 and v3 separately) into a new scratch directory, extracted the text with `pdftotext -layout`, and read the passages myself. I shallow-cloned `OpenHands/software-agent-sdk` from github.com (HEAD 69e2688, 2026-10-07). For the other ids R3-04 cites, I checked titles and dates on their arXiv abstract pages. I did not reach metr.org or any other blocked host, and I did not route around blocks.

## Verdict: TRUST WITH CORRECTIONS

R3-04 settles the MAST conflict correctly at its core. 15.7 / 12.4 / 6.2 / 8.2 / 23.5 are the final (v3) figures. 11.5 / 6.54 / 8.64 / 9.16 / 31.41 are the 151-trace v1 figures. Every headline number I checked reproduces from the PDFs. Every arXiv id resolves to the paper R3-04 names.

Corrections:

1. **MAST has three numeric versions, not two.** v2 (22 Apr 2025, "200+ traces") has its own set: step repetition 17.14, unaware of termination 9.82, premature termination 7.82, no/incomplete verification 6.82, verification category 21.30. This is the "third site" set that A-05 saw (41.77 / 36.94 / 21.30). R3-04 does not give these values.
2. **What the MAST percentages mean.** They are shares of all labelled failure occurrences, and they sum to 100 across the 14 modes. They are not per-trace prevalence. R3-04 calls them "prevalence of each mode across traces (multi-label)". v2 says it outright: FM-3.2 + FM-3.3 are "13.48% of all observed failures". In v3, 44.2 + 32.3 + 23.5 = 100. W2-05's "verification about a quarter of failures" is therefore the correct reading.
3. **Valmeekam false positives.** The LLM verifier passed 38 of 45 invalid plans (false-positive rate 84.45%). R3-04 says 7 of 45, but 7/45 is the true-negative count.
4. **SLA Table 4 example columns are mixed up.** For libexpat over 70M to 170M tokens, the checkpoint is 16.12, full SLA is 18.78 and "w/o Advisor reconstruction" is 16.86. The 19.07 figure belongs to the 140M to 240M column, where the checkpoint is 17.69 and full SLA is 20.13. The direction R3-04 reports is correct: every ablation is worse in all four columns.
5. **SWE-Search cost.** The "about 14x" multiplier holds only for GPT-4o. The other four models are about 5x (GPT-4o-mini 5.3x). Three of those costs are marked as estimates.
6. **Minor.**
   - The 151-trace figure is Figure 2 in v1, not Figure 1.
   - In v3 the old figure is embedded inside the Figure 2 workflow schematic. In v2 it sits inside Figure 3.
   - LoopsBench: the Codex, Claude Code goal and Ralph runs all have 102 scheduled tasks. Only the dynamic-workflow run has 112.

## Task 1: MAST version table (arXiv 2503.13657)

Version history from the abstract page: v1 17 Mar 2025; v2 22 Apr 2025; v3 26 Oct 2025, which is the latest (v4 returns 404). The unversioned PDF is byte-identical to v3. Only v3 carries the "NeurIPS 2025 Track on Datasets and Benchmarks" footer.

| Quantity | v1 (Fig. 2, "151 traces") | v2 (Fig. 2, "200+ traces") | v3 (Fig. 1 + Sec. 4 text, "1642 traces") |
|---|---|---|---|
| FM-1.3 step repetition | 11.5% | 17.14% | **15.7%** |
| FM-1.5 unaware of termination | 6.54% | 9.82% | **12.4%** |
| FM-3.1 premature termination | 8.64% | 7.82% | **6.20%** |
| FM-3.2 no/incomplete verification | 9.16% | 6.82% | **8.20%** |
| FM-3.3 incorrect verification | 13.61% | 6.66% | **9.10%** |
| FC3 task verification (category) | 31.41% | 21.30% | **23.5%** |
| FC1 / FC2 | 37.17 / 31.41 | 41.77 / 36.94 | 44.2 / 32.3 |
| Premature (3.1) vs unaware (1.5) | premature higher | premature lower | premature lower |
| Annotation | expert GT stage | expert + o1 judge (kappa 0.77) | mostly o1 judge (kappa 0.77; human IAA 0.88) |

In every version the categories sum to about 100%, so the figures are shares of failure labels.

**Who quoted which version:**
- **W2-05** quoted v3 (15.7 / 12.4 / 6.2 / 8.2 / 23.5) through a secondary write-up. Its numbers are correct for v3.
- **A-05** quoted the MAST repo README figure (`taxonomy_v11_cropped-1.png`), which is the v1 151-trace figure. That figure also appears unchanged as a sub-panel in v2 Fig. 3 and v3 Fig. 2, with no trace count. A-05's "third site" (41.77 / 36.94 / 21.30) is v2.
- **BUILD_PLAN section 6** rests on the v1 figure.
- **R3-04** is correct on v3 versus v1 but leaves out v2.

**Figures to use:** v3 only. The FM figures are 15.7 / 12.4 / 6.2 / 8.2 / 9.1, and the FC3 category is 23.5. Each should carry this wording: "share of failure labels over 1,642 traces from 7 multi-agent frameworks (2024-25), mostly o1-annotated; the mix varies by system (AppWorld is heavy on premature termination); the ordering of 3.1 versus 1.5 flipped between v1 and v2/v3". Cite 23.5 only as the category total, never as FM-3.2. The v3 text gives FM-2.4 as 0.85% while the figure gives 0.80%. That is immaterial.

## Task 2: claims table

| Claim | Reader said (R3-04) | Paper/source says | Rating |
|---|---|---|---|
| METR 80% vs 50% horizon ratio | 4-6x shorter; intro says "roughly 5x" | 2503.14499v4 (10 Jul 2026), Sec. 3.2.1: "models' 80% time horizons are 4-6x shorter"; intro: "roughly 5x shorter" (Fig. 17); 80% doubling 204 days | REPRODUCED |
| METR doubling time | 207 days (CI 166-240), about 7 months | Sec. 3.2: "doubled every 207 days with a 95% bootstrapped confidence interval 166-240 days"; abstract: about 7 months, o3 50% horizon about 110 min | REPRODUCED |
| METR failure table | repeating failed actions 12/31 (GPT-4 1106) vs 2/32 (o1); premature abandonment 8 vs 16 | Table 2 matches | REPRODUCED |
| Huang et al. 2310.01798 scope | ICLR 2024; intrinsic self-correction lowers accuracy; GSM8K / CSQA / HotpotQA; GPT-3.5, GPT-4, GPT-4-Turbo, Llama-2-70B; at most 2 rounds; not coding | v2 file, "Published as a conference paper at ICLR 2024"; datasets and models as stated; GPT-4 GSM8K 95.5 -> 91.5 -> 89.0; maximum two rounds; the paper itself says code-executor feedback can help | REPRODUCED |
| Valmeekam 2310.08118 | 55% LLM+LLM vs 88% LLM+VAL (40% with no backprompting); verifier 61% accurate; "7 of 45 incorrect plans passed" | Table 1: 55/100, 88/100, 40/100. Table 2: 61/100 accuracy; **38/45 invalid plans judged valid (FPR 84.45%)**; 7/45 is the true-negative count | WITH-CORRECTION (55 vs 88 holds; the false-positive count is wrong) |
| SWE-Marathon (arXiv **2606.07682**, v1 5 Jun 2026, preprint) | claude-code 41.9% -> 3.2% with run length; 877 identical consecutive calls; 32% duplicate calls (terminus-2); Table 4: timeout 31.4, premature 7.6, poor self-verification 4.0; 0/71 compaction | Sec. 5.2 matches verbatim. The 32% is non-consecutive "repeat an earlier (function, arguments) pair"; verbatim consecutive retries are only 1.3%. Table 4, n=526, matches. "0 of 71 ... against 8.9% without" matches. Correlational; run-length bins are undefined | REPRODUCED (note: the 32% is mostly non-adjacent duplication) |
| SWE-Search 2410.20285 | +23% relative mean; about 14x cost (GPT-4o) | Table 1 mean +23% (GPT-4o 25.7 -> 31.0, +17). Table 3: GPT-4o $40.86 -> $576.00 (14.1x); GPT-4o-mini 5.3x; the other three about 5x (marked as estimated costs). Table 4: compute-matched gaps are small | WITH-CORRECTION (14x is GPT-4o only; others about 5x) |
| ACE 2510.04618 +17.0 on AppWorld | ReAct 42.4 -> 59.4 (+17.0) offline with ground-truth labels; abstract "+10.6% on agents"; ICLR 2026 | v3 (29 Mar 2026), ICLR 2026 header. Table: ReAct+ACE with GT labels 59.4 (+17.0); without labels 57.2 (+14.8); 59.5 (+17.1) for the second ACE row; DeepSeek-V3.1 | REPRODUCED (+17.0 is percentage points, offline, with labels; the headline figure is +10.6%) |
| OpenHands SDK stuck defaults | `action_observation=4`, `action_error=3` (nudge at 3, stuck on 4th), `monologue=3`, `alternating_pattern=6` (6 actions + 6 observations = 3 A/B cycles); events since last user message | Clone at 69e2688: `types.py` `StuckDetectionThresholds` defaults are 4/3/3/6 (`ge=1`, configurable). `stuck_detector.py`: action-error is stuck when streak `> threshold`; the nudge fires once at streak `== threshold`; the alternating check compares `[i]` with `[i+2]` over 6 actions and 6 observations; scanning starts after the last user message; the context-window loop check needs >= 10 events. Note: the alternating check does not require A != B | REPRODUCED |
| OpenHands paper 2407.16741 has no stuck-detection design | NOT-IN-PAPER | Title and id match; I did not re-grep the PDF | Accepted, not re-tested |
| LoopsBench 2608.00267 Ralph | 7.84% resolve rate, 13.24 rounds, 0.17 regressions per run; uncontrolled | v2 (10 Aug 2026), Table 4 matches. Appendix N.3: "not a model controlled causal comparison". Ralph run id `stage4-A1-opus47`. Codex, Claude Code goal and Ralph runs each have 102 scheduled tasks; dynamic has 112 | REPRODUCED (minor: the task-count wording) |
| Stateless Language Agents 2610.07625 | v1 6 Oct 2026, Stanford/SambaNova; fresh role-specific context; Table 4 ablations worse in all four comparisons; 1B run without Worker isolation 1336 vs 1112.0; CORAL 98% of final 1,500 sessions had no tool calls after 231M | Title, authors, date and all of those numbers match. The libexpat example is miscited: 16.12 -> 18.78 (full) vs 16.86 (w/o Advisor), and the 19.07 is from the 17.69 -> 20.13 column. The 1336 run is a single run | WITH-CORRECTION (example values only) |
| BRIDGE 2602.07267 (supports the 30-60 min horizon edit) | "80% horizon under 1 hour" | v2 (2 Jul 2026): at 80%, frontier horizons are METR about 54 min, SWE-bench 44 min, MLE-bench 1.1 h, GDPval 20 min; about 40 min overall, "roughly 1/3 of the about 2 hours horizon at 50%" | REPRODUCED |
| METR 2026 horizon figures (Opus 4.5 80% about 27 min, Frontier Risk Report about 1.5 h) | [SNIPPET] | metr.org not reachable | UNREACHABLE |
| MAST (Task 1) | v3 15.7 / 12.4 / 6.2 / 8.2 / 23.5; v1 11.5 / 6.54 / 8.64 / 31.41 | As in the table above, plus v2's own numbers | WITH-CORRECTION (v2 omitted; the percentages are shares of failures, not trace prevalence) |

## ID integrity

I found no fabricated or mismatched arXiv ids. Every id resolves to the title R3-04 gives:
- Fully read: 2503.13657, 2503.14499, 2310.01798, 2310.08118, 2606.07682, 2410.20285, 2510.04618, 2608.00267, 2610.07625, 2602.07267.
- Title and date checked on the abstract page: 2407.16741, 2505.05115, 2601.11868, 2608.01964, 2610.02163, 2509.09677, 2007.11112, 2604.07988, 2602.14849, 2608.03836, 2603.24755, 2310.12397, 2406.01297, 2608.08950 (title "Independent Patch Verification ... Bidirectional Reconstruct-and-Verify"; the abstract names it RETRACE), 2608.23564, 2606.26300, 2407.01476, 2407.21787, 2310.08560, 2502.12110.

Version and date stamps match R3-04 where it states them: METR v4 10 Jul 2026, ACE v3 29 Mar 2026, LoopsBench v2 10 Aug 2026, SLA v1 6 Oct 2026, SWE-Marathon v1 5 Jun 2026, Huang v2 14 Mar 2024.

## Task 3: R3-04 "numbers to change", for BUILD_PLAN

### Safe to apply

1. **Stall defaults (BUILD_PLAN line 86), relabelled.** The values are confirmed from source: `StuckDetectionThresholds` 4 / 3 (nudge at 3, stuck on the 4th) / monologue 3 / alternating 6 actions (= 3 A/B cycles). Label them "OpenHands SDK shipped defaults, uncalibrated, configurable". The semantics BUILD_PLAN already states are correct.
   - Adding a duplicate-call metric is also safe: track the maximum run of consecutive identical calls and the fraction of non-adjacent duplicate (tool, args) pairs. The evidence is SWE-Marathon: 877 consecutive identical calls, 32% duplicates on terminus-2, 4% on claude-code, and pass rate falling with run length.
   - State that a consecutive-repeat detector will miss most of that 32%, because verbatim retries were only 1.3%.
   - Treat the polling allowlist (issue #762) as a design choice. Its source is only a [SNIPPET].
2. **Horizon budget, starting at 30-60 min (line 88).** This is safe as planner guidance labelled as a guess.
   - METR's 80% horizon is 4-6x shorter than its 50% horizon.
   - BRIDGE puts the frontier 80% horizon at about 40-54 min on METR and SWE-bench tasks.
   - Keep "1-2 h is the upper end", "calibrate on logged durations", and not an exit criterion (A-05).
   - Do not cite the METR 2026 [SNIPPET] figures as verified.
3. **Reverse the section 6 rejection of W2-05's MAST correction, reworded.** The rejection's stated basis ("MAST's own figures give 8.64 > 6.54"; "15.7/12.4/23.5 were wrong") is factually wrong for the current paper. Replace it with the following:

   > MAST v3 (1,642 traces, NeurIPS 2025 D&B): step repetition 15.7%, unaware of termination 12.4%, premature termination 6.2%, no/incomplete verification 8.2%, Task Verification category 23.5% (shares of failure labels). The 11.5 / 6.54 / 8.64 / 31.41 figures are the 151-trace v1 figure. The 3.1-vs-1.5 ordering flipped between v1 and v2/v3 and varies by system. Evidence is multi-agent frameworks only, so both detectors stay.

   SWE-Marathon (2026, single-agent CLIs) is consistent with this: timeout 31.4% vs premature termination 7.6% of 526 failures.
4. **R3-04 edit 4 (Ralph wording), edit 6 (verifier evidence note) and edit 7 (line 136 status).** These are safe, with one fix to edit 6: Valmeekam's verifier passed 38 of 45 invalid plans, not 7.

### Do not apply as written

- **R3-04 edit 1 wording "premature 'done' is rarer than non-termination".** Do not state this as a general fact, or in a way that would remove premature-done safeguards. The data are label shares from multi-agent frameworks, and the ordering is unstable across versions. Use the reworded version above.
- **Any use of "23.5%" as no/incomplete verification**, or as "% of traces". FM-3.2 is 8.2%, and the figures are shares of failure labels.
- **R3-04 edit 8 "budget about 14x cost" for tree search.** Use "about 5-14x (14x on GPT-4o; some costs estimated)" and note that compute-matched resampling closes most of the gap. Keep it deferred (Phase 5b).
- **METR 2026 figures** (Opus 4.5 80% about 27 min, 50% about 12 h, Frontier Risk Report): UNREACHABLE. Do not put them in BUILD_PLAN as numbers.
- **SLA's libexpat example values**: fix them (see the claims table) before quoting.
- **Edit 5 (durable execution)** is acceptable only as "per LogAct (preprint)". Temporal, Restate and DBOS docs remain unread.
