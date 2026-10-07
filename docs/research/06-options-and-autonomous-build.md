# 06 - Parallel option exploration (goal type 3) and autonomous greenfield build (goal type 4)

Researched 2026-10-07. Evidence base is thin in places: only 4 web searches were run, most of it practitioner/vendor writing. Claims from search results are cited; everything under "Implications" and the proposed templates is design opinion, not sourced fact. Source dates were mostly not exposed by the search tool and are marked "date n/a" (Kiro guide states a 2026-08-25 verification).

## Summary

- Best-of-N is a proven lever for coding agents, but gains depend more on the selector than on N: Best@K sits far below Pass@K in published work. Hybrid selection (execution tests plus learned/LLM verifier) beats either alone.
- Scoring should be a pipeline: hard gates (pass/fail), then measured metrics (Pareto), then judged criteria only among survivors. Judge LLMs have documented position, verbosity and self-preference biases; mitigate with order swaps, cross-family judges, and rubrics anchored to evidence.
- Spec-driven tools (Spec Kit, Kiro, BMAD) converge on the same pipeline: constitution/principles, requirements with testable acceptance criteria, design/plan, task list with stable IDs, per-task implementation. Their known failure modes (spec drift, self-consistent errors, seams between artifacts, ceremony overhead, lost rationale) map directly onto a dark factory.
- The factory should treat the conversation as input to a structured spec, propose its own definition of done (DoD), and encode the DoD as machine-checkable criteria wherever possible, flagging the remainder as "judged" criteria for the user.
- Rejection feedback should create a new goal that carries the original goal and decisions log, plus a "DoD defect" classification that feeds future elicitation templates.
- Proposed concrete artifacts are at the end of "Implications": a conversation-to-spec template and a sub-goal graph (YAML) format.

## Findings

### A. Parallel option exploration

**A1. Evidence that fan-out works, and where it saturates.**
- CodeMonkeys reached 57.4% on SWE-bench Verified with Claude Sonnet 3.5 using parallel candidates plus selection; selecting across an ensemble of existing top submissions' edits reached 66.2%, beating the best single member. https://scalingintelligence.stanford.edu/blogs/codemonkeys (date n/a)
- Meta's CWM: 53.9% without test-time scaling vs 65.8% with it; method generates k candidate solutions and 40 generated unit tests in parallel loops. https://arxiv.org/pdf/2510.02387 (2025-10)
- Satori-SWE: Best@50 of 41.6 matched a baseline needing Best@500 (~10x sampling), with selection combining a reward model and unit tests. https://arxiv.org/pdf/2505.23604 (2025-05)
- R2E-Gym: hybrid (execution + execution-free) verifiers scale significantly better; more test-agent rollouts can be more compute-efficient than more editing rollouts. https://arxiv.org/pdf/2504.07164 (2025-04)
- SWE-HERO: with K=32, Best@32 of 64.6 vs Pass@32 of 79.8, showing selection quality is the bottleneck. https://arxiv.org/pdf/2604.01496 (2026-04)
- Caveat: these are single-issue bug-fix benchmarks with a ground-truth test oracle. Open-ended "which architecture/library" choices lack that oracle (see A4).

**A2. Judge bias (relevant to the "judged criteria" tier).**
- Position bias varies strongly by judge and task (9 judges, 22 tasks, ~80k instances); simple order rearrangement reduces it. https://arxiv.org/html/2406.07791v1 (2024-06)
- Self-preference bias: judges rate their own outputs higher, linked to lower perplexity. https://arxiv.org/pdf/2410.21819 (2024-10)
- Verbosity and authority bias (fabricated citations raise scores) and a 12-bias catalogue (CALM). https://arxiv.org/html/2410.02736v1 (ICLR 2025); survey https://arxiv.org/pdf/2412.05579 (2024-12)
- Same-family judges share properties and agreement patterns, so a same-family "panel" is less independent than it looks (inference from the position-bias paper's family finding; a panel-specific study was not found: UNVERIFIED).
- Secondhand figures (GPT-4 same verdict on only 65% of order-swapped pairs; AlpacaEval win rate 22.9% to 64.3% by prompting for length) come from blogs and are UNVERIFIED against primary papers. https://grepture.com/blog/llm-as-a-judge-bias

**A3. Isolation mechanics (no web source needed; standard practice, stated as design knowledge).**
- One git worktree per candidate on its own branch shares the object store cheaply, but does not isolate ports, databases, caches, docker daemon, or global tool state. For a home server, isolate candidates in containers (one per candidate, own network namespace, own DB volume, CPU/memory cgroup limits) with the worktree bind-mounted. Use per-candidate port ranges or no host ports.
- Multi-agent edits to overlapping scope collide at merge time (noted for spec-driven multi-agent setups in https://www.verdent.ai/guides/coding/spec-driven-development, date n/a). Options exploration avoids this by design because candidates never merge; only one wins, or ideas are re-implemented.

**A4. Scoring design (synthesis, supported by A1/A2).**
- Gates first: build, tests, lint, security scan, constraint checks. Gate failure removes a candidate; gates are never traded off against metrics.
- Metrics second: latency, memory, binary size, test coverage, cost, LOC. Report as vector. With few candidates (typically 2-5) and incommensurable metrics, Pareto-front filtering is more honest than a weighted sum; weights hide value judgments and can be gamed. Use weighted scoring only after Pareto filtering, when a tie-break is needed, and record the weights in the decisions log.
- Judged criteria last, only among survivors: maintainability, fit to existing conventions, simplicity. Use rubrics with evidence quotes, pairwise comparison with both orderings, judges from a different model family than the generator, and length-normalized instructions.
- Where the judged tier decides the outcome and the margin is small, that is a genuine fork: escalate per the autonomy rule rather than let a noisy judge pick.

**A5. Merging ideas across candidates.** CodeMonkeys' cross-submission selection (A1) shows selection across diverse sources helps, but it selects whole patches, not merged ones. Merging is not demonstrated in the sources found; treat as UNVERIFIED for quality. Safer pattern: pick winner, then spawn a follow-up sub-goal "adopt idea X from loser B into winner", which re-runs gates.

**A6. Stopping rules and cost control (synthesis).**
- Fixed budget per goal (tokens and wall-clock) set in the goal, not by the agent.
- Early elimination (successive halving): run all candidates to a cheap checkpoint (compiles, smoke test), drop the bottom half, continue. Satori-SWE's 10x efficiency (A1) supports spending effort on smarter allocation over raw N.
- Stop when: one candidate dominates on the Pareto front for all metrics, or the budget is exhausted, or the top-2 gap is below measurement noise (then escalate or pick the cheaper to maintain).
- Per-candidate reproducibility: rerun metric measurements 3x on shared hardware; self-hosted runners are noisy neighbours of each other.

**A7. When NOT to fan out (synthesis).** One option is clearly dominant (autonomy rule); the task has a single correct answer discoverable by reading docs; candidates would need to touch shared external state (prod DB, a single GPU, rate-limited API); evaluation cost exceeds expected variance between options; no oracle and no way to produce a meaningful judge. Start with a cheap "dominance check" step (research and reasoning) before any fan-out.

### B. Autonomous build from a vision

**B1. Spec-driven tooling landscape.**
- GitHub Spec Kit: constitution (`.specify/memory/constitution.md`), then specify (independently testable user stories, requirement IDs, measurable outcomes), optional clarify, plan (checked against constitution), tasks (IDs like T001, parallel markers, exact paths), analyze (cross-artifact consistency), implement. Supports 30+ agents including Claude Code. https://codemyspec.com/blog/github-spec-kit-guide (2026); https://docs.plannotator.ai/frameworks/github-spec-kit (date n/a). The official repo is authoritative for command names (not fetched).
- Kiro: three files under `.kiro/specs/` (requirements, design, tasks); acceptance criteria in EARS notation; requirements-first vs design-first; bugfix spec (current / expected / unchanged behaviour); approval gate per phase. https://www.verdent.ai/guides/agents/kiro-spec-driven-development (verified by its author 2026-08-25; third-party, confirm vs official docs).
- BMAD: role-per-document chain (Analyst, PM, Architect, Scrum Master, Dev, QA); Product Brief, PRD, architecture, then self-contained story files with rationale, constraints, embedded tests and links back to sources; Quick Flow track for small work. https://codemyspec.com/blog/bmad-method-explained ; https://www.augmentcode.com/guides/bmad-method-ai-development (dates n/a). No controlled evidence of benefit found: UNVERIFIED.
- EARS patterns (ubiquitous, event-driven "WHEN...THE SYSTEM SHALL", state-driven "WHILE", unwanted "IF...THEN", optional "WHERE") are cited via Kiro sources above. Format origin (Mavin et al.) is background knowledge, not re-verified.
- PRD and ADR formats: not researched with sources this session; use ADR (context / decision / consequences / status) for the decisions log from background knowledge: UNVERIFIED here.

**B2. Documented failure modes of spec-driven / agent builds** (all from practitioner writing, not controlled studies): spec/code drift when no update model is chosen (flow-back / flow-forward / living spec); self-consistent errors when one agent writes spec, design, code and tests (green tests confirm an invented assumption); seams between artifacts where agents take the cheaper reading and do not silently repair wrong specs like humans do; overlapping agent tasks collide at merge; ceremony exceeding task size; rationale loss for constraints that don't mechanize. Sources: https://www.verdent.ai/guides/coding/spec-driven-development ; https://hackernoon.com/spec-driven-development-has-a-blind-spot-the-seams ; https://sys0.substack.com/p/agent-native-software-engineering (dates n/a).

**B3. Elicitation by conversation.** No strong primary source found for agent-led elicitation strategy (UNVERIFIED as research; below is design). Spec Kit's explicit clarify stage and Kiro's phase approval gates show tooling consensus that ambiguity is resolved before planning, not during coding.

## Implications for the dark factory

1. **Fan-out is a goal type with a mandatory dominance pre-check.** Before spawning candidates, one agent writes a short "options memo": candidates, why each is plausible, whether one dominates. If one dominates, proceed (no escalation, no fan-out). Fan-out only on a genuine fork.
2. **Isolation:** each candidate = container + worktree + private volumes, built on the self-hosted runner pool with a concurrency cap (e.g. N <= 3-4 on a home server). Candidate branches are named `goal-<id>/opt-<k>`; losers are archived (diff + metrics + logs), not deleted, so the follow-up "adopt idea" sub-goals and the rejection loop can use them.
3. **Scoring pipeline as a goal-schema field:**
   ```yaml
   scoring:
     gates: [build, unit, integration, lint, sec_scan, constraint_checks]
     metrics: [{name: p95_ms, dir: min, repeats: 3}, {name: rss_mb, dir: min}]
     selection: pareto_then_judge      # or weighted(w=...) with weights logged
     judged: [{name: maintainability, rubric: rubrics/maint.md, judges: 2, cross_family: true, both_orders: true}]
     escalate_if: {judge_margin_below: 0.1, pareto_front_size_gt: 2}
   budget: {tokens: 4_000_000, wall_min: 240, max_candidates: 4, halving_checkpoints: [compiles, smoke]}
   ```
4. **Judge hygiene:** judge model family differs from the generator; always evaluate both orders; judge sees evidence artifacts (metrics, diffs, test output), not candidate self-descriptions; a judged tier never overrides a gate or a dominating metric.
5. **Spec is the contract, tests are the oracle, and the author of tests should not be the author of code.** Counter the "self-consistent error" failure: a separate verifier agent (ideally different prompt context, optionally different model) writes acceptance tests from the spec's acceptance criteria before the builder sees them; builder cannot edit them without a logged decision.
6. **Pick an update model explicitly: living spec plus decisions log (ADR-style).** Code changes that contradict the spec must either update the spec (logged as a decision) or be reverted. A nightly "analyze" job (Spec Kit style) checks spec-task-code-test consistency and opens a drift sub-goal on failure.
7. **Scale ceremony to blast radius.** Small changes skip to a one-paragraph goal (BMAD Quick Flow analogue); only type-4 builds get the full conversation-to-spec pipeline.
8. **Scope-creep guard:** spec has an explicit `out_of_scope` list and `deferred` backlog; any agent-discovered requirement not traceable to a vision statement or acceptance criterion goes to `deferred` as a proposed goal, not into the current milestone. Each task and PR must cite a requirement ID; untraced diff hunks fail a gate.
9. **Elicitation strategy (design):** ask in this order, stopping each layer when answers become confirmatory: (a) purpose and user/outcome, (b) top 3-5 user-visible behaviours as examples, (c) hard constraints (stack, hosting, data, budget, security), (d) non-goals, (e) quality bars (performance, reliability, UX), (f) acceptance: "how will you know it's done?" Max ~5 questions per turn, offer a default with each question (the user can answer "default"), prefer concrete examples over abstract questions, and read the repo/env first so nothing discoverable is asked. **Stop asking when:** every vision statement maps to at least one testable criterion, every unresolved item is either low-impact (take the default, log it) or flagged as a risk, and the last round produced no new requirement. Only architectural forks or contradictory statements get escalated back mid-build.
10. **Rejection loop:** on rejection, the user's feedback is stored verbatim and becomes goal `G-n+1` with `parent: G-n`, context = original goal + full decisions log + the DoD. The factory first classifies each complaint: (i) criterion existed and was falsely passed (verifier bug), (ii) criterion missing (DoD defect), (iii) criterion wrong/ambiguous (spec defect), (iv) new requirement (scope change). Types ii-iii append a lesson to a persistent "DoD lessons" file that the elicitation and DoD-proposal steps read next time (e.g. "user cares about visible polish; add visual-review criterion", "always include install-from-scratch test"). Type i triggers fixing the verifier before the fix. Limit automatic rounds (e.g. 3) per original goal before escalating for a conversation.
11. **Milestone verification:** each milestone ends with gates (build/test/acceptance subset), a spec-trace report (requirements covered / uncovered / extra), and a short demo artifact (screenshots, run log, URL on home server). Failure runs a bounded repair loop, then escalates. Never let a milestone start on top of a red previous one.

### Proposed conversation-to-spec template (`spec.md`)

```markdown
# <Project name>        status: draft|proposed|accepted   version: n
## 1. Vision (user's words, verbatim quotes kept)
## 2. Users and outcomes
## 3. Scenarios (Given/When/Then or EARS), each with ID  S-01...
## 4. Requirements  R-01: WHEN <event> THE SYSTEM SHALL <response>   (traces to S-xx; priority MUST/SHOULD/COULD)
## 5. Constraints (stack, hosting, data, security, budget, licences)  -- hard vs preferred
## 6. Non-goals / out of scope
## 7. Quality bars (measurable: p95 < X, cold start < Y, accessibility level)
## 8. Assumptions made by the factory (each with default taken, impact, reversible?)  A-01...
## 9. Open risks and escalation triggers
## 10. Proposed Definition of Done (factory-authored)
   - Machine-checkable: criterion ID, command/test, threshold, linked R-xx
   - Judged: rubric, who judges (agent panel / user), linked R-xx
   - Demo script: steps the user can run in <5 minutes
## 11. Milestones (summary; full graph in goals.yaml)
## 12. Decisions log (ADR entries: id, context, decision, consequences, date, reversible)
```
Rules: every R traces to a scenario or vision quote; every MUST has at least one machine-checkable DoD item or an explicit "judged by user" flag; the user is shown sections 1, 4, 6, 8, 10 for confirmation (the user owns acceptance).

### Proposed sub-goal graph format (`goals.yaml`)

```yaml
goal: G-001
type: build                 # build | explore_options | fix | maintain
objective: "..."
spec: spec.md@<hash>
success:
  gates: [{id: D-01, cmd: "make acceptance", req: [R-01,R-02]}]
  judged: [{id: D-09, rubric: rubrics/ux.md, req: [R-07]}]
constraints: [C-01, C-02]
allowed_outcomes: [done, partial_with_report, escalate]
external_state: [{name: staging_db, access: read_only}]
escalation: "architectural change | genuine fork | ambiguity that changes DoD"
budget: {tokens: ..., wall_min: ...}
children:
  - id: G-001.M1
    title: "Walking skeleton"
    depends_on: []
    requirements: [R-01]
    verify: [D-01]
    on_fail: {retries: 2, then: escalate}
  - id: G-001.M2
    type: explore_options   # fork resolved by a type-3 sub-goal
    depends_on: [G-001.M1]
    ...
status: pending|running|verified|failed|escalated
parent: null                # set to G-prior for rejection-derived goals
derived_from_feedback: null # verbatim user text, plus classification i-iv
```
Graph rules: DAG only (reject cycles); a node is "done" only when its `verify` items pass on a clean checkout in CI; children may add siblings only via a logged decision; max depth 3 by default to limit sprawl; first milestone is always a walking skeleton (end-to-end thin slice, deployed on the home server) to flush out seam failures early.

## Open questions

- Does merging ideas across candidates beat pick-the-winner in practice? No evidence found.
- What judge-panel size and composition is cost-effective for non-coding judged criteria? No panel-specific study found.
- Best way to measure benchmark noise on shared self-hosted runners (dedicated runner for metrics?).
- How to verify "judged" criteria like UX or aesthetics when the user is the only true oracle: should the factory present a pre-acceptance preview and treat it as a soft checkpoint?
- Which of Spec Kit's commands/files to adopt directly vs a bespoke format; current official repo not fetched. Is compatibility with Spec Kit/Kiro file layouts worth it for human tooling interop?
- Empirical failure rates of fully autonomous greenfield builds (long-horizon agent studies) were not researched; recommend a follow-up search.
- PRD/ADR/EARS primary sources not fetched this session.

## Sources

- https://scalingintelligence.stanford.edu/blogs/codemonkeys (date n/a)
- https://arxiv.org/pdf/2510.02387 (CWM, 2025-10)
- https://arxiv.org/pdf/2505.23604 (Satori-SWE, 2025-05)
- https://arxiv.org/pdf/2504.07164 (R2E-Gym, 2025-04)
- https://arxiv.org/pdf/2604.01496 (SWE-ZERO/HERO, 2026-04)
- https://arxiv.org/html/2406.07791v1 (position bias, 2024-06)
- https://arxiv.org/pdf/2410.21819 (self-preference, 2024-10)
- https://arxiv.org/html/2410.02736v1 (CALM, ICLR 2025)
- https://arxiv.org/pdf/2412.05579 (LLM-judge survey, 2024-12)
- https://grepture.com/blog/llm-as-a-judge-bias (blog, secondhand figures)
- https://codemyspec.com/blog/github-spec-kit-guide ; https://docs.plannotator.ai/frameworks/github-spec-kit
- https://www.verdent.ai/guides/agents/kiro-spec-driven-development (checked 2026-08-25)
- https://codemyspec.com/blog/bmad-method-explained ; https://www.augmentcode.com/guides/bmad-method-ai-development
- https://www.verdent.ai/guides/coding/spec-driven-development ; https://hackernoon.com/spec-driven-development-has-a-blind-spot-the-seams ; https://sys0.substack.com/p/agent-native-software-engineering
