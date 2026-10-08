# 05 - Dependency updates and the constraints registry (Goal type 1)

Research date: 2026-10-07. Tagging: [V] = verified in a fetched/searched source (URL given); [U] = UNVERIFIED (from background knowledge or a single secondary source). Several primary doc sites (docs.renovatebot.com, docs.github.com) were unreachable by direct fetch from the research sandbox, so Renovate/Dependabot details come from search snippets and secondary posts; re-verify against primary docs before implementing.

## Summary

- Upgrade detection is a solved polling problem for registries; the hard part is deciding what to do when a new version breaks. Renovate/Dependabot stop at "open a PR" and have no first-class concept of a justified, expiring hold.
- Cooldown / minimum-release-age is now standard across uv, pip, npm, pnpm, Bun, Yarn, Renovate and Dependabot. It is a supply-chain defense, not a stability signal, and it is easy to misconfigure (units differ, transitive/lockfile paths bypass it).
- LLM changelog analysis is useful for triage and migration drafting but is not reliable as the sole gate: changelog quality varies, and the best evidence is for diff-based and compiler/test-driven approaches. The test suite plus a characterization of the consumed surface is the authority; the LLM proposes.
- Recommended core: a constraints registry (YAML, one file per repo, machine-checked) where a pin is a first-class successful outcome with reason, evidence, upstream link, and a revisit trigger; a nightly re-test loop probes the held-back versions so pins either expire or get renewed with fresh evidence.
- Upgrade pipeline: detect -> cooldown -> advisory check -> canary in sandbox -> decision procedure -> (adapt PR | pin | constrain | shim | escalate). Bisect over published versions when the first-bad version is unknown.

## Findings

### 1. Release detection per ecosystem

| Source | Detection mechanism | Notes |
|---|---|---|
| PyPI | JSON API `/pypi/<pkg>/json`, Simple API (PEP 691), RSS feeds | Yank is whole-release; installers skip yanked unless exactly pinned [V: https://docs.pypi.org/project-management/yanking, PEP 592]. Yanked flag must be read per release. |
| npm | registry packument (`/<pkg>`), `time` map, `deprecated` field | `time` gives publish timestamps needed for cooldown. `npm deprecate` is the npm analogue of a soft yank [U]. |
| crates.io | sparse index (`index.crates.io`), yank flag in index lines | Yank semantics similar to PyPI [U]. |
| Go modules | proxy `GET $GOPROXY/<mod>/@v/list`, `@latest`, `.info` timestamps; `retract` directives in go.mod | `retract` is Go's in-band known-bad mechanism [U]. Go cooldown is only an open proposal as of Mar 2026 [V: https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down]. |
| GitHub releases/tags | REST `releases/latest`, atom feeds `/releases.atom`, ETag conditional polling | Right source for CLIs and non-packaged tools. Tags without releases need `git ls-remote --tags`. |
| Homebrew | formulae.brew.sh JSON API; `brew livecheck` | [U] |
| Containers | registry tags list + digest via OCI API; pin tag AND digest | Renovate handles digest pinning; a Renovate flaw pre-44.3.1 skipped minimumReleaseAge for digest updates [V: https://releasealert.dev/cve/CVE-2026-88884, secondary]. |
| Non-package CLIs (Claude Code, Codex, Kimi, OpenCode) | whichever channel they ship on: npm / PyPI / GitHub releases / install script; plus `<cli> --version` on the host | Detect by the artifact the factory actually installs, not the marketing channel. Record resolved version + binary checksum. |
| Models/APIs | model-list endpoint diff, deprecation pages, response-header version fields | Out of scope here (see other docs); same registry schema applies with `kind: model`. |

Design: one `Watcher` interface returning `{upstream, version, published_at, yanked, deprecated, notes_url, source_etag}`; polling with conditional requests on a home server is cheap. Webhooks are unnecessary.

### 2. Renovate and Dependabot: what to copy, what is missing

Copy from Renovate:
- `minimumReleaseAge` (formerly stabilityDays) adds a `renovate/stability-days` status check; with `internalChecksFilter=strict` (the default) no branch is created until the age passes, and held updates appear on the Dependency Dashboard [V: https://docs.renovatebot.com/key-concepts/minimum-release-age/]. Needs a datasource with release timestamps; Maven can be unreliable [V, same].
- Known gap: lockFileMaintenance and transitive resolution can bypass the age gate (Renovate discussion #38115; the 44.x security preset nulls minimumReleaseAge for lockFileMaintenance) [V: https://github.com/renovatebot/renovate/discussions/38115, secondary summary]. Lesson: apply the cooldown inside the factory's own resolver, not only at the bot layer.
- `packageRules` (match by manager/package/update type, set automerge, grouping, allowedVersions, schedule), grouping presets, `schedule`, `automerge` gated on required checks, `lockFileMaintenance` as a scheduled job, Dependency Dashboard as the human-readable state [U beyond the above; widely documented].
- Mend's best-practices preset defaults to a 3-day npm minimum age [V: Nesbitt/Willison summary above, secondary].

Copy from Dependabot:
- `cooldown` with per-semver-level days (`semver-major-days`, etc.) and include/exclude lists; applies to version updates, not security updates [V, secondary: https://dev.to/instasla/taming-dependabot-a-2026-guide-to-grouping-cooldowns-and-cutting-pr-noise-246e]. The claim that GitHub made a 3-day cooldown the default on 2026-07-14 is single-source [U].
- `groups` (first-match wins) and `multi-ecosystem-groups` for one PR across ecosystems [V, same secondary].
- Known failure: pnpm 11 minimumReleaseAge verifies after resolve, so Dependabot can write a too-new transitive into the lockfile and CI fails [V: https://dev.classmethod.jp/en/articles/pnpm-11-minimum-release-age-dependabot-ci-failure/].

What both lack (the factory's differentiator):
1. No durable "we decided not to upgrade, here is why" record. Renovate `ignoreDeps`/`allowedVersions` and Dependabot `ignore` are bare rules; comments are optional prose, no evidence, no expiry, no re-test.
2. No notion of cause: they cannot distinguish "needs code change" from "upstream regression".
3. No cross-ecosystem or non-package upstreams (CLIs, models).
4. Cooldown is age-only: "a cooldown knows one thing about a package: how old it is" [V: https://arrangeactassert.com/posts/minimum-release-age-is-necessary-but-not-enough/]. Not a substitute for advisory/malware lookup.
5. Unit chaos across tools (npm days, pnpm minutes, Bun seconds) [V: Nesbitt]. Normalize in the factory and compile down to each tool's native setting.

Cooldown tooling reference: uv `exclude-newer = "7 days"` plus `exclude-newer-package` [V: https://pydevtools.com/handbook/how-to/how-to-protect-against-python-supply-chain-attacks-with-uv/]; pip 26.0 `--uploaded-prior-to` [V: https://pydevtools.com/blog/did-pip-26-close-the-gap-with-uv/]; pnpm 10.16 `minimumReleaseAge`, npm 11.10 `min-release-age`, Bun 1.3, Yarn 4.10 `npmMinimalAgeGate` [V: Nesbitt/Willison, secondary]. Pitfall: a global uv cooldown can break builds when build deps have no release in the window [V: https://github.com/NousResearch/hermes-agent/issues/19043].

### 3. Known-bad-version handling

- Registry-level: PyPI yank, crates yank, npm deprecate, Go retract. Treat as hard signals; a yanked version is never an upgrade target, and a currently-pinned yanked version triggers a re-evaluation.
- Advisories: OSV.dev `POST /v1/querybatch` returns IDs + modified timestamps, then fetch full records by ID and cache; ecosystem/name are case-sensitive [V: https://oneuptime.com/blog/post/2026-07-23-query-osv-api/view, secondary]. Malware appears as `MAL-` IDs; check the `withdrawn` field, since some are false positives [V: https://osv.dev/vulnerability/MAL-2023-71 and siblings]. `osv-scanner fix` exists but is limited (npm, Maven; experimental) [V: https://osv.dev/blog/posts/announcing-guided-remediation-in-osv-scanner, secondary summaries], so do not rely on it for Python/Rust/Go remediation.
- Upstream issue tracking (functional regressions, not security): no standard. The factory must link a GitHub issue to each bad version and poll its state (open/closed, linked PR, "fixed in" milestone/release). Detect fix by: issue closed AND a release newer than the bad one exists, then re-test (do not trust closure).
- Pin vs advisory conflict: if the pinned version has a known CVE, the pin is only allowed with a recorded risk acceptance (severity, exploitability, reachability) and a shorter revisit window; critical/reachable vulns force escalate.

### 4. LLM changelog and breaking-change analysis

Evidence is thinner than vendors imply:
- A 2026 study of 303 Python repos found 53% of breaking changes arrive in minor releases and go undetected an average >5 weeks before release; a commit-diff LLM approach reached best F1 0.85 (0.82 without chain-of-thought) [V: https://digitalcommons.kennesaw.edu/masterstheses/145, thesis; search summary].
- JS study (FSE 2026): 83.8% of libraries keep breaking-change records but quality "varies considerably"; agent BDUpdater recovered 90.5% of breaking updates across 13 libraries/84 clients [V: https://nesa.zju.edu.cn/download/auto_20260423_yifan-2026fse.pdf, via search summary].
- J.P. Morgan Java agent using migration docs: 71.4% precision on three synthetic repos [V: https://arxiv.org/html/2510.03480v2].
- I found no benchmark for changelog-only breaking-change detection [U gap]. Treat reliability as unmeasured.

Operational conclusions:
1. Release notes are a hint; the authority is (a) the project's own tests run against the new version, and (b) a diff of the API surface the repo actually consumes (public symbols imported, CLI flags/JSON output schemas invoked, HTTP endpoints called). Semver labels are weak (53% of breakage in minors).
2. Use the LLM for: summarizing notes into "what touches us", mapping notes to call sites, drafting the adapt patch, classifying a test failure as {our bug, upstream regression, environment flake}, and writing the evidence bundle. Do not use it to declare "safe, skip testing".
3. For CLIs without typed APIs, diff observable contracts: `--help` output, flag set, exit codes, JSON/NDJSON event schemas, config file keys. Cheap, deterministic, and the LLM only interprets the diff.
4. Require LLM claims to cite a release-note line or diff hunk; store the quote in evidence.

### 5. Resolution decision procedure

Inputs: candidate version V_new, current V_cur, canary results, advisory data, upstream issue search.

Step 0: Hard gates. If V_cur has a critical reachable vuln or malware -> must move off V_cur (adapt/constrain-to-safe/escalate); pin to V_cur not allowed.

Step 1: Classify the failure from canary evidence:
- A. Deterministic failure caused by an intentional upstream API change (documented removal/rename, deprecation completed).
- B. Failure matching an open or recently filed upstream bug/regression (issue exists, other users report it, maintainers acknowledge or a later release fixes it).
- C. Failure unexplained / flaky / environment.
- D. No failure.

Step 2: Choose outcome.

| Outcome | Choose when | Evidence required |
|---|---|---|
| Adopt (D) | canary green, cooldown passed, no advisory against V_new | green run IDs, surface diff clean |
| Adapt | A, change is mechanical and local (rename, signature, flag), contained within existing architecture, adapted tests pass on V_new AND on V_cur (backward compatible) if feasible | patch diff, test results both versions, release-note citation |
| Pin / hold | B and no safe adaptation; or A where adaptation is large/architectural but V_cur still safe; or V_new is yanked/withdrawn | upstream issue link, minimal repro or failing test log, bad-version list, V_cur advisory scan clean |
| Constrain | B with a known bad range and good later/earlier versions (e.g. `!=1.4.2`, `<1.5`) so non-bad versions can flow | bisect result (first bad, last good), repro on bad, green on neighbors |
| Work around | B and fix pending; a small shim/patch/config flag keeps V_new working, patch is under N lines, isolated, and has an owner/expiry linked to the upstream fix | shim diff, tests on V_new, upstream issue, removal trigger |
| Escalate | architectural/design change; genuine fork needed; ambiguity (C persists after 3 reruns); security conflict; adapt touches public API of this repo; multiple upstreams conflict (A needs X>=2, B needs X<2) | summary + options + recommendation |

Tie-breakers (opinionated): prefer pin/constrain over adapt when the fix is upstream-pending, because adapting to a regression bakes it in. Prefer adapt over pin when the change is intentional and permanent (class A) and small: pinning against intended direction accrues debt. Never choose work-around without a removal trigger. Time-box: any pin older than its revisit date without fresh evidence escalates.

Reliability rule: a hold needs reproduction ON the bad version and success on the held version in the same sandbox in the same run, so "pin" is never based on a flaky single failure (rerun N=3).

### 6. Constraints registry format

One file per repo: `.factory/constraints.yaml`, plus a derived lock-safe projection into native tools (`pyproject` constraints, `package.json` overrides/pins, Renovate `packageRules` or Dependabot `ignore`) so humans and other bots respect it. Registry is source of truth; generated files carry a header pointing back.

```yaml
schema: factory.constraints/v1
upstreams:
  fastapi:
    kind: pypi                      # pypi|npm|crate|go|github-release|container|cli|api|model
    constraints:
      - id: fastapi-hold-0.115
        outcome: constrain          # pin | constrain | workaround
        spec: "<0.115.0"            # or exact "==0.114.2"; ecosystem-native syntax
        exclude: ["0.115.0", "0.115.1"]   # optional explicit bad versions
        last_good: "0.114.2"
        first_bad: "0.115.0"        # from bisect
        created: 2026-10-07
        created_by: factory/run-8841
        reason: "Request validation regression breaks POST /ingest with union bodies"
        severity_if_ignored: breaks-prod    # breaks-prod | breaks-ci | degraded | cosmetic
        evidence:
          repro: ".factory/evidence/fastapi-hold-0.115/repro.md"
          ci_runs: ["runner-2/run-8841/jobs/3"]
          bisect_log: ".factory/evidence/fastapi-hold-0.115/bisect.json"
          reruns: {fail: 3, pass_on_good: 3}
        upstream:
          issues: ["https://github.com/<org>/<repo>/issues/<n>"]   # fill with real link
          fix_pr: null
        security_review: {last_good_advisories: none, checked: 2026-10-07, accepted_risk: null}
        revisit:
          triggers:
            - {type: new_release, when: "newer than first_bad"}
            - {type: issue_closed, issue: 0}        # index into upstream.issues
            - {type: schedule, every: 14d}
          max_age: 90d              # hard expiry: past this, escalate
        retest:
          last_run: 2026-10-07
          last_result: still_broken # still_broken | fixed | flaky | n/a
          next_candidates: []       # versions probed by canary
        status: active              # active | probing | released | expired | superseded
        removal_plan: "Drop constraint when canary passes on latest and issue closed"
```

Re-test loop (so pins do not rot):
1. Every watcher tick, for each active constraint: if any trigger fires, or `schedule` elapsed, run canary on the newest eligible version (and on `first_bad+1 ...latest` candidates: sampled, not all).
2. Result `fixed` (green N times, surface diff acceptable) -> open an auto-merge PR that removes/relaxes the constraint, set `status: released`, keep the record in `history/`.
3. `still_broken` -> update `retest`, extend `revisit`, comment on upstream issue only if the factory has permission and a new datapoint (never spam).
4. `flaky` or `max_age` exceeded -> escalate with a summary, never silently renew past max_age.
5. If upstream issue is closed as "wontfix / by design": reclassify to class A and open an adapt attempt or escalate.
6. Dashboard: list all active constraints with age, severity, next probe. A pin count and median age metric guards against debt.

### 7. Canary and sandbox testing

- Run candidate versions in a throwaway environment: ephemeral container/VM per run on the self-hosted runner, no production credentials, network restricted (egress allow-list to registries) so a malicious release cannot reach secrets [U design choice; supported by the supply-chain rationale in the cooldown sources].
- Matrix: {current, candidate, latest-in-range} x {unit, integration, contract/surface tests, smoke run of the real entrypoint}. For CLI upstreams, add a recorded-session test (golden transcripts of the wrapper's interaction with the CLI) plus a live smoke test with a cheap prompt.
- Install lifecycle scripts disabled where possible (npm `--ignore-scripts`) in the canary resolver step [U].
- Do not promote on resolver success; promote on the repo's test results. Lockfile update commits only after canary green.
- Runner hygiene: self-hosted runners should not run untrusted-PR-style code with persistent state; use ephemeral runners [U, standard guidance].

### 8. Gating the user's own upgrades

Apply the same machinery to the repo's own releases consumed by downstream repos: a downstream consumer-contract suite runs on every candidate release of a repo (e.g., library -> its dependents) before tag/publish; the factory auto-merges on green and holds on red with the same decision procedure. Self-upgrades of the factory itself (Claude Code, runner images) go through a separate canary repo first and are always pinned by digest.

### 9. Bisect strategy

When first-bad is unknown (jump from V_cur to latest fails):
1. Get ordered candidate list (exclude yanked/prereleases unless configured), from the registry's version list.
2. Verify endpoints: V_cur passes, V_new fails (rerun 3x each).
3. Binary search with the registry as the search space (`O(log n)` installs), each probe in a fresh sandbox, same lockfile otherwise frozen (change only the one package; record resolver output for transitive drift).
4. Output `last_good`, `first_bad`, and the transitive diff. If probes are non-monotonic (fail, pass, fail), switch to linear scan over the suspect window and record the flake rate.
5. If the first-bad is a package release with a source repo, optionally run git bisect between the two release tags using a source build, to point at the culprit commit for the upstream issue. Tag-to-tag only; do not build arbitrary commits automatically unless allowed.
6. Cost cap per bisect (probes, wall time); beyond cap, escalate.
Dependency-pair bugs: if the failure disappears when another package is held, record `interacts_with` in the constraint.

### 10. Per-repo policy file

`.factory/policy.yaml` (separate from constraints; policy is human-authored, constraints mostly machine-authored):

```yaml
schema: factory.policy/v1
goal: dependency-freshness
cooldown: {default: 3d, major: 14d, security: 0d, per_upstream: {claude-code: 1d}}
schedule: {detect: "every 6h", merge_window: "any", lockfile_maintenance: "weekly"}
autonomy:
  auto_merge: [patch, minor]            # on green CI + canary
  require_human: [major, "anything touching public API", new-pin-older-than-90d]
  allowed_outcomes: [adapt, pin, constrain, workaround, escalate]
  adapt_limits: {max_files: 5, max_lines: 150, no_new_dependencies: true}
  workaround_limits: {max_lines: 40, require_removal_trigger: true}
success_criteria: [ci_green, contract_suite_green, smoke_green]
groups:
  - {name: "lint-tooling", match: ["ruff", "mypy"]}
ignore_updates: []                      # prefer constraints.yaml, not this
upstreams:
  - {name: fastapi, kind: pypi}
  - {name: claude-code, kind: cli, channel: npm, probe: "claude --version"}
escalation: {channel: "<issue|email|chat>", sla: 24h, on: [architectural, fork, cve-conflict, expired-pin]}
```

Auto-merge on green is fine for the user's stated autonomy; the factory should still require the canary result, not just CI, for pin/constrain removal PRs.

## Worked examples

### A. FastAPI pin
1. Watcher sees fastapi 0.115.0 (version numbers illustrative; fill with real ones). Cooldown passes. OSV query clean.
2. Canary: tests fail on POST /ingest. Classification: failure not tied to a documented API change; search upstream issues, find one matching the traceback (open, other reporters).
3. Bisect: 0.114.2 good, 0.115.0 first bad, 0.115.1 bad, 0.116.x untested. Rerun 3x each.
4. Decision: class B, adaptation would change request models (architectural) -> Constrain `<0.115.0`, since later versions may fix it; evidence = repro + bisect + issue link. Pin is not a failure; PR auto-merged with the registry change plus the generated native constraint.
5. Revisit: new release triggers canary on newest; issue-closed trigger; 14-day schedule; 90-day max age. When 0.116.x passes, a removal PR auto-merges; if still broken at 90 days, escalate: "adapt now or fork?"

### B. Multi-harness messaging CLI (depends on Claude Code, Codex, Kimi, OpenCode CLIs)
- Each CLI is an upstream with `kind: cli`, watched on its real channel (npm for some, GitHub releases for others, [U: channels vary]); the installed version is recorded per harness adapter.
- Contract per adapter: flags used, output-stream schema (e.g. JSON events), exit codes, auth/config paths. Canary runs adapter tests against golden transcripts and a live smoke prompt on the candidate CLI version in a sandbox with a throwaway account.
- Failure: Codex's new release changes an event field name. Class A (intentional, release-note line cited). Adapt: adapter parses both old and new field names (backward compatible), tests pass on both -> auto-merge. Supported-version range in the adapter is widened.
- Failure: Kimi CLI release hangs on non-interactive mode. Class B; bisect over npm/GitHub versions; Constrain `kimi-cli != X` and mark the harness "degraded - pinned to last_good"; messaging tool still ships. Because users install the CLI themselves, the pin is enforced at runtime: the tool's doctor/preflight warns when the installed CLI is in the bad range, and the registry's bad-version list ships with the tool.
- Failure: OpenCode renames its config schema and drops the old one; adapt requires redesigning how the tool provisions config for all harnesses -> Escalate (architectural).
- Key difference from libraries: the user's machine, not the factory, controls the installed CLI version, so constraints must be expressed both as dev-time pins and runtime compatibility ranges (`tested_range`, `known_bad`).

## Implications for the dark factory

1. Make the constraints registry a first-class, validated artifact (JSON Schema), owned by the factory; generate native tool config from it. Block merges that add a pin without evidence, upstream link, and revisit.
2. Own the cooldown in the factory resolver; also set native cooldowns as defense in depth. Normalize units. Security fixes bypass cooldown but not canary.
3. Do not use Renovate/Dependabot as the brain. Use them (or just the factory's watchers) for detection and PR mechanics; the decision procedure lives in the factory. Reuse their grouping/schedule/dashboard ideas.
4. LLM role: triage, mapping notes to call sites, drafting adapt patches, classifying failures. Gate on tests and surface diffs. Require cited quotes. Record per-decision confidence and cost.
5. Add a bisect worker with cost caps and a "reproduce 3x" rule before any hold.
6. Metrics: active pins, median pin age, pins past revisit, time-to-detect fix, adapt-vs-pin ratio, auto-merge rate, reverted auto-merges (the signal to tighten policy).
7. Treat CLI/API/model upstreams with the same schema by adding `tested_range` and `known_bad` runtime fields.

## Open questions

- Where do constraints live for upstream-controlled installs (user's machine): bundled with releases or fetched? Trust and update-lag trade-offs.
- Should the factory comment on or open upstream issues itself (permissions, reputation, rate)? Default: draft only, human approves.
- How to detect "fixed upstream" for projects that fix silently with no issue link; probing every release is expensive for large suites.
- Cost control for canaries on live-model CLIs (token spend, nondeterminism); how many reruns are enough.
- Primary-source verification still needed: Renovate packageRules/allowedVersions/lockFileMaintenance details, Dependabot cooldown defaults, crates.io/Go/Homebrew detection specifics, PyPI 14-day upload change (single source: https://pydevtools.com/blog/pypi-rejects-new-files-after-14-days/ [U]).
- Whether an LLM-assisted changelog gate measurably beats test-only gating; no published benchmark found.

## Sources

- PyPI yanking: https://docs.pypi.org/project-management/yanking (undated); PEP 592
- uv cooldown: https://pydevtools.com/handbook/how-to/how-to-protect-against-python-supply-chain-attacks-with-uv/
- pip 26: https://pydevtools.com/blog/did-pip-26-close-the-gap-with-uv/
- uv cooldown pitfall: https://github.com/NousResearch/hermes-agent/issues/19043
- Cooldown across package managers (2026-03-04): https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down ; https://simonwillison.net/2026/Mar/24/package-managers-need-to-cool-down/
- Critique of age-only gating: https://arrangeactassert.com/posts/minimum-release-age-is-necessary-but-not-enough/
- pnpm 11 + Dependabot: https://dev.classmethod.jp/en/articles/pnpm-11-minimum-release-age-dependabot-ci-failure/
- Renovate minimum release age: https://docs.renovatebot.com/key-concepts/minimum-release-age/
- Renovate lockfile gap: https://github.com/renovatebot/renovate/discussions/38115
- Renovate digest age advisory: https://releasealert.dev/cve/CVE-2026-88884
- Dependabot cooldown/groups (secondary): https://dev.to/instasla/taming-dependabot-a-2026-guide-to-grouping-cooldowns-and-cutting-pr-noise-246e
- OSV API (secondary): https://oneuptime.com/blog/post/2026-07-23-query-osv-api/view ; OSV malware records e.g. https://osv.dev/vulnerability/MAL-2023-71
- OSV guided remediation: https://osv.dev/blog/posts/announcing-guided-remediation-in-osv-scanner
- LLM breaking changes: https://digitalcommons.kennesaw.edu/masterstheses/145 ; https://nesa.zju.edu.cn/download/auto_20260423_yifan-2026fse.pdf ; https://arxiv.org/html/2510.03480v2
