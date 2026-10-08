# A-03 - Audit of W2-03 (dependency-update literature)

Auditor: independent model, 2026-10-07. Method: WebSearch summaries (independent queries, not reusing W2-03's wording), one raw.githubusercontent.com fetch (github/docs), one fetch of the Datadog Security Labs article (.md). Same access limits as the researcher: no paper bodies read. "SUPPORTED" below means re-seen in an independent search summary or a fetched primary/vendor page. It does not mean body-level verification.

## Verdict: TRUST WITH CAVEATS

The headline numbers are real and correctly transcribed where I could re-check them: Hejderup 58/20 and 47/35, Jayasuriya 11.58%, Venturini 44%, Go 28.6% / 3.1% / one-third, Fruntke-Krinke 23/19, Byam 27/78, DepBench 104/203, Gruber 170 reruns, the Axios and s1ngularity windows, and the Dependabot 3-day default. Tagging is mostly honest and on the conservative side. I found no fabricated figures. There are four main problems:

1. The summary-level aggregation mixes denominators. It sets npm's per-package rate beside Maven's per-update rate and calls both "per-update".
2. The rerun-rule correction misapplies Gruber's 170-rerun figure. It also ignores that doc 05 already pairs failures on the bad version with passes on the good one.
3. Two takeaways stretch the data: the cross-benchmark "~20% to ~90%" comparison, and BUMP as the factory's benchmark when its targets are Python and CLI repos.
4. Several key sources are preprints a few weeks old, tagged the same as peer-reviewed work. I could not confirm through search that BreakGuard (2608.20167) or the nf-core study (2607.10839) exist.

The evidence summaries are safe to use. Do not adopt the rerun tiers or the BUMP benchmark as written.

## Claims table

| # | Claim (W2-03) | Rating | Evidence |
|---|---|---|---|
| 1 | Maven: 11.58% of updates break the client; nearly half of client-impacting breaks are in non-major updates (Jayasuriya, ISSTA 2023) | SUPPORTED | Independent search summary quotes the abstract (18,415 artifacts, 142,355 deps, 71.60% outdated, 11.58%). |
| 2 | npm: ~12% of dependents and 14% of releases hit a break in non-major updates; 44% of manifesting breaks are in minor/patch (Venturini, TOSEM 2023) | SUPPORTED (number); denominator misused | The abstract matches. But 12% is per dependent package over the observation window, not per update. The doc's "per-update client breakage is about 10-14% (Maven, npm)" (§A and Summary 1) is therefore a mismatched-denominator comparison. |
| 3 | Go: 28.6% of non-major upgrades break; 3.1% of breaking changes are used by clients; about a third of clients may be affected | SUPPORTED | ASE listing summary: 86.3% compliant, 28.6%, 63.7% to 92.2%, 33.3% of clients, 3.1%. Note that 28.6% is a library-side API break rate, not a client-impact rate. |
| 4 | Summary 1: "roughly a quarter to a third of non-major upgrades contain an API-level break, across ecosystems" | UNSUPPORTED as a cross-ecosystem statement | Only Go gives about 29%. "One third" comes from Maven's 2014 data, which Ochoa 2021 itself revises (83.4% compliance). npm and Maven client numbers measure something else. Treat it as Go-only. |
| 5 | Tests cover 58%/20% of direct/transitive dependency calls and detect 47%/35% of injected faults (Hejderup & Gousios, JSS 2022) | SUPPORTED | Search summary quotes the abstract. Population: Java only, artificial faults. W2-03 caveats this correctly. |
| 6 | LLM repair on BUMP: Fruntke-Krinke up to 23% agentic / 19% zero-shot; Byam 27% full build, 78% of individual errors | SUPPORTED | Both confirmed. Byam's 27% is o3-mini with the richest prompt (the best of 5 models), not a typical model. |
| 7 | DepBench: best config 104/203 (51%), 5 ecosystems | SUPPORTED (preprint) | Confirmed: 51.2%. Preprint from 2026-08-31, not peer reviewed. W2-03 omits its four-state oracle, in which a repair without the bump must fail; that oracle is useful to the factory (see Missing). |
| 8 | BDUpdater: 90.5% of breaking updates recovered, 13 libraries / 84 clients, ~$0.05 per client; 83.8% of libraries keep breaking-change records | 83.8%: SUPPORTED. 90.5%/13/84/$0.05: PLAUSIBLE-UNCHECKED | The abstract snippet was truncated before the results. W2-03 itself flags that "recovered" is undefined. |
| 9 | BreakGuard detects 30.3% of BUMP breaks; 95.1% of detections are crash-type | PLAUSIBLE-UNCHECKED (low) | Two searches did not surface the paper (arXiv 2608.20167) at all. The [S2] tag on 95.1% is honest. Do not cite it in policy text until someone sees it. Related work the search did surface: an FSE 2024 study found that 2.30% of Maven updates had behavioural breaks affecting client tests. |
| 10 | Dependabot: 502,752 PRs, 70.13% merged, median lag 0.18 days (He et al., TSE 2023) | PLAUSIBLE-UNCHECKED | The paper is confirmed, but no snippet showed the numbers. The figures are internally consistent with known summaries. |
| 11 | nf-core: Renovate 98.46% vs Dependabot 72.13% merged (arXiv 2607.10839) | PLAUSIBLE-UNCHECKED (low) | Search did not find the paper. The closest nf-core paper (2601.09612) has no bot split. It is one community, and Renovate there may handle internal or template updates, so the rates are not comparable. W2-03 already says not to use it to choose a bot. Keep it out of dashboards as an anchor. |
| 12 | Cooldown adoption: 64.3% of 251 ecosystem configs choose 7 days; 83 of 92 adoptions security-motivated (arXiv 2609.16605) | PLAUSIBLE-UNCHECKED | The paper exists (GA July 2025 framing confirmed). No snippet showed the numbers. It is a 3-week-old preprint, and the denominator is ecosystem config entries, not repos. |
| 13 | Dependabot default cooldown is 3 days, version updates only | SUPPORTED (primary) | Fetched `github/docs` `data/reusables/dependabot/default-cooldown-period.md`: "cooldown period of 3 days to version updates ... does not apply to security updates". Agrees with W2-07 [V]. W2-03's [S2] tag understates it. The date 2026-07-14 is still unverified. |
| 14 | Incident windows: Axios ~3 h, s1ngularity ~4 h +1 h; 12 h would have blocked both; Ultralytics 12 h/1 h; xz ~5 weeks (all attributed to Datadog) | Axios/s1ngularity: SUPPORTED. Ultralytics/xz: UNSUPPORTED as attributed | The fetched Datadog article has the Axios and s1ngularity figures and "one week is the commonly recommended guidance". It does not mention Ultralytics or xz. Those figures need another source. |
| 15 | Gruber et al.: ~170 reruns for 95% confidence that a passing test is not flaky (non-order-dependent) | SUPPORTED (number); MISAPPLIED | Confirmed (ICST 2021). The figure answers "is this passing test flaky at a low failure rate?". The factory's question is different: "is a failure on V_new paired with a pass on V_cur real?". See Overreach. |
| 16 | Backstabber: mean 209 / median 67 days from availability to report | PLAUSIBLE-UNCHECKED | 209 days appears only in a secondary blog (Xebia; range -1 to 1216 days). I did not see the median. |
| 17 | Kennesaw "53% in minor" is a share of coded breaks, not a probability per minor release (correction 1) | SUPPORTED as a critique | This follows from the claim's own wording. The correction is right. |

## Tag-integrity findings

- Tags are honest at row level. The fetch failures are disclosed, and nothing is claimed as [V]. The [S2] downgrades (95.1%, 10.05%, BreakGuard cost) are appropriate.
- Problem: [S] makes no distinction between peer-reviewed papers and preprints only weeks old (DepBench 2026-08-31, cooldown study 2026-09-15, BreakGuard 2608, nf-core 2607, BigBag 2606). Add a `[pre]` marker. For BreakGuard and nf-core, even existence is unconfirmed by independent search.
- Problem: the Summary bullets aggregate tagged rows into untagged generalisations, which then read as evidence-backed. Examples: "a quarter to a third" (claim 4), "per-update 10-14%" (claim 2), and "same model class moves from ~20% to ~90%" (§C takeaway a). That last figure compares Java/BUMP with o3-mini and full-build success against JS, 13 libraries, and "recovered". It is not the same model, benchmark or metric. The text does partly caveat this, but it still states a lever size it cannot support.
- Problem: Correction 6 says N=3 is "not supported by flaky-test literature (170 reruns ...) [S]". That makes a [U] design argument look like a sourced finding (see Overreach).
- The Dependabot 3-day default is tagged [S2] here but [V] in W2-07. Reconcile to [V] for the 3-day default and its exclusion of security updates; keep [U] for the date.
- Every proposed default is marked [U] or [D] in the defaults table. That part is honest.

## Overreach findings

1. **Two-tier rerun (3 deterministic / 10 behavioural / escalate 30): not justified as written.**
   - Gruber's 170 measures how many runs it takes to catch a rarely failing flaky test. To hold back a version, the factory needs to rule out a flaky test producing k fails on V_new and k passes on V_cur. Doc 05 §5 already requires both, in the same sandbox and run.
   - Under independence, a flake with failure rate p on both versions produces that pattern with probability p^k(1-p)^k, which peaks at p=0.5. That is 1.56% for k=3, 0.098% for k=5, and 9.5e-7 for k=10.
   - W2-03's "12.5%" [D] ignores the paired requirement. Low-rate flakes, which drive Gruber's 170, almost never produce 3/3 fails.
   - The real gaps are different. Independence fails for environment- or order-correlated flakes. Version-dependent nondeterminism is a real regression, not noise. Rerunning a whole suite 20-60 times is expensive, and it costs real tokens for live-model CLI canaries (doc 05 open question).
   - The 10/30 numbers have no derivation. Recommendation: rerun only the failing test IDs, use paired k=5 with sequential early stop, run V_cur/V_new interleaved in random order, and record per-test flake history. Escalate on mixed results rather than jumping to 30.
2. **Consumed-API surface diff as a Phase 2 exit criterion: sequencing conflict.** BUILD_PLAN builds the surface extraction and diff library in Phase 3 (line 42). AexPy, griffe and cargo-semver-checks diff the whole public API, not the consumed subset. "Consumed" needs import and call-site extraction, which has false negatives for dynamic Python. Evidence (claim 3: only 3.1% of Go breaks are used) supports filtering to consumed symbols, so the direction is right but the phase is wrong.
3. **`!=bad` exclusion over `<` caps for libraries: a defensible hedge, under-specified.** No study backs it (W2-03 says so).
   - It is Python-centric. Cargo version requirements have no `!=`. npm needs `<X || >X`. Go uses `exclude` in go.mod, which applies only to the main module, plus `retract` upstream.
   - Exclusion lets later untested bad versions flow to downstream users. Doc 05's FastAPI example has 0.115.1 also bad. Caps exist precisely when the upper end of the bad range is unknown.
   - The registry needs an ecosystem-specific compile step and a rule: cap while the bad range is open-ended, exclusions once a fixed version is confirmed.
4. **Monotonicity probe "3 evenly spaced versions": weak.** Three probes rarely reveal a fail-pass-fail window, and doc 05 §9.4 already switches to linear scan when probes are non-monotonic. A cheaper, stronger check is to confirm the boundary: rerun last_good and first_bad with paired k, and also probe first_bad+1. Interleaved bugs that are fixed and then reintroduced are caught by the re-test loop anyway.
5. **BUMP-slice replay benchmark: wrong population.** BUMP is Java/Maven with Docker images of about 250 GB. The Phase 2 targets are a Python library repo and a CLI-contract repo, and the published 19-27% anchors are Java-specific. Use replays of real breaking updates from the target repos' own histories, plus a DepBench subset if it covers PyPI or npm. Use BUMP only if Java enters scope. The idea of measuring the factory's own adapt rate is sound.
6. **"Plan for 50-80% adapt failure" [D]: reasonable as a planning prior,** but it mixes metrics (first attempt vs agent run, Java vs mixed). Keep it as a [U] prior to be replaced by the factory's own measured rate.
7. **Cooldown 3d default / 7d profile: fine as policy.** Note that the Datadog article W2-03 cites recommends one week, and that 7 days is the modal user choice. Both 3d and 7d are policy choices. W2-03 says this.

## Missing (risks and alternatives not covered)

- **DepBench's four-state oracle** (base passes; upgrade without repair fails; upgrade with repair passes; repair without upgrade fails). It maps directly onto factory anti-gaming for adapt patches. It catches "fixes" that quietly revert or neutralise the bump. Add it to the adapt acceptance gate.
- **Behavioural-break base rate.** An FSE 2024 study ("Understanding the Impact of APIs Behavioral Breaking Changes on Client Applications") found that 2.30% of Maven updates had behavioural breaks affecting client tests. This calibrates the "unmeasured residual" in §D and is not cited.
- **Transitive drift.** Jayasuriya names transitive changes as a major breakage factor. The defaults table has no rule that the canary diffs the resolved transitive set and attributes the failure to it (doc 05 §9.3 records it only during bisect).
- **Ecosystem mismatch.** Nearly all repair and detection numbers are Java or JS. The factory's first targets (a Python library and harness CLIs) have no measured anchors, and no study covers CLI or event-schema contract breaks.
- **Auto-merge rollback mechanics** (revert PR, re-pin on post-merge failure) as a safety net, given that tests catch fewer than 50% of injected faults. W2-03 adds a metric but no action.
- **Preprint risk** for numbers that move policy (DepBench, cooldown adoption).

## Recommended edits

### Doc 05

| W2-03 proposed edit | Decision | Reason |
|---|---|---|
| 1. Replace §4 "evidence thinner" block with Findings C/D table; demote thesis; JPM to [U] | ACCEPT | Corrections are right. Add a `[pre]` marker to preprints and drop BreakGuard numbers until someone sees the paper. |
| 2. Three-signal gate (tests, consumed-API diff, differential tests) | ACCEPT | Supported direction (claims 3, 5). Note the 30.3%/95.1% figures are unconfirmed. |
| 3. LLM context = breaking-change record + API diff + exact errors | ACCEPT | Byam's richest prompt and BDUpdater support it as a direction. |
| 4. Replace rerun N=3 with the 3/10/30 two-tier rule | MODIFY | Use paired, interleaved reruns of the failing tests only, with k=5 and sequential stop. Keep N=3 for deterministic compile, import and missing-symbol classes. Record per-test flake history. Escalate on mixed results. Remove the Gruber-170 justification. |
| 4b. Monotonicity probe in §9 | MODIFY | Replace "3 evenly spaced" with a boundary confirmation (paired reruns at last_good/first_bad, plus first_bad+1). |
| 5. Adapt: build-level success only, no partial fixes | ACCEPT | Byam 27% vs 78% supports it. Add DepBench's four-state check (the repair alone, without the bump, must fail the new tests, or must not revert the version). |
| 6. `bad_spec_kind`, default `exclusion` for libraries | MODIFY | Use a cap while the bad range's upper end is unknown, and switch to exclusions once a fixed version is confirmed. Compile per ecosystem (Cargo has no `!=`; npm uses `<X \|\| >X`; Go uses exclude/retract). Tag [U]. |
| 7. §2 cooldown evidence summary + dormant-package extension | ACCEPT (summary); ACCEPT as optional [U] (extension) | Upgrade the 3-day default to [V] per W2-07 and github/docs. Move the Ultralytics and xz figures off the Datadog attribution. |
| 8. Record bot merge rates as context | MODIFY | Keep He et al. 70.13% [S]. Drop the nf-core 98.46%/72.13% comparison (existence unconfirmed, single community). |
| 9. Mark adapt/workaround limits and auto_merge as [U]; add rerun block and `reverted_auto_merge_rate` | ACCEPT with change | Use the modified rerun block (`rerun: {deterministic: 3, paired_k: 5, sequential: true}`). |
| 10. Add sources | ACCEPT | |
| Correction 10: rename snippet-only [V] to [S] in doc 05 | ACCEPT | |

### BUILD_PLAN.md

| W2-03 proposed edit | Decision | Reason |
|---|---|---|
| Line 18: bounded adapt budget, automatic fall-through; no partial fixes | ACCEPT | |
| Line 29: 3d/7d cooldown profile marked as policy | ACCEPT | Consistent with W2-07. |
| Line 40: canary includes consumed-API surface diff (Phase 2 exit) | MODIFY | Phase 2 should use a minimal consumed-symbol and import check plus the lockfile and transitive diff. The full surface-diff library stays in Phase 3 (line 42). Otherwise Phase 2 depends on Phase 3. |
| Line 40: BUMP-slice replay benchmark | MODIFY | Replace it with a replay set of real breaking updates from the target Python and CLI repos' history (plus a DepBench subset if it covers those ecosystems). Use BUMP only if Java is in scope. Keep the goal of measuring the factory's own adapt rate. |
| Line 71: drop "3-day default single-source"; add "re-read primary papers for W2-03 numbers" | ACCEPT | The 3-day default is now [V]. Also mark the PyPI 14-day item [V] and note it is not a cooldown, per W2-07. |
| Metrics line (adapt success rate, reverted-auto-merge rate, flake rate per test, pin age) | ACCEPT | Add a rollback action tied to the reverted-merge metric. |
