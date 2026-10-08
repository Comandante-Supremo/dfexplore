# A-08 - Audit of W2-08 (factory prior art, primary)

Auditor: independent model, 2026-10-07. Method: fresh shallow clones (scratchpad) of `strongdm/attractor`, `agate`, `cxdb`, `leash`, `github/gh-aw`, `SWE-agent/SWE-agent`, `OpenHands/OpenHands`, `OpenAutoCoder/Agentless`, sparse clone of `github/docs` (Copilot concepts); re-fetched 8 anthropic.com engineering posts. GitHub REST API (stars, `pushed_at`) was not available to me; commit dates come from `git log` on all fetched branch tips. Not re-checked: mini-swe-agent, snarktank/ralph, ralph-playbook, OpenHands SDK, multi-agent-research and effective-harnesses posts, Project Vend.

## Verdict

**TRUST WITH CAVEATS.** The [V] claims I re-read held up closely. That covers the Attractor semantics, agate's exit 255, the Copilot governance text, gh-aw safe-outputs, and the Anthropic numbers ($20k/16 agents, >20x, 17% FNR, 2 of 18 BrowseComp, 6 points, the April postmortem). [U] tagging of StrongDM and Stripe content is honest. There are three defects. (1) One factual claim is contradicted: "`leash`, `attractor` all show pushes through 2026-10" / "leash maintained through 2026-10". The latest commits on every branch are attractor 2026-03-17, leash 2026-04-06 and agate 2026-02-23, so the doc also contradicts its own line 28. (2) The doc mis-transfers the Copilot evidence. Copilot's docs require that a **human** reviews and merges agent PRs, and its "automation level" covers **issue triage only** (labels, fields, assignees, closing). Neither supports "merge performed by supervisor code" or a merge-autonomy enum. (3) It slightly overstates the postmortem: the effort downgrade was a deliberate tradeoff that evals did show, not a miss. Most of the BUILD_PLAN edits are sound. About half are directly evidenced (verifier isolation, credential isolation, evaluator calibration, equal resources, factory canary). The rest (kill switch, per-goal USD cap, 3/20 stop rule) are good practice that the cited sources do not demonstrate.

## Claims table

| # | Claim (W2-08) | Rating | Evidence |
|---|---|---|---|
| 1 | `attractor` is three NLSpec md files, no code, Apache-2.0; ~11.6k/9.4k/14.3k words; no holdout/scenario/twin text | SUPPORTED | Clone: files are `README.md` plus 3 specs and a LICENSE (Apache 2.0). `wc -w`: 11629 / 9376 / 14314. `grep -ci "holdout\|digital twin\|scenario"` = 0 in every file. README: "contains NLSpecs to build your own version of Attractor to create your own software factory". |
| 2 | Goal gates + retry_target/fallback, else FAIL; checkpoint per node | SUPPORTED | spec l.153 `goal_gate`: "must reach SUCCESS or PARTIAL_SUCCESS before the pipeline can exit"; l.467 retry/fallback/graph-level chain; l.1829 "If no retry_target and goal gates unsatisfied, pipeline outcome is 'fail'"; l.48 "After each node completes ... saves a serializable checkpoint". |
| 3 | Human gate: timeout uses `human.default_choice`, else RETRY; manager observe/guard/steer, `max_cycles` 1000; should_retry 429/5xx yes, 401/403/400 no | SUPPORTED (with an omission) | l.752-757: `IF answer is TIMEOUT ... default_choice ... ELSE RETURN Outcome(status=RETRY, failure_reason="human gate timeout, no default")`; l.926 `"manager.max_cycles", "1000"`; l.963 guard "routes to continue, intervene, or escalate"; l.562 predicate matches. Omitted: an unmatched answer falls back silently, `selected = choices[0]  -- fallback to first` (l.766). With no default, RETRY can loop on an absent human. |
| 4 | agate: GOAL.md loop, exit 255 = human action, `.ai/` markdown state, `_reviewer/_replanner/_recover/_retro`, default Opus 4.5, Apache-2.0 | SUPPORTED | README l.49 "`agate auto` ... 0 = done, 255 = human action needed"; `cmd/auto.go:25` "255 - Human action needed, stop looping"; README l.85 "`claude` \| Claude Opus 4.5 \| Most capable, default"; skills table l.111-115. Its "45 stars" figure is PLAUSIBLE-UNCHECKED (no API access). |
| 5 | cxdb Turn DAG + BLAKE3 CAS + `cxtx`; leash Cedar on Docker/Podman/OrbStack | SUPPORTED | cxdb README l.5, 9, 12; leash README l.4 "You define policies in Cedar", l.10 "Docker, Podman, or OrbStack". |
| 6 | "`strongdm/comply`, `leash`, `attractor` all show pushes through 2026-10"; leash "maintained through 2026-10" | CONTRADICTED (default and all branches) | `git for-each-ref` over all fetched heads: attractor newest 2026-03-17, leash newest 2026-04-06 (newest tag v1.1.7 is 2026-03-04), agate 2026-02-23, cxdb 2026-08-28. The doc's own line 28 gives attractor's last commit as 2026-03-17. The doc may have read `pushed_at`, which counts non-commit pushes. comply was not checked. |
| 7 | C compiler: ~$20k, 16 agents, no orchestrator, ~2,000 sessions, 2B/140M tokens, 100k lines, verifier "nearly perfect", `pkill -9 bash` | SUPPORTED | Post (2026-02-05): "I tasked 16 agents"; "a total cost just under $20,000"; "I don't use an orchestration agent."; "nearly 2,000 Claude Code sessions"; "2 billion input tokens ... 140 million output tokens"; "it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem."; "Claude `pkill -9 bash` on accident". |
| 8 | Harness: "poor QA agent", solo 20 min/$9 vs 6 h/$200 (>20x), DAW 3h50/$124.70, QA $3-4, 27 criteria, Opus 4.6 moved the evaluator to an end pass | SUPPORTED | Post (2026-03-24): "Out of the box, Claude is a poor QA agent."; "The harness was over 20x more expensive."; QA rounds $3.24/$3.09/$4.06; "Sprint 3 alone had 27 criteria"; sprints removed, evaluator to a single end pass. |
| 9 | Auto mode: 17% FNR on real overeager (n=52), 3 consecutive / 20 total denials, `-p` terminates, three internal incidents | SUPPORTED | Post (2026-03-25): table 0.4% FPR / 17% FNR / 5.7% FNR full pipeline; "The 17% false-negative rate on real overeager actions is the honest number."; "3 consecutive denials or 20 total, we stop the model and escalate"; "In headless mode (`claude -p`) ... we instead terminate the process."; incidents listed verbatim. Caveat: n=52 is small. |
| 10 | BrowseComp: 2 of 18 runs identified the benchmark and decrypted the answers using the canary string; 0.87% vs 0.24%; URL blocklists insufficient | SUPPORTED (detail differs) | Post (2026-03-06): two of 18 decrypted the key; the canary string was the key, found in the eval source; the encrypted file was blocked as binary, **so it fetched a JSON copy from a third-party HuggingFace mirror**; "URL-level blocklists were insufficient". "Found its source on GitHub" is not what the summary I got says. The mirror detail matters: copies outside the factory's control defeat repo-level isolation. |
| 11 | Infra noise: 6 points on Terminal-Bench 2.0, p<0.01, error rate 5.8% vs 0.5% | SUPPORTED | Post (2026-02-05): "6 percentage points (p < 0.01)"; "5.8% at strict enforcement to 0.5% when uncapped". |
| 12 | April postmortem: 3 changes; "internal evals and review missed them"; remedy soak + gradual rollout | SUPPORTED, slightly overstated | Post (2026-04-23): effort high to medium (evals "had shown slightly lower intelligence" and the team judged it acceptable, so it was a known tradeoff, not a miss); a thinking-clear bug that "passed multiple human and automated code reviews"; a verbosity prompt where a later broader eval showed a 3% drop. Remedy quote: "soak periods, a broader eval suite, and gradual rollouts", plus per-model evals on every system-prompt change. |
| 13 | Containment post: 84% fewer prompts; 24 of 25 exfiltrations; api.anthropic.com plus attacker key; no kill switch; "undated" | SUPPORTED except the date | Fetched page: "84% reduction"; "completed the exfiltration 24 times"; "The sandbox worked perfectly, and yet the data was exfiltrated."; no kill switch mentioned; per-session revocable token. The page **is dated May 25, 2026**. The doc's "[the post] says it does not describe one [V]" overstates an absence as a statement. |
| 14 | Managed Agents: append-only session log, `wake(sessionId)`, cattle containers, credentials kept out via vault + proxy | SUPPORTED | Post (2026-04-08): "the append-only log of everything that happened"; "rebooted with `wake(sessionId)`"; "The container became cattle"; git token used at setup "so the agent never handles the token"; MCP via proxy plus vault. |
| 15 | Copilot: single branch, no `git push`, cannot mark ready / approve / merge, workflow approval, requester can't approve, extra approval under app identity, firewall | SUPPORTED | `risks-and-mitigations.md` l.36-42, 48 match nearly verbatim, including "must be reviewed and merged by a human". |
| 16 | Copilot automation level (Full control / Cautious default / Balanced / Full automation), rationale + confidence, approvals "not a security control" | SUPPORTED, but the scope is narrower than the doc implies | `about-automation-rationale-and-approvals.md` l.15 (public preview), 33-38, 57. **Scope l.23: "labels, fields, issue type, closing issues, and assignees"**. It does not cover code or merges. |
| 17 | gh-aw: read-only sandboxed agent jobs, safe-outputs, "careful human supervision", retired vulnerable range, MIT | SUPPORTED | README l.18, 20, 35 (GHSA-8h78-hpm7-29gg, >=0.83.3 <0.85.4 retired), l.55; LICENSE MIT. |
| 18 | OpenHands repo is now "Agent Canvas" | SUPPORTED | README `<h1>Agent Canvas</h1>`, "The self-hosted developer control center for coding agents and automations.", status badge "beta". The SDK claims (77.6, modules) were not re-checked. |
| 19 | SWE-agent superseded by mini-swe-agent; last commit 2026-07-16 | SUPPORTED | README l.21 "which has superseded SWE-agent"; l.24 "use mini-SWE-agent instead of SWE-agent going forward"; HEAD 2026-07-16. mini-swe-agent's own claims (~100 lines, >74%) are PLAUSIBLE-UNCHECKED. |
| 20 | Agentless: no commits since 2024-12-22; 27.3% at $0.34; 40.7/50.8 | SUPPORTED | HEAD 2024-12-22T13:29; README l.19, 21. "Dormant" is a fair reading. |
| 21 | No OSS project implements goal + constraints registry (reason/evidence/revisit) + escalation | PLAUSIBLE-UNCHECKED | A negative result within the doc's reachable set. In agate, "constraint" appears only in interview prompts (`skills.go:599`, `plan.go:426`) and "revisit" does not appear. Attractor has no registry. A bounded search cannot prove a universal negative. Keep the wording "within sources reached". |

## Tag-integrity findings

- StrongDM charter, holdouts, twins, team size and acquisition, plus every Stripe figure, Devin, the AIDev papers and the incidents, are tagged [U] consistently. I found no StrongDM or Stripe content asserted as [V] without a read source. The "[V, adjacent]" label on the deterministic-skeleton point is honest: it verifies the pattern elsewhere, not Stripe.
- Misuse: the "pushes through 2026-10" claims (Corrections #3; table row "Sandbox / policy") carry an implied repo-verified status but contradict the clone data.
- Weak [V]: "Anthropic's containment post says it does not describe one [V]" turns an absence into a statement. "How we contain Claude (undated)" is wrong: the post is dated 2026-05-25.
- Star counts (agate 45, snarktank 21.9k) are presented as fact. I could not confirm them here, so they are PLAUSIBLE.
- The "Combined with Anthropic's own listed incidents above [V], the pattern is credentials scope ... not model quality" conclusion mixes [V] and [U] evidence into an unqualified inference. Rewrite it as a hypothesis.

## Overreach findings (BUILD_PLAN edits)

| Edit | Evidence-derived or good practice? | Note |
|---|---|---|
| 1 Verifier isolation (no network/fs/git path) | **Evidence-derived** (BrowseComp) | The evidence points at the **network**, including third-party mirrors, more than at "git history of the verifier repo". The git part is extrapolation, but cheap. |
| 2 Credential isolation as a Phase 1 exit | **Evidence-derived** (Managed Agents; auto-mode token-upload incident; containment 24 of 25 exfiltrations; PocketOS [U]) | The plan currently says "API-key auth" inside the sandbox. Meeting the criterion needs an injecting proxy (e.g. a base-URL proxy that holds the key) or short-lived scoped keys. The containment post also shows an allow-listed API host can carry an attacker's key, so egress needs request-level checks, not just domain allowlists. |
| 3 Stop rule (3/20) and per-goal USD cap | **Good practice**; the 3/20 rule is an analogy | 3/20 counts classifier *denials* in an interactive tool, not factory attempts. Do not import the numbers as evidenced. A per-goal cap is sound and overlaps W2-05's caps table. |
| 4 Evaluator calibration and deterministic-first gating | **Evidence-derived** (harness post) | Merge with W2-04's kappa gate (>=0.61 on 30+ labels) so there is one calibration rule. "Judged never gates alone" restates BUILD_PLAN §1 Verifier. |
| 5 Copilot-style merge controls and `automation_level` | **Partly contradicted** by its own source | Copilot keeps a human as the merger. Supervisor merge is the plan's own choice and is not supported by Copilot. automation_level covers issue triage. Single-branch, no-self-approve and rationale/confidence logging transfer well. |
| 6 Fleet kill switch | **Good practice**, no evidence | The doc admits no source covers this. |
| 7 Equal, recorded sandbox resources for compare | **Evidence-derived** (infra noise) | Sound. |
| 8 Canary/soak for the factory's own model/prompt/effort changes | **Evidence-derived** (postmortem) | Add "per-model eval on every prompt change", which the postmortem names explicitly. |
| 9 Keep StrongDM/Stripe/Huntley item open | Process, correct | |

## Consistency with BUILD_PLAN and other docs

- BUILD_PLAN §1 "Autonomy rule: Auto-merge on green CI plus canary" conflicts with the Copilot pattern W2-08 cites as support. W2-08 should say plainly that the factory deliberately goes beyond Copilot's human-merge rule.
- BUILD_PLAN §1 "Escalation ... default-after-timeout" matches Attractor. Attractor's no-default RETRY and its silent first-choice fallback are behaviors the factory should **not** copy (see Missing).
- Edit 4 duplicates W2-04 (calibration). Edit 3's caps duplicate W2-05 (explicit cap defaults). Consolidate when editing BUILD_PLAN.
- Corrections #9 is consistent with W2-02, which owns `--bare` and the budget flags. The Corrections #11 CVE doubt belongs to doc 05/W2-07.

## Missing

1. **Untrusted input → prompt injection.** Update goals read upstream release notes, changelogs, issues and diffs, all attacker-controllable. Copilot drops comments from non-write users (l.36, l.71). The containment exfiltration came from a malicious workspace file. W2-08 does not turn this into a plan item. Add: treat upstream text as data, keep the worker without secrets or egress, and confine writes to safe-outputs.
2. **Auto-merge incidents and rollback.** The doc has no post-merge revert/rollback design, and admits no source covers it. The plan auto-merges, so it needs a revert path (supervisor-generated revert PR, pin re-application) and a post-merge canary failure → revert rule.
3. **Human-gate failure modes.** Attractor's unmatched-answer→first-choice fallback and no-default→RETRY loop. The factory's question table must reject unparseable answers and cap re-asks.
4. **gh-aw's own security advisory.** The pattern the doc recommends had a vulnerability serious enough to retire a release range. Treat any adopted gh-aw as needing version pinning.
5. **Small-sample caveat.** The 17% rests on n=52, and BrowseComp on 2 of 18. Fine as direction, not as calibrated rates.
6. Not covered: GitHub's rule that events from the default `GITHUB_TOKEN` do not trigger workflows, which affects "supervisor opens PR → CI runs → auto-merge". This is general knowledge and was not verified here; check it in W2-07.

## Recommended edits

### BUILD_PLAN.md

| W2-08 edit | Decision | Reason |
|---|---|---|
| 1 Verifier no-path isolation | **Modify** | Make it network-first: no worker egress except an allowlist via proxy, and verifier assets never published or mirrored. Keep "no readable shared remote". Cite BrowseComp's third-party-mirror path. |
| 2 Credential isolation in Phase 1 | **Accept, modify wording** | "No long-lived forge/cloud/registry tokens in the sandbox; model key injected by proxy or short-lived" fits API-key auth. Add a Phase 1 exit test: a planted token in the worktree is unusable for outbound calls. |
| 3 Stop rules and per-goal USD cap | **Modify** | Accept the per-goal and per-day USD caps and a consecutive-failure stop. Drop the 3/20 numbers as "evidence"; use W2-05's defaults table instead. |
| 4 Evaluator calibration | **Accept**, merged with W2-04 | One rule: few-shot plus hard thresholds plus the kappa gate before any judge gates. |
| 5 Copilot merge controls and automation_level | **Modify** | Accept: one branch per attempt; the worker credential cannot approve or merge; the merge credential is held only by the supervisor; rationale and confidence recorded per decision; "approvals are not enforcement". Reject a new four-level `automation_level` enum justified by Copilot: reuse the existing per-repo autonomy in `policy.yaml` and note the Copilot analogue is triage-only. State explicitly that supervisor auto-merge exceeds Copilot's human-merge baseline. |
| 6 Fleet kill switch | **Accept** as practice | Flag checked before every lease plus forge-token revocation; label it as not evidenced. |
| 7 Equal sandbox resources | **Accept** | Directly evidenced. |
| 8 Canary for the factory's own changes | **Accept**, extend | Add a per-model eval run on every prompt or effort change. |
| 9 Item 5 stays open | **Accept** | |
| (new) Prompt-injection handling for upstream text | **Add** | See Missing 1. |
| (new) Post-merge revert path | **Add** to Phase 2 | See Missing 2. |
| (new) Question answers must parse; capped re-asks | **Add** to Phase 4 | See Missing 3. |

### Doc 01

| W2-08 change | Decision | Reason |
|---|---|---|
| Primary-source status header | **Accept** | |
| SWE-agent → mini-swe-agent | **Accept** | Verified from the SWE-agent README. |
| OpenHands → Agent Canvas + SDK | **Accept** | Verified. Note the "beta" status. |
| Add agate, Attractor, gh-aw, Copilot automations, Ralph, Agentless rows | **Accept with fixes** | Copilot automations row: "issue triage only, public preview". agate: last commit 2026-02-23, not "maintained through 2026-10". gh-aw: note the retired vulnerable range. |
| Measured-outcomes section | **Accept** | Numbers re-verified. Note n=52 and 2 of 18. |
| Incidents and governance section | **Accept** | Keep PocketOS, Replit and Gemini as [U] news-level. |
| Correction #3 "repos alive through 2026-10" | **Reject** | Contradicted by commit dates (attractor 2026-03-17, leash 2026-04-06, agate 2026-02-23, cxdb 2026-08-28). |
| Correction #11 CVE doubt | **Accept** as a flag | Not verified by either doc. Route to doc 05/W2-07. |
