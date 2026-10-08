# 01 - Prior art landscape for a dark factory

Research date: 2026-10-07. Method: web search snippets only (WebFetch of primary pages was not performed), so most claims rest on secondary sources. Items marked UNVERIFIED were not confirmed against a primary source. Many cited pages carry no visible publication date; where known, dates are given.

## Summary

- "Dark factory" is an established but contested term in 2026 for a pipeline where agents build and ship with little or no human code review. The strongest public exemplars are StrongDM's Software Factory (no human writes or reviews code) and Stripe's Minions (blueprint workflows, human review retained).
- Two ideas recur across serious implementations: (1) verification must be held out from the coding agent (holdout scenarios, digital twins), and (2) workflows mix deterministic nodes with agent nodes rather than letting the agent drive everything.
- Renovate and Dependabot already model most of the constraints registry: ignore/pin, cooldown/release-age, grouping, automerge on green CI. Neither records reason/evidence/upstream link/revisit trigger; that is the gap the factory fills.
- General coding agents (Claude Code Actions, Codex cloud, Copilot agent, Devin, OpenHands, SWE-agent, Sweep, Factory) are execution engines with a trigger and a PR as output. None supplies a goal object with machine-checkable criteria, a decisions log, and an escalation rule. That layer is the factory's value.
- Recommendation: adopt Renovate (detection and PR mechanics) and Claude Code headless/Agent SDK (execution); build the goal model, constraints registry, scoring/judging, conformance suite, and orchestration state yourself. Borrow patterns from StrongDM and Stripe; do not adopt a closed platform as the core.

## Findings

### 1. The term "dark factory" and lights-out software

- Definition: borrowed from lights-out manufacturing; a pipeline where agents take features, refactors, tests, bug fixes end to end. Humans define intent and review outcomes rather than code. Dan Shapiro's five-level autonomy ladder puts it at the top (as reported in Jan 2026) ([Pulumi](https://www.pulumi.com/blog/dark-factory-pattern-pulumi-autonomous-iac/), [OpenHands blog](https://www.openhands.dev/blog/what-is-an-ai-dark-factory), [Tessl pattern](https://tessl.io/patterns/agentic-development-workflow/dark-factory/)). Primary Shapiro post not read: UNVERIFIED.
- Skeptical view: Tessl calls it real for a handful of small elite teams and contested as a general model; Osmani argues the review gate / human judgment is the hard-to-scale bottleneck ([Osmani](https://addyosmani.com/blog/software-factories/)).
- OpenHands' take: autonomy comes from redesigning the process so fewer steps need judgment, and depends on reproducible environments, machine-checkable verification, and bounded workflows, not just better models ([OpenHands](https://www.openhands.dev/blog/what-is-an-ai-dark-factory)). This matches the GOAL abstraction directly.
- Much coverage is vendor/blog content (MindStudio, Pulumi); adoption figures are self-reported.

### 2. StrongDM Software Factory (closest prior art)

- Charter forbids humans writing or reviewing code; correctness judged by behavior ([Ry Walker](https://rywalker.com/research/strongdm-factory)). Three people, about three months (same source). Feb 2026 coverage: [Simon Willison](https://simonwillison.net/2026/feb/7/software-factory/).
- Scenarios: end-to-end user stories stored outside the coding agent's view (holdout sets); quality measured probabilistically as fraction of trajectories satisfying the user ([Willison](https://simonwillison.net/2026/feb/7/software-factory/)). Rationale: agents that control code and tests game them (return true, rewrite tests) ([Ry Walker](https://rywalker.com/research/strongdm-factory)).
- Digital Twin Universe: behavioral clones of Okta, Jira, Slack, Google Docs/Drive/Sheets that model state, errors, auth, rate limits, enabling high-volume testing ([Letsdatascience](https://www.letsdatascience.com/news/strongdm-builds-software-factory-with-agentic-testing-c5aae799)).
- Attractor repo is spec-only (markdown) fed to an existing agent; CXDB is a context store. StrongDM reportedly acquired by Delinea 2026-03-05 (UNVERIFIED, single secondary source), so long-term direction is uncertain.
- Borrow: holdout acceptance scenarios, twins of external services (very relevant to CLI/API conformance), probabilistic satisfaction scoring. Avoid: zero-review as a default; make it a per-repo autonomy setting.
- Further reading on the architecture: [Aktagon](https://signals.aktagon.com/articles/2026/03/dark-factory-architecture-how-level-4-actually-works/), [Blake Crosley](https://blakecrosley.com/pl/blog/the-dark-factory-verification-layer) (not read in detail).

### 3. Stripe Minions

- Blueprints: workflows mixing deterministic nodes (lint, git, PR template, test gates) the model cannot override with agent nodes for open-ended work ([MindStudio](https://www.mindstudio.ai/blog/stripe-minions-blueprint-architecture-deterministic-agentic-nodes), [ByteByteGo](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)).
- Reported 1,300+ PRs/week, human review retained; built on a fork of Block's Goose; tooling via an MCP server (tool count disputed: 400 vs about 500) ([Ry Walker](https://rywalker.com/research/stripe-minions), [ByteByteGo](https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs)). Primary Stripe posts not read: UNVERIFIED. Trigger: Slack emoji (secondary source, [jangwook.net](https://jangwook.net/en/blog/en/stripe-minions-autonomous-coding-agents-1300-prs/)).
- Borrow: the blueprint model (deterministic skeleton, bounded agent steps, capped CI rounds). This is the best template for factory goal runners.

### 4. Dependency-update bots

**Renovate** (self-hostable, open source; also runs on GitLab/Gitea etc.)
- Trigger: schedule/cron plus Dependency Dashboard; PR per update, grouped via packageRules. Autonomy: per-rule `automerge`, `automergeType: branch` merges only if CI passes, otherwise opens a PR ([example config](https://git.nemunai.re/iac/renovate-config/compare/efbd35396f9c6581c1c63e85054fb29461f85e3e..30056f54e5ec0ea95a4139acdebb9b15b3f79195)). `dependencyDashboardApproval` gates specific updates behind human approval.
- Pin/ignore: `packageRules` with `matchPackageNames` can set `automerge: false`, `enabled: false`, or allowedVersions; `ignoreDeps` blocks entirely (from general knowledge, verify in docs).
- Supply-chain gating: `minimumReleaseAge` creates a pending `renovate/stability-days` status so automerge waits; `internalChecksFilter: strict` delays branch creation ([docs diff](https://git.shivering-isles.com/github-mirror/renovatebot/renovate/-/commit/135e6cd078c703c1b160d92d690dea6efbbf93ba.diff)). Pitfalls: digest updates bypassed age checks before 44.3.1 ([CVE-2026-88884](https://app.opencve.io/cve/CVE-2026-88884)); in v43 `timestamp-required`, releases lacking timestamps stay pending forever so automerge never fires ([GitLab MR](https://gitlab.com/gitlab-com/public-sector/pipeline/-/merge_requests/129)).
- State: stored in the repo config, the PR/branch, and the dashboard issue. No structured reason/evidence/revisit field beyond comments.
- Verification: whatever CI you have. No semantic compatibility check.
- Borrow: packageRules layering, release-age gating, dashboard as human-visible state, branch-automerge on green. Avoid: relying on it for "adapt the code" work; it only bumps versions.

**Dependabot** (GitHub-hosted; the version-update engine is not realistically self-hosted, verify)
- `.github/dependabot.yml` only, one file per repo. `ignore` by dependency name with optional version/update-type; `cooldown` (default-days and per-semver-level, include/exclude, version updates only, not security updates); `groups` (version updates, plus separate security-update groups) ([skill file summary](https://git.cynarski.dev/Awesome/awesome-copilot/src/branch/marketplace/skills/dependabot/SKILL.md), [Apache INFRA](https://infra.apache.org/dependabot.html), [DEV](https://dev.to/jpoehnelt/automatically-approving-and-merging-dependabot-pull-requests-2i9j)). Secondary sources; verify on docs.github.com.
- No native auto-merge: pair with a workflow using `dependabot/fetch-metadata` and `gh pr merge --auto`, restricted to patch (same sources). `@dependabot ignore` comments store ignores in PR interaction. Reported deprecation of merge/close/reopen commands from Jan 2026 (UNVERIFIED).
- Borrow: cooldown that exempts security fixes; the ignore-with-comment convention. Avoid: ignores that are silent and have no revisit trigger, which is exactly what the registry must fix.

### 5. Autonomous coding agents and platforms

**Claude Code headless / GitHub Actions (native base for the factory).** `claude -p` runs the full agent loop and exits with a status; `--bare` skips auto-discovery of hooks, skills, MCP, CLAUDE.md for reproducible CI; guardrails `--max-turns`, `--allowedTools`, `--max-budget-usd` ([SFEIR](https://institute.sfeir.com/en/claude-code/claude-code-headless-mode-and-ci-cd/), [hidekazu-konishi](https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html)). `claude-code-action` is built on the Agent SDK; triggers are @claude mentions, PR open, and other repository events. Self-hosted-runner behavior of the action: UNVERIFIED (check README). State: stateless per run; whatever you persist externally.

**OpenAI Codex cloud.** Container with repo checkout, setup script (with internet), agent phase with internet off by default (configurable), secrets removed before the agent phase (third-party claim), `AGENTS.md` for lint/test commands; outputs a diff and PR ([OpenAI docs](https://developers.openai.com/codex/cloud/environments), [intro](https://openai.com/index/introducing-codex/)). Cloud-only; not self-hostable. Borrow: setup-vs-agent phase split with secrets stripped.

**GitHub Copilot coding agent.** Issue/PR-assigned, runs in Actions. Since 2025-10-28 supports self-hosted runners via ARC only, Ubuntu x64 only, persistent runners not recommended, built-in firewall incompatible so you must supply network controls ([changelog](https://github.blog/changelog/2025-10-28-copilot-coding-agent-now-supports-self-hosted-runners), [docs](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment)). Org-level runner config split between code review and cloud agent in July 2026 (secondary: [releases.sh](https://releases.sh/release/rel_5CecWVbNraaqeKFWn3DDt-firewall-setup-steps-head-branch-instructions-for-copilot-review)). Also GitHub's `gh-aw` agentic-workflows tooling has ARC guidance ([gh-aw](https://github.github.com/gh-aw/guides/arc-dind-copilot-agent/)); worth a closer look as prior art for markdown-defined agentic workflows (not evaluated). Borrow: ephemeral ARC runners per run.

**Devin (Cognition).** Closed, cloud-hosted; self-hosting enterprise-only; pricing and benchmarks conflict across sources ([OpenHands comparison](https://www.openhands.dev/blog/devin-ai-alternatives), [toolhalla](https://toolhalla.ai/blog/devin-vs-openhands-vs-swe-agent-2026)). Avoid as a core dependency (also conflicts with "built on Claude").

**OpenHands.** MIT core, local/self-hosted/cloud, LiteLLM/BYOK, SDK, can drive Claude Code/Codex/Gemini CLI via ACP, Git integrations ([OpenHands](https://www.openhands.dev/blog/devin-ai-alternatives); author has a stake). Closest self-hostable platform; candidate for sandboxed runtime but adds a second agent stack.

**SWE-agent.** MIT, CLI, research/benchmark oriented, lacks admin/audit/SDK ([aicoolies](https://aicoolies.com/comparisons/openhands-vs-devin-vs-swe-agent)). Useful as an eval harness reference, not a production orchestrator.

**Sweep.** GitHub App turning issues into PRs ([aiagentrank](https://aiagentrank.io/compare/factory-ai-vs-sweep)); current status/maintenance UNVERIFIED.

**Factory (Droids).** Task in natural language, plans/codes/tests/ships across terminal, IDE, browser, Slack; adjustable autonomy supervised to autonomous ([Factory](https://factory.ai/product/droids)). Closed SaaS; borrow the idea of a per-scope autonomy dial.

## Comparison table

| System | Trigger | Autonomy / HITL | Verification | State | Self-host | Borrow / avoid |
|---|---|---|---|---|---|---|
| Renovate | cron, dashboard | per-rule automerge; dashboard approval | repo CI; release-age gate | config + PRs + dashboard issue | Yes | Borrow rules, age gate. No reasons/evidence |
| Dependabot | schedule | ignore/cooldown; automerge via workflow | repo CI | dependabot.yml, PR comments | No (GitHub) | Borrow security exemption in cooldown |
| StrongDM factory | spec/scenarios | no human code review | holdout scenarios, twins | specs, CXDB | Own infra | Borrow holdouts, twins |
| Stripe Minions | Slack emoji etc. | human review of PR | deterministic gates + CI (2 rounds) | blueprint run | Internal | Borrow blueprints |
| Claude Code headless/Action | CLI, @claude, events | allowlist, budget, turns | you design | none built in | Yes (runner) | Execution engine |
| Codex cloud | UI, @codex | PR for human | tests via AGENTS.md | per-task container | No | Setup/agent phase split |
| Copilot agent | issue assign | PR review required | CI | PR | ARC runners | Ephemeral runners |
| Devin | chat, Slack | supervised | internal | session | Enterprise only | Avoid as core |
| OpenHands | UI, API, GitHub | configurable | pluggable | event log | Yes | Optional sandbox |
| SWE-agent | CLI | autonomous | SWE-bench style | trajectories | Yes | Eval harness ideas |
| Sweep / Factory | issue / multi-surface | PR / dial | CI / internal | PR / internal | No / No | Autonomy dial |

## Implications for the dark factory

1. **Do not build a PR-bump bot.** Run Renovate self-hosted as the detection layer for goal type 1. Treat its output (an update PR plus red/green CI) as an input event to a factory goal, not the end state. Mirror its packageRules into the constraints registry, with the registry as source of truth and a generator that writes Renovate/Dependabot config from it, so every pin has reason, evidence, upstream link, revisit trigger. Neither bot has this; it is the differentiator.
2. **Auto-merge only on a stronger gate than repo CI.** Use `automergeType: branch`-style semantics plus a release-age gate and the factory's own success criteria. Copy Dependabot's rule that security fixes skip cooldown. Beware the Renovate failure modes found above: pending-forever checks and digest bypass; add a watchdog that flags PRs stuck in pending beyond N days (escalation rule).
3. **Adopt the blueprint pattern (Stripe) as the goal runner shape:** deterministic nodes (checkout, run conformance suite, lint, push, write log entries) around bounded agent nodes (diagnose, adapt, workaround), with capped CI rounds and a budget. Outcomes map to leaf nodes: adapt, pin, constrain, workaround, escalate.
4. **Hold out acceptance from the coding agent (StrongDM).** The agent that changes code must not see or edit the conformance scenarios or judged scoring rubric. For the multi-harness messaging example, build a CLI contract suite with recorded and simulated twins of Claude Code, Codex, Kimi, OpenCode CLIs, run against new versions in a scheduled canary job; this detects upstream change before an update goal even fires. For goal type 3, same principle: automated metrics plus an independent judge, never self-scored.
5. **Execution engine = Claude Code headless / Agent SDK** on your own runners, ephemeral per run (Copilot's ARC lesson: non-persistent runners), setup phase with network and secrets separated from agent phase without (Codex's lesson), `--bare` for reproducibility, budget/turn/tool limits. Do not adopt Devin/Codex cloud/Factory as core: cloud-only, and not Claude.
6. **State is yours.** None of the surveyed tools has a progress log, decisions log, or checkpoints as first-class objects. Keep these in git (markdown/JSONL in a state branch or repo) so they are diffable and reviewable; link PRs to them. Renovate's dashboard issue is a good human-facing view to imitate.
7. **Autonomy dial per repo** (Factory's idea; StrongDM and Stripe are the two endpoints). Default to Stripe-like (auto-merge on gates, human sees PRs) and allow StrongDM-like zero-review only for repos with a strong holdout suite.
8. **Greenfield (type 4):** no surveyed system proposes its own definition of done for user acceptance; StrongDM comes closest via scenarios. Novel; design carefully.

### Build vs adopt per component

| Component | Decision | Reason |
|---|---|---|
| Update detection (deps) | Adopt Renovate, self-hosted | Mature, rich rules, self-hostable |
| Upstream CLI/API change detection | Build | No prior art for multi-harness CLI contracts |
| Execution agent | Adopt Claude Code headless / Agent SDK | Constraint: built on Claude |
| Sandbox/runner | Adopt own CI runners; consider OpenHands runtime only if isolation falls short | Already exist |
| Goal model, constraints registry, logs | Build | Gap in all tools |
| Workflow orchestration | Build thin (blueprint pattern); evaluate gh-aw | Small state machine, low risk |
| Verification (holdouts, twins, judge) | Build | Core differentiator |
| Auto-merge | Adopt GitHub/Gitea auto-merge + branch protection | Commodity |
| Eval harness for option testing (type 3) | Build, borrow SWE-agent/SWE-bench ideas | Domain specific |

## Open questions

- Which forge hosts the repos (GitHub, Gitea, GitLab)? It decides Dependabot/Copilot/gh-aw relevance and native auto-merge.
- Does `claude-code-action` work cleanly on self-hosted/ARC runners, or should the factory call `claude -p` directly from its own orchestrator? (UNVERIFIED.)
- How should the registry be stored: in-repo file (visible, reviewable) vs. central store (cross-repo queries)? Probably in-repo with a central index.
- Revisit triggers: how to detect "upstream issue fixed" automatically (poll linked issue state, release notes parse, or just re-run the canary on each new version)?
- StrongDM and Stripe primary sources should be read in full before final design; both are known here mainly through secondary summaries.
- Licensing and maintenance health of Sweep, OpenHands, SWE-agent in late 2026 not checked.

## Sources

- https://www.pulumi.com/blog/dark-factory-pattern-pulumi-autonomous-iac/
- https://www.openhands.dev/blog/what-is-an-ai-dark-factory
- https://tessl.io/patterns/agentic-development-workflow/dark-factory/
- https://addyosmani.com/blog/software-factories/
- https://simonwillison.net/2026/feb/7/software-factory/
- https://rywalker.com/research/strongdm-factory
- https://www.letsdatascience.com/news/strongdm-builds-software-factory-with-agentic-testing-c5aae799
- https://signals.aktagon.com/articles/2026/03/dark-factory-architecture-how-level-4-actually-works/
- https://blakecrosley.com/pl/blog/the-dark-factory-verification-layer
- https://www.mindstudio.ai/blog/stripe-minions-blueprint-architecture-deterministic-agentic-nodes
- https://blog.bytebytego.com/p/how-stripes-minions-ship-1300-prs
- https://rywalker.com/research/stripe-minions
- https://jangwook.net/en/blog/en/stripe-minions-autonomous-coding-agents-1300-prs/
- https://git.shivering-isles.com/github-mirror/renovatebot/renovate/-/commit/135e6cd078c703c1b160d92d690dea6efbbf93ba.diff
- https://app.opencve.io/cve/CVE-2026-88884
- https://gitlab.com/gitlab-com/public-sector/pipeline/-/merge_requests/129
- https://git.nemunai.re/iac/renovate-config/compare/efbd35396f9c6581c1c63e85054fb29461f85e3e..30056f54e5ec0ea95a4139acdebb9b15b3f79195
- https://infra.apache.org/dependabot.html
- https://git.cynarski.dev/Awesome/awesome-copilot/src/branch/marketplace/skills/dependabot/SKILL.md
- https://dev.to/jpoehnelt/automatically-approving-and-merging-dependabot-pull-requests-2i9j
- https://institute.sfeir.com/en/claude-code/claude-code-headless-mode-and-ci-cd/
- https://hidekazu-konishi.com/entry/claude_code_cicd_and_headless_automation.html
- https://developers.openai.com/codex/cloud/environments
- https://openai.com/index/introducing-codex/
- https://github.blog/changelog/2025-10-28-copilot-coding-agent-now-supports-self-hosted-runners
- https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment
- https://releases.sh/release/rel_5CecWVbNraaqeKFWn3DDt-firewall-setup-steps-head-branch-instructions-for-copilot-review
- https://github.github.com/gh-aw/guides/arc-dind-copilot-agent/
- https://www.openhands.dev/blog/devin-ai-alternatives
- https://toolhalla.ai/blog/devin-vs-openhands-vs-swe-agent-2026
- https://aicoolies.com/comparisons/openhands-vs-devin-vs-swe-agent
- https://aiagentrank.io/compare/factory-ai-vs-sweep
- https://factory.ai/product/droids
