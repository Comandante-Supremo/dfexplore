# dfexplore: dark factory documentation

A "dark factory" is an autonomous system that runs agentic exploration, updates and builds with humans only at the review boundary. This repo holds the research and the plan for building one on a home server with self-hosted CI runners, using Claude.

- [DESIGN.md](DESIGN.md): the design decisions, each with an evidence pointer. Start here.
- [BUILD_PLAN.md](BUILD_PLAN.md): phases with exit tests, owner decisions, open items.
- [EVIDENCE.md](EVIDENCE.md): every number the design relies on, with its scope, source and verification status.
- [examples/harness-compat.md](examples/harness-compat.md): worked example (not core design) of an `update` goal: keeping a multi-harness messaging tool compatible with upstream CLIs.
- Research (dated 2026-10-07; each file tags unverified claims):

| Doc | Topic |
|---|---|
| [01-prior-art-landscape](research/01-prior-art-landscape.md) | Renovate, Dependabot, coding agents, StrongDM/Stripe factories; build vs adopt |
| [02-claude-agent-stack](research/02-claude-agent-stack.md) | Headless Claude Code, Agent SDK, hooks, auth/terms, models and pricing |
| [03-verification-and-conformance](research/03-verification-and-conformance.md) | Verifier design, CLI contract testing, anti-gaming, judge reliability |
| [04-state-and-long-running](research/04-state-and-long-running.md) | Durable state, goal store schema, state machine, stall detection |
| [05-dependency-updates-and-constraints](research/05-dependency-updates-and-constraints.md) | Resolution decisions, constraints registry, canary, bisect, policy file |
| [06-options-and-autonomous-build](research/06-options-and-autonomous-build.md) | Parallel option scoring, conversation-to-spec, sub-goal DAG, rejection loop |
| [07-infrastructure-and-escalation](research/07-infrastructure-and-escalation.md) | Forge/CI, sandboxes, control plane, triggers, async escalation |

Wave two (primary-source verification) and audits:

- `research/wave2/W2-01` to `W2-08`: second-pass research that corrects and deepens the docs above (harness CLI contracts, Claude SDK, dependency-update literature, verification and judges, long-horizon agents, options and builds, update tooling, factory prior art).
- `research/wave2/audit/A-01` to `A-08`: independent audits of each wave-two doc, run on a different model, with accept/modify/reject decisions for every proposed edit.
- Precedence: where a wave-two doc or audit disagrees with a wave-one doc, the wave-two doc as amended by its audit wins. The wave-one docs have not been rewritten; `DESIGN.md`, `BUILD_PLAN.md` and `EVIDENCE.md` carry the accepted corrections. Known wave-two errors that audits rejected are listed in EVIDENCE.md E22, so do not copy numbers from the W2 docs without checking the audit.

Wave three (full-text paper reading, arXiv) and verifiers:

- `research/wave3/R3-01` to `R3-05`: readers that opened the actual arXiv PDFs behind the wave-one and wave-two numbers (judges and reward hacking, dependency-update studies, options and builds, long-horizon agents, agent-authored PRs and incidents).
- `research/wave3/audit/V-01` to `V-03`: independent re-verification (different model) of the readers' highest-impact numbers, with safe-to-apply lists.
- Precedence: wave three as amended by its verifiers beats wave two beats wave one. `EVIDENCE.md` carries the reconciled numbers and lists what was corrected or dropped (E22).

Wave four (build references):

- `research/wave4/W4-01-oss-build-references.md`: component-by-component survey of open-source projects whose code, schemas or patterns the build can adopt, borrow or read, with licenses and last-commit dates; about 45 repos were cloned and read. Includes a "copy before Phase 0" list and the components with no good reference.

Reading note: sourcing is uneven. Several researchers could not fetch primary documentation pages and relied on search summaries. Treat anything tagged UNVERIFIED, `[U]` or `[BK]` as a hypothesis until checked (see EVIDENCE.md E23 for what is still unreachable).
