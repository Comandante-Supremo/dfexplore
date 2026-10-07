# W2-03 - Dependency-update literature: breaking changes, bots, LLM repair, detection, cooldowns

Research date: 2026-10-07. Scope: verify and deepen docs/research/05-dependency-updates-and-constraints.md and the dependency parts of docs/BUILD_PLAN.md.

## Evidence-level legend (read this first)

Primary-source access failed (see "Fetch failures"). I could not open any paper body. So **no claim below earns a full [V]**. Tags used:

- **[S]** = Seen in a WebSearch result summary whose cited URL is the primary page (arXiv/ACM/venue page, abstract level). Paper body NOT read. Search summaries are model-generated and can contain mistakes; numbers marked [S] should be re-read in the paper before they become hard-coded defaults.
- **[S2]** = [S] but the number came via a secondary/vendor page, not the paper's own page.
- **[D]** = derived by me (arithmetic or reasoning) from [S] inputs.
- **[U]** = unverified / guess / recommendation with no empirical backing.

Fetch failures (all recorded): `WebFetch` returned `getaddrinfo ENOTFOUND arxiv.org` for arxiv.org/abs for 2401.09906, 2411.05830, 2412.04478, 2407.06249, 2110.07889, 2206.07230 and arxiv.org/html/2510.03480v2. `curl` through the agent proxy: arxiv.org returned `CONNECT 403` (organization egress policy; not retried). curl also failed (no connection) for ar5iv, export.arxiv.org, api.semanticscholar.org, semanticscholar.org, huggingface.co, alphaxiv, dl.acm.org, doi.org, ieeexplore, link.springer.com, zenodo, openreview, researchgate, tudelft, core.ac.uk, openalex, crossref and blog.yossarian.net. Only raw.githubusercontent.com answered (not useful for these papers). All findings therefore come from WebSearch result summaries.

## Summary

1. **Breakage in "safe" releases is common and label-independent.** Across ecosystems, roughly a quarter to a third of non-major upgrades contain an API-level break, but only a minority of those break any given client: ~11.6% of Maven updates break the client [S], ~12% of npm dependents hit a break during non-major updates [S], and Go shows 28.6% of non-major upgrades with breaks but only 3.1% of breaking changes actually used by clients [S]. A semver label is a weak prior, not a gate. Default: treat every update, patch included, as needing a canary.
2. **Passing tests is a weak gate.** Tests caught only 47% (direct) and 35% (transitive) of injected dependency faults, and covered only 58%/20% of dependency calls [S]. LLM-generated differential tests detected 30.3% of BUMP breaks, 95.1% of those crash-type [S2]. Wave one's "the test suite is the authority" needs a surface-diff and contract layer next to it.
3. **LLM repair of breaking updates is a minority-success tool without docs or structure**: 19-27% full-build/test success on BUMP (Java) [S], 104/203 (51%) on DepBench with the best agent [S], but 90.5% on JavaScript when the agent is fed structured library breaking-change records [S]. So: always attempt adapt with a bounded budget, but plan for failure as the base case; pin/constrain fall-through must be cheap and automatic.
4. **Cooldown has anecdotal-incident support, no effectiveness study.** GitHub's Dependabot default is 3 days (changelog 2026-07-14) [S2]; the first adoption study found 64.3% of adopters choose 7 days [S]. Documented incidents were pulled within ~1-5 hours [S2]; the xz case (~5 weeks) is not stopped by any practical cooldown [S2].
5. **Bots: merge rates 59-98% depending on bot and ecosystem; no study isolates auto-merge outcomes.** The factory's auto-merge default has no direct empirical backing.
6. **Upper-bound pinning: no measurement study found.** What exists argues for exact pins (lock) in applications and few caps in libraries; the registry should emit exclusions (`!=bad`) in preference to caps for library repos [D/U].

## Verified findings

### A. Breaking changes and semver by ecosystem

| Ecosystem | Finding | Source | Tag |
|---|---|---|---|
| Maven | 83.4% of 119,879 library upgrades (293,817 clients) comply with semver; compliance increased over time. Original 2014 study: ~1/3 of all releases had a breaking change, same rate for minor and major. Snippet also gave per-level breaking rates 61.8% major / 37.9% minor / 14.6% patch for one of the paper's datasets (dataset ambiguous). | Ochoa et al., "Breaking Bad? Semantic Versioning and Impact of Breaking Changes in Maven Central", EMSE 2021, arXiv 2110.07889 | [S] |
| Maven | 18,415 artifacts, 142,355 direct deps, 71.60% not up to date; **11.58% of updates break the client**; transitive changes a major factor; nearly half of client-impacting breaks are in non-major updates. | Jayasuriya et al., "Understanding Breaking Changes in the Wild", ISSTA 2023, DOI 10.1145/3597926.3598147 | [S] |
| Maven | 14.78% of API changes break compatibility across 317 libraries, 9K releases, 260K clients. | Xavier et al., SANER 2017 | [S] |
| Maven | 67% of artifacts have at least one semver violation in history. | Keshani, Vos, Proksch, ICSME 2023 journal-first | [S] |
| npm | ~12% of dependent packages and 14% of releases hit a breaking change during non-major updates; **44% of manifesting breaks came from minor/patch releases**; clients recovered themselves in about half of cases, mostly by upgrading/downgrading the provider without changing constraint config. | Venturini et al., "I Depended on You and You Broke Me", TOSEM 32(4) 2023 (arXiv 2301.04563) | [S] |
| Go | GoSVI over 124K libraries / 532K clients: **28.6% of non-major upgrades have breaking changes**; semver compliance rose from 63.7% (2018-09) to 92.2% (2023-03); only 3.1% of breaking changes are used by clients but about one third of clients may be affected. API-level (signature) breaks only. | Li et al., ASE 2023, arXiv 2309.02894 | [S] |
| Cargo | cargo-semver-checks scan: >3% of ~14,000 releases had at least one semver violation; 1 in 6 of the top 1000 crates broke semver at least once. No peer-reviewed crates.io patch-vs-minor split found. | FOSDEM 2024 slides (maintainer) | [S2] |
| PyPI | No peer-reviewed PyPI breakage-rate study found. A master's thesis (not peer reviewed) coded 9,081 breaking-change issues from 303 Python repos: 53% of breaking changes were introduced in minor releases; avg >5 weeks between commit and release without detection; PyCoReX LLM agent best F1 0.85 (0.82 without CoT) across 12 LLMs. Separately: 35% of scikit-learn clients affected by default-argument breaking changes vs 0.13% for NumPy (93 DABCs in 3 libraries). | Kennesaw State thesis (digitalcommons.kennesaw.edu/masterstheses/145); Montandon-line default-argument study (ar5iv 2408.05129) | [S] |
| Python configs | 74% of configuration issues stem from insufficient constraints; 183,864 library releases had potential configuration issues; PyEGo infers deps for only 65% of releases. | "Less is More?", ICSE 2024, arXiv 2310.12598 | [S] |
| Reviews | A 2026 SLR on breaking changes in ecosystems (arXiv 2605.24397) confirms non-trivial breakage in minor/patch for Maven and npm. | arXiv 2605.24397 | [S] |

Reading across these: per-update client breakage is about 10-14% (Maven, npm) in ecosystems that mostly follow semver, and a large share (44% to nearly half) of the client-visible breakage is in non-major releases. Compliance rates are rising (Maven, Go), so older numbers overstate current breakage. Python/PyPI and Cargo lack good client-level numbers.

### B. Update bots: acceptance, lag, auto-merge

- Dependabot regular updates: 502,752 PRs, **70.13% merged**, median merge lag 0.18 days; security updates 73.71% (confirmed) / 76.01% (unconfirmed) merged. After adoption, mean technical lag fell from 48.99 to 25.38 days at 90 days; 35.7% of projects reached zero lag. Survey: developers trust notification, question automation. He et al., TSE 2023, arXiv 2206.07230 [S].
- Dependabot-preview security PRs: 2,904 JavaScript projects, 65.42% accepted, most merged within about a day; severity and breaking-change risk not strongly tied to merge time; only 3.2% of manually examined PRs had build breakages; seven non-merge reasons, mostly concurrent manual edits. Alfadel et al., MSR 2021 (DOI 10.1109/MSR52588.2021.00037) [S].
- Dependabot security updates (JS): most merged within days; rejected ones are fixed manually, "often up to several months". Mohayeji et al., MSR 2023 Distinguished Paper (DOI 10.1109/MSR59073.2023.00042); extension EMSE 2025 (tests/CI influence acceptance) [S].
- nf-core (35,411 PRs, single ecosystem, arXiv 2607.10839): overall 91.8% merged; **Renovate 98.46%** (median 0.43 d) vs **Dependabot 72.13%** (0.54 d) vs other bots 51.94% [S].
- A mining study of ~1M PRs / 500 projects: 59.5% of (bot) PRs accepted [S, secondary summary].
- 10.05% of closed Dependabot PRs are followed by manual rework on the same dependency (cited via the He et al. summary; origin paper not identified) [S2].
- **Auto-merge outcomes: no empirical study found.** No data on reverted auto-merges. [gap]
- Cooldown adoption study: Tanaka et al., "An Exploratory Study of Dependabot Cooldown Adoption in Open-Source GitHub Projects", arXiv 2609.16605 (2026-09-15, preprint). Cooldown GA July 2025. Security motive in 83 of 92 adoption events with known motivation; of 251 ecosystems retaining cooldown, 97.2% set a general delay, **64.3% chose 7 days**, per-update-type settings <10% each [S].

### C. LLM / agent repair of breaking updates and API migration

| Benchmark/tool | What it is | Result | Source | Tag |
|---|---|---|---|---|
| BUMP | 571 reproducible breaking updates, 153 Java/Maven projects; 243 (43%) are compilation failures; categories: direct compile error, indirect compile error, Java-version incompatibility, Werror. Docker images, ~250 GB. | benchmark | Reyes et al., SANER 2024, arXiv 2401.09906 | [S] |
| Breaking-Good | Build-log + dependency-tree explanation tool | explains root cause for 70% of the 243 compile-failure updates; helped in user study | SCAM 2024, arXiv 2407.03880 | [S] |
| Fruntke & Krinke | agentic vs recursive zero-shot LLM repair on BUMP | **up to 23% (agentic) vs 19% (zero-shot)** repair by test-suite success | FSE 2025, DOI 10.1145/3729366 | [S] |
| Byam | 5 LLMs (Gemini-2.0 Flash, GPT-4o-mini, o3-mini, Qwen2.5-32B, DeepSeek V3) on BUMP | **27% full-build fix** with richest prompt (buggy line + API diff + error messages); 78% of individual compile errors fixed | arXiv 2505.07522 | [S] |
| BigBag | LLM writes reusable AST transformations fixing all clients of one library update | evaluated on 157 BUMP compile-failure updates (numbers not seen) | arXiv 2606.24446 | [S] |
| BDUpdater ("Break to Adapt") | JS agent: aggregates a library's own breaking-change records into structured lists, then maps to client call sites | **90.5% of breaking updates recovered**, 13 libraries / 84 clients, ~$0.05 per client; 83.8% of libraries keep breaking-change records, quality uneven | FSE 2026, DOI 10.1145/3808116 | [S] |
| DepBench ("Update from Hell") | 203 dependency-upgrade tasks, 5 ecosystems, from bot-opened merged PRs, hidden code-level breakage | best agent config solves **104/203 (51%)**; varies by model/harness/ecosystem | arXiv 2608.30300 | [S] |
| SWE Refactor Bench | 20 whole-repo migrations | 28/520 runs (5.4%) pass all three stages; best score 47/100 | arXiv 2608.23564 | [S] |
| BeyondSWE / DepMigrate | 23 packages with major upgrades | best overall score 46.12 (not DepMigrate-only) | arXiv 2603.03194 | [S] |
| SWE-Chain | chained release-level upgrades (xarray, pytest, urllib3, Flask...) | no rate seen | arXiv 2605.14415 | [S] |
| GitChameleon | version-conditioned Python completion with unit tests | v1: 116 problems, best pass@1 ~35.7%, pass@10 ~42.8%, error feedback +5.4%/+4.7%. v2 (arXiv 2507.12367): 328 problems, baseline success ~48-51% for enterprise models; weakest on semantic/behavioural changes and recent versions | arXiv 2411.05830 / 2507.12367 | [S] (v1 numbers via secondary summaries, GPT-4o figures inconsistent between them) |
| LibEvolutionEval | 8 libraries, version-specific completion; NAACL 2025 | library evolution hurts accuracy; version-specific doc retrieval helps; no numbers seen | arXiv 2412.04478 | [S] |
| CodeUpdateArena | synthetic API updates, 670 tasks, 54 functions, 7 packages (count from a later paper) | prepending update docs in context does not match fine-tuning on update examples | arXiv 2407.06249 | [S] |
| J.P. Morgan Java agents | multi-agent using migration docs, ASEW 2025 workshop (pp. 34-37) | wave one cited 71.4% precision on 3 synthetic repos; I could not re-see that figure | arXiv 2510.03480 | [U] (number not re-verified) |
| PCREQ | infers a compatible requirements.txt for a Python library upgrade; REQBench 2,095 cases | 94.03% inference success vs PyEGo 37.02%, ReadPyE 37.16% | arXiv 2508.02023 | [S] |

Take-aways for design: (a) feeding structured, verified change knowledge (API diff, error messages, the library's own breaking-change records) is the dominant lever; the same model class moves from ~20% to ~90% when it gets that, though the 90% setting is JS and measures recovery of documented breaks. (b) Build-level success is far lower than error-level success (27% vs 78% in Byam), so a partial patch is common: the factory must accept "partially adapted" as a failed adapt and not merge. (c) Cost per attempt is cents ($0.05/client BDUpdater [S]; BreakGuard cost reported as ~$0.09 per instance in one summary and ~$0.90 per detected change in another, unresolved [S2]); budget is not the constraint, correctness is.

### D. Detecting undocumented breaking changes

- **Test-based**: Hejderup & Gousios, JSS 183 (2022), DOI 10.1016/j.jss.2021.111097, arXiv 2109.11921: 521 Java projects; tests cover 58% of direct and 20% of transitive dependency calls; 1,122,420 artificial faulty updates across 262 projects; tests detect 47% direct / 35% transitive faults; their call-graph change-impact approach 74%/64%; on 22 real Dependabot updates it found 3 semantic conflicts and 5 unused dependencies [S]. The faults are synthetic, so this is an upper-ish bound on real-fault detection for those test suites, not a measurement of real breaking updates.
- **LLM test generation**: BreakGuard (arXiv 2608.20167): generates tests per client-library interaction, runs on old and new library, on BUMP. Best context (full class code) detected **30.3%** of breaking changes; ~95.1% of detections were crash-type (missing class/method); behavioural changes largely undetected [S2 for the 95.1%]. DiffTestGen (arXiv 2607.16024) exposes behavioural differences in 78.2% of PRs but targets PR diffs, not dependency upgrades [S].
- **API/type diff**: AexPy (Du & Ma, ISSRE 2022, DOI 10.1109/ISSRE55969.2022.00052) detects declared and undeclared breaking changes in Python packages [S]; griffe `check` for API regressions [S2]; cargo-semver-checks for Rust (above); GoSVI for Go; "Has My Release Disobeyed Semantic Versioning?" (ASE 2022, arXiv 2209.00393) for Java via semantic differencing [S]. Static API diff catches signature/removal breaks, not behavioural or default-value changes.
- **Snapshot/trace**: GILESI records call traces during client tests and compares across versions (per the 2026 SLR) [S2].

Conclusion: signature breaks are detectable cheaply and deterministically (API diff of what the repo consumes); behavioural breaks need the repo's own tests plus value-level differential assertions. Neither alone is adequate; both together still leave an unmeasured residual.

### E. Release notes / changelog quality

- Wu et al., ICPC 2022, arXiv 2203.15592: 1,731 GitHub issues about release notes; 48.47% about production, content 25.61%, accessibility 17.65%, presentation 8.27%; producers omit information more than they err, **especially for breaking changes** [S].
- Bi et al., release note production and usage: 32,425 release notes from 1,000 GitHub projects, 15 interviews, 314 survey responses; stakeholders disagree on what notes should contain [S].
- Nath et al., arXiv 2511.18187 (2025): 47% of release artifacts lack traceability links, 12% have broken links [S].
- BDUpdater (above): 83.8% of JS libraries keep breaking-change records, uneven quality (implicit references, vague adaptation instructions) [S].
- Python deprecation: poor documentation and vague deprecation notices hinder users (Wang et al., per summary) [S2].
- No benchmark of changelog-only breaking-change detection found (wave one's gap stands).

### F. Pinning and upper bounds

- Dietrich et al., "Dependency Versioning in the Wild", MSR 2019: framing only (fixed = predictable runtime; ranges = flexibility for fixes); I did not see its measurements [S, abstract only].
- Recovery in practice: npm clients that recovered from a break did so mostly by changing the resolved provider version, not the constraint (Venturini) [S].
- "Less is More?" (ICSE 2024): 74% of PyPI configuration issues come from insufficient (too open) constraints; recommends complete and strict constraints validated at release time [S]. This concerns a library's declared metadata being wrong, not the effect of upper caps on downstream solvability.
- LooCo (FSE): automatically loosening mal-configured constraints made 54.8% of unsolvable cases solvable [S2].
- Poetry FAQ and Python discourse: caps stop downstream users from updating even when nothing breaks; suggested for applications, avoided for libraries [S2, non-empirical].
- **No study measuring the effect of upper-bound caps on breakage or resolution failure found.** The debate is evidenced by opinion only.

### G. Supply-chain cooldown / minimum release age

- Dependabot cooldown GA July 2025; default 3 days from changelog dated 2026-07-14, explained by GitHub as past the window where most attacks live; security updates exempt; limited against dormant backdoors, maintainer sabotage, build-system compromise. GitHub blog "The case for a cooldown: Why Dependabot now waits before issuing version updates" [S2: snippet partial; 3-day figure consistent across several secondary posts].
- Adoption: 7 days most common (64.3% of 251 ecosystems) [S, arXiv 2609.16605].
- Incident windows (vendor analysis, Datadog Security Labs): Axios malicious versions found in ~3 h; s1ngularity/Nx live ~4 h (+1 h to revoke); Ultralytics 12 h and 1 h phases; xz-utils ~5 weeks. Authors argue a 12 h cooldown would have blocked the first two [S2]. These are hand-picked incidents by cooldown advocates, not a distribution.
- Takedown/detection latency in the academic record is old and heavy-tailed: 2019-era data had mean 209 days / **median 67 days** from availability to public report (Backstabber's Knife Collection, arXiv 2005.09535); >72% of reported malicious PyPI packages persisted on mirrors long after discovery (arXiv 2309.11021) [S]. These argue that a short cooldown only defends against the fast-detected class.
- **No peer-reviewed study of cooldown effectiveness exists in what I could find.** Any cooldown length is a policy choice, not a measured optimum.

### H. Bisecting versions / delta debugging

- Delta debugging (Zeller) is the basis of git-bisect; Artho's iterative delta debugging handles "fails now, passed earlier" by narrowing to an adjacent pass/fail pair, using binary search when a passing version is known [S2, people.kth.se idd-full.pdf].
- Practitioner tools: `where-broke` (npm), `@sigma/bisect` (JSR), git bisect inside a dependency (Test Double) [S2].
- No empirical study of version-level bisection accuracy, monotonicity violations, or flake sensitivity found. Both tools assume a reliable pass/fail oracle and a single changing variable.
- Flaky-test rerun cost evidence: Gruber et al., "An Empirical Study of Flaky Tests in Python", arXiv 2101.09077: for non-order-dependent flaky tests, ~170 reruns needed for 95% confidence a passing test is not flaky; ~31 random-order runs for order-dependent; Alshammari et al. (24 Java projects, 10,000 reruns each) still missed some known-flaky tests [S].

## Corrections to wave-one docs

1. 05 §4: "A 2026 study of 303 Python repos found 53% of breaking changes arrive in minor releases ... [V: thesis]". Problems: it is a master's thesis (not peer reviewed); the 303 repos are the source of 9,081 issues, and "53%" is the **share of coded breaking changes that landed in minor releases**, not the chance that a minor release breaks. Wave one then states in §4 item 1 "Semver labels are weak (53% of breakage in minors)", which is a defensible direction but should cite the peer-reviewed rates instead (Maven 11.58% of updates impact clients; npm 44% of manifesting breaks in minor/patch; Go 28.6% of non-major upgrades).
2. 05 §4, JP Morgan: "71.4% precision on three synthetic repos [V: https://arxiv.org/html/2510.03480v2]". I could not re-see this number; the paper is a short ASEW 2025 workshop paper (pp. 34-37). Downgrade to [U] and do not use as evidence of capability.
3. 05 §4: "I found no benchmark for changelog-only breaking-change detection [U gap]" stands, but the section omits the much more decision-relevant measured numbers: BUMP/Byam/Fruntke-Krinke (19-27%), DepBench (51%), BreakGuard (30.3% detection), Hejderup (47%/35% fault detection by tests).
4. 05 §2: "The claim that GitHub made a 3-day cooldown the default on 2026-07-14 is single-source [U]". Now corroborated: GitHub blog URL plus several secondary reports with the changelog date; still not fetched from github.com, so [S2] rather than [V]. One aggregator claimed 24 h and should be ignored.
5. 05 §2: "Mend's best-practices preset defaults to a 3-day npm minimum age [V ... secondary]" is not touched by this wave; leave tagged secondary.
6. 05 §5/§9 and BUILD_PLAN: "Reliability rule: ... rerun N=3". Not supported by flaky-test literature as a way to *rule out* flakiness (170 reruns for 95% confidence on non-order-dependent flakes in Python code [S]). N=3 is adequate only for failures that are deterministic by construction (compile errors, import errors, removed symbols). See defaults below.
7. 05 §5 "Adapt ... if feasible" and policy `adapt_limits: {max_files: 5, max_lines: 150}` and `workaround_limits: max_lines 40`: no literature supports these thresholds; they are guesses and should be labelled [U] in the doc.
8. 05 §7/§10 `auto_merge: [patch, minor]` on green CI + canary: no study evaluates auto-merge outcomes; and tests detect <50% of dependency faults [S]. Label as a policy choice with a rollback metric, not evidence-backed.
9. 05 §9 "O(log n) installs" bisect: valid only under monotonicity; nothing measured. Wave one already includes the linear-scan fallback; add an explicit monotonicity check.
10. 05 header says tags [V] are verified in "fetched/searched" sources. Several [V] items were verified only from search snippets. Recommend renaming to [S] where only snippet-level.

## Implications and concrete changes

### Defaults: evidence-backed vs guessed

| Parameter | Wave-one value | Recommended | Evidence status |
|---|---|---|---|
| Cooldown, default | 3d | Keep 3d for patch/minor; allow 7d profile | 3d = GitHub vendor default [S2]; 7d = modal user choice [S]; incident windows are 1-5 h [S2]. Both lengths are policy, not measured. |
| Cooldown, major | 14d | Keep, mark guess | [U] no data. |
| Cooldown, security | 0d | Keep, but still run canary and malware lookup | Matches Dependabot (security updates exempt) [S2]. |
| Cooldown, new-publisher/first-release-in-a-year packages | none | Add optional "dormant package" extension to 14d | [U]; motivated by xz-style long windows [S2], not tested. |
| Failure reruns (hard failures) | 3 | 3 on bad version and 3 on good version, only when failure class is deterministic (compile/import/symbol-missing) | [D] 3/3 consecutive failures of a p=0.5 flake happen 12.5% of the time, so N=3 cannot exclude flakes. |
| Failure reruns (test-assertion/behavioural) | 3 | 10 on bad, 10 on good, and compare to the flake rate of the same test on last-good history; escalate to 30 only if ambiguous | [U] chosen from the flaky-test literature (31 runs order-dependent, 170 for 95% on non-OD [S]); full 170 is too costly and I mark 10/30 as pragmatic guesses. |
| Class-C persists | "3 reruns" then escalate | keep as above | [U] |
| Adapt attempts | unspecified | up to 3 attempts, each with API diff + compiler/test error + release-note excerpt in context; accept only if full build/test passes on V_new and V_cur | Directional evidence: richer context raises success (Byam P8 27%; BDUpdater 90.5%) [S]; first-attempt success 19-27% (Java) to ~51% (mixed) means plan for 50-80% failure and make fall-through to pin/constrain automatic [D]. Attempt count is [U]. |
| Adapt-vs-pin threshold | max 5 files / 150 lines | Replace size limits with outcome gates: patch compiles and passes on both versions; touches no public API; reviewer-visible diff. Keep size caps as secondary [U]. | No literature on size thresholds. |
| Merge of any LLM patch without human | auto on green | keep, but require: surface diff of consumed API empty or explained, plus differential check on V_cur vs V_new | Tests alone catch 35-47% of injected faults [S]. |
| Bisect | O(log n) | binary search with monotonicity probe (test 3 evenly spaced versions) then linear on suspect window | [U] |
| Pin max age | 90d | keep | [U] |
| Revisit schedule | 14d | keep | [U] |
| Bot merge-rate expectation for dashboards | none | track own merge rate; reference 70% (Dependabot, 502,752 PRs) and 98% (Renovate in nf-core) as external anchors | [S] |

### Changes to docs/research/05-dependency-updates-and-constraints.md

1. Replace the "Evidence is thinner than vendors imply" block in §4 with a table of the numbers in Findings C and D, with the thesis demoted and the JPM number marked [U].
2. §4 operational conclusion 1: change "the authority is (a) the project's own tests ... and (b) API surface diff" to a three-signal gate: (a) repo tests, (b) consumed-API surface diff (AexPy/griffe for Python, cargo-semver-checks for Rust, GoSVI/apidiff for Go, japicmp-style for Java), (c) differential generated tests where cheap. Note that (c) mostly finds crash-type breaks (30.3%/95.1% [S2]).
3. §4 operational conclusion 2: when giving the LLM migration context, include the library's own breaking-change record, the API diff and exact compiler/test errors (the three inputs associated with the high success rates), not release notes alone.
4. §5 reliability rule: replace "rerun N=3" with the two-tier rerun rule above, and record the flake rate of each test on last-good history in the evidence bundle. Add the monotonicity probe in §9.
5. §5 outcome table, "Adapt": require green on V_new AND V_cur (already stated) and add "no partial fix: build-level success only", since error-level success (78%) far exceeds build-level (27%) [S].
6. §6 registry: add `bad_spec_kind: exclusion | cap` and default to `exclusion` (`!=X`) for library repos; allow `cap` (`<X`) only for application repos or time-boxed holds. Rationale: caps block downstream resolution even when nothing breaks (Poetry FAQ [S2]); no empirical measurement exists [U], so this is a design hedge.
7. §2: add cooldown evidence summary (3d default; 7d modal choice; incident windows hours; xz weeks) and an explicit note that the factory's cooldown is a policy knob without an effectiveness study. Add the dormant-package extension as optional.
8. §2: record Renovate vs Dependabot merge rates (98.46% vs 72.13% nf-core; 70.13% across 502,752 Dependabot PRs) as context; do not use as a reason to prefer either bot.
9. §10 policy: mark `adapt_limits`, `workaround_limits`, `auto_merge` as [U] choices; add `rerun: {deterministic: 3, behavioural: 10, escalate_to: 30}` and `metrics.reverted_auto_merge_rate` with a tighten-policy trigger.
10. Add to Sources: the arXiv ids and DOIs in the Sources section below.

### Changes to docs/BUILD_PLAN.md

- Line 18 (decision outcomes): add "adapt is attempted with bounded budget; failure falls through to pin/constrain automatically" and "partial fixes are never merged".
- Line 29 (detection/cooldown): keep "cooldown enforced in factory resolver"; add default 3d/7d profile and note it is a policy choice, not measured.
- Line 40 (Phase 2): add exit criterion: canary includes a consumed-API surface diff; add a benchmark task: replay a slice of BUMP (Java) or a set of real Python breaking updates to measure the factory's own adapt success rate against the 19-27% / 51% published anchors. This makes the factory's adapt-vs-pin thresholds empirical rather than guessed.
- Line 71 (open verification items): remove "the 3-day Dependabot default is single-source" (now corroborated at [S2]); keep "PyPI 14-day upload change" (not touched here; one result mentioned a "14 day release lock" but I did not verify it); add "re-read primary papers for the numbers in W2-03 before hard-coding".
- Add a metrics line: measured adapt success rate, reverted-auto-merge rate, flake rate per test, pin age.

## Remaining unverified items

- Every number tagged [S]/[S2] needs a body-level read (arXiv ids/DOIs in Sources). Highest priority for hard-coding: Hejderup test-detection rates, Fruntke-Krinke 23%/19%, Byam 27%/78%, DepBench 104/203, BDUpdater 90.5% definition of "recovered", BreakGuard 30.3% and its cost.
- Ochoa per-level breaking rates (61.8/37.9/14.6): which dataset and metric? Ambiguous in the summary.
- GitChameleon v1 GPT-4o numbers conflict between secondary summaries.
- PyPI client-level breakage rate and Cargo patch-vs-minor split: no peer-reviewed source located.
- Effect of upper-bound caps on breakage/resolution: no study located.
- Auto-merge outcomes (revert rates) for Dependabot/Renovate: no study located.
- Cooldown effectiveness: no study; takedown-latency numbers are 2019-era (median 67 days to public report) and may be stale; current npm/PyPI registry takedown medians not located.
- Version-bisection accuracy/non-monotonicity: no empirical study located.
- Dependabot GitHub changelog 2026-07-14 and PyPI 14-day upload change: not fetched from the vendor pages.
- Kennesaw thesis F1 0.85 and 53%: thesis only, author/title not seen.
- Whether the 10.05% "manual rework after closed Dependabot PR" figure belongs to a different paper than He et al.

## Sources

Primary-venue pages that appeared in search results (abstract-level only; bodies not fetched):

- BUMP: arXiv 2401.09906, SANER 2024
- Breaking-Good: arXiv 2407.03880, SCAM 2024
- Byam: arXiv 2505.07522
- Fruntke & Krinke, "Automatically fixing dependency breaking changes", FSE 2025, DOI 10.1145/3729366
- BigBag: arXiv 2606.24446
- "Break to Adapt" (BDUpdater): FSE 2026, DOI 10.1145/3808116
- DepBench "Update from Hell": arXiv 2608.30300; SWE-Chain: arXiv 2605.14415; SWE Refactor Bench: arXiv 2608.23564; BeyondSWE: arXiv 2603.03194; RepoRescue: arXiv 2607.01213
- LLM Agents for Automated Dependency Upgrades: arXiv 2510.03480 (ASEW 2025)
- GitChameleon: arXiv 2411.05830, 2507.12367; LibEvolutionEval: arXiv 2412.04478 (NAACL 2025); CodeUpdateArena: arXiv 2407.06249
- PCREQ/REQBench: arXiv 2508.02023
- BreakGuard: arXiv 2608.20167; DiffTestGen: arXiv 2607.16024
- Ochoa et al.: arXiv 2110.07889 (EMSE 2021); Jayasuriya et al., ISSTA 2023, DOI 10.1145/3597926.3598147; Venturini et al., TOSEM 2023, arXiv 2301.04563; Go: arXiv 2309.02894 (ASE 2023); semver disobedience: arXiv 2209.00393 (ASE 2022); AexPy ISSRE 2022 DOI 10.1109/ISSRE55969.2022.00052; SLR: arXiv 2605.24397
- Hejderup & Gousios, JSS 2022, arXiv 2109.11921, DOI 10.1016/j.jss.2021.111097
- He et al., Dependabot, arXiv 2206.07230; Alfadel et al. MSR 2021 DOI 10.1109/MSR52588.2021.00037; Mohayeji et al. MSR 2023 DOI 10.1109/MSR59073.2023.00042; nf-core study arXiv 2607.10839
- Dependabot cooldown adoption: arXiv 2609.16605
- Release notes: Wu et al. arXiv 2203.15592 (ICPC 2022); Bi et al. (release note production and usage); Nath et al. arXiv 2511.18187
- Configuration/constraints: arXiv 2310.12598 (ICSE 2024); Dietrich et al., MSR 2019 (Dependency Versioning in the Wild)
- Flaky tests: arXiv 2101.09077; Alshammari et al. ICSE 2021
- Malicious packages: arXiv 2005.09535; arXiv 2309.11021
- Kennesaw thesis: https://digitalcommons.kennesaw.edu/masterstheses/145
- Default-argument breaking changes: ar5iv 2408.05129

Vendor/secondary pages (cooldown): GitHub blog "The case for a cooldown" (github.blog/security/supply-chain-security/...), dev.classmethod.jp "dependabot default cooldown 3 days", Datadog Security Labs "The case for dependency cooldowns in a post-axios world", cargo-semver-checks FOSDEM 2024 slides, Poetry FAQ, Artho iterative delta debugging (people.kth.se/~artho/papers/idd-full.pdf), where-broke (npm).
