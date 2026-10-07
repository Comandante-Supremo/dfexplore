# dfexplore: dark factory documentation

A "dark factory" is an autonomous system that runs agentic exploration, updates and builds with humans only at the review boundary. This repo holds the research and the plan for building one on a home server with self-hosted CI runners, using Claude.

- [BUILD_PLAN.md](BUILD_PLAN.md): agreed design, phased build plan, open decisions, and the second-pass research list. Start here.
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
- Precedence: where a wave-two doc or audit disagrees with a wave-one doc, the wave-two doc as amended by its audit wins. The wave-one docs have not been rewritten; `BUILD_PLAN.md` (v2) carries the accepted corrections. Known wave-two errors that audits rejected are listed in BUILD_PLAN section 6, so do not copy numbers from the W2 docs without checking the audit.

Reading note: sourcing is uneven. Several researchers could not fetch primary documentation pages and relied on search summaries. Treat anything tagged UNVERIFIED, `[U]` or `[BK]` as a hypothesis until checked (see the second-pass list in the build plan).
