# 02 - Claude agent stack: building blocks for the dark factory

Researched 2026-10-07. Sources are official Anthropic docs unless noted. Doc pages were fetched on 2026-10-07 and carry no per-page date, so the fetch date is the citation date. Anything not verified is marked UNVERIFIED.

## Summary

- Everything the factory needs exists as a headless, scriptable primitive. `claude -p` (CLI), the Agent SDK (Python/TypeScript) and Managed Agents (hosted, beta) all drive the same Claude Code agent loop or an equivalent one.
- Recommended core: Agent SDK or `claude -p --bare` running on the home server, authenticated by API key. Managed Agents with a self-hosted sandbox is the one hosted option worth a pilot.
- Hard controls exist at every level: permission modes and rules, hooks, `--max-turns`, `maxBudgetUsd` and `--max-budget-usd`, subagent depth/concurrency caps, session budgets (Managed Agents), and OpenTelemetry.
- Machine-checkable success criteria map directly onto three existing features: `/goal` (Stop-hook evaluator), Stop hooks (deterministic scripts), and Managed Agents Outcomes (separate-context grader with a rubric). Prefer deterministic gates (CI, tests) over model graders wherever possible.
- Subscription vs API: the policy moved on 2026-10-07 (see Auth). For an unattended, always-on factory, use API key billing. Subscription OAuth tokens are for "ordinary individual use". The newly announced Max/Team monthly API credits can offset API spend.
- Current models: Opus 5.5 (default, $4/$20 per MTok), Sonnet 5.5 ($2/$10), Haiku 5.5 ($0.10/$0.50 up to 100K prompt tokens), Fable 5.1 ($10/$50, top capability). All have a 1M context window.

## Findings

### 1. Claude Code headless (`claude -p`)

- `-p` runs non-interactively; exit code is 0 on success and non-zero on failure. Failures inside the run (for example missing auth) are printed as the result on stdout. [Headless docs](https://code.claude.com/docs/en/headless)
- Output formats: `text`, `json` (result, `session_id`, `total_cost_usd`, per-model cost), and `stream-json` (newline-delimited events; use with `--verbose --include-partial-messages`). `--json-schema` yields validated output in `structured_output`. [Headless](https://code.claude.com/docs/en/headless)
- `stream-json` carries the `system/init` event with `mcp_servers`, `mcp_server_errors`, `plugins` and `plugin_errors`, so CI can fail fast when a harness component did not load. It also carries `system/api_retry` events, `permission_denied` messages and `parent_tool_use_id` for subagent attribution. [Headless](https://code.claude.com/docs/en/headless)
- `--bare` skips auto-discovery of hooks, skills, plugins, MCP servers, memory and CLAUDE.md. It is recommended for scripted calls and will become the default for `-p`. Bare mode never reads OAuth or the keychain, so it needs `ANTHROPIC_API_KEY` or `apiKeyHelper` (cloud-provider credentials still work). Bare mode also disables background tasks. [Headless](https://code.claude.com/docs/en/headless)
- Without `--bare`, `-p` runs the hooks in `.claude/settings.json` and connects `.mcp.json` servers with no trust dialog. For a factory that checks out untrusted dependency repos, this is a prompt-injection and config-injection surface. [Headless](https://code.claude.com/docs/en/headless)
- Resume: `--continue`, `--resume <session_id>` (works across directories on one machine since v2.1.223), or `--resume <path-to-.jsonl>`. A resumed run reports the whole session's cumulative cost. Sessions are local files, so the factory's external state must not depend on them. [Headless](https://code.claude.com/docs/en/headless), [cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
- Permission flags: `--allowedTools` with rule syntax (for example `Bash(git diff *)`), `--permission-mode`, and `--permission-prompts none`, which denies anything that would prompt and tells Claude not to retry (v2.1.259+). [Headless](https://code.claude.com/docs/en/headless)
- SIGTERM gives exit 143 and leaves the turn unfinished. SIGINT or SDK `interrupt()` ends the turn cleanly. Background subagents keep `-p` open until done, capped at a 10-minute idle ceiling. [Headless](https://code.claude.com/docs/en/headless)
- `/goal <condition>` works in `-p` and loops until a small fast model judges the condition met, impossible, or a fatal error occurs. The evaluator only sees what is in the transcript, so Claude must surface proof such as a test exit code. Add a turn bound in the condition text. It is a session-scoped prompt-based Stop hook. [Goal](https://code.claude.com/docs/en/goal)

### 2. Agent SDK (Python / TypeScript)

- It is the Claude Code binary as a library, with the same built-in tools, hooks, subagents, MCP, permissions, sessions and skill loading. [SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- Budget and turn caps: `maxBudgetUsd` / `max_budget_usd` (counts subagent spend, ends the query with `error_max_budget_usd`), `maxTurns`, and env vars `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` (default 3) and `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` (default 20). The TypeScript `env` option replaces the subprocess environment, so spread `process.env`. [Subagents](https://code.claude.com/docs/en/agent-sdk/subagents)
- Cost figures (`total_cost_usd`, `modelUsage`) are client-side estimates, not billing data. Use the Console Usage and Cost API for authoritative numbers. [Cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking)
- Branding: products built on the SDK may not be called "Claude Code" or "Claude Code Agent". Allowed: "Claude Agent" or "{YourAgent} Powered by Claude". [SDK overview](https://code.claude.com/docs/en/agent-sdk/overview)
- `claude-agent-sdk` is a harness only: you host and deploy it. The plain Messages API tool runner is a lighter option for narrow tool sets with no filesystem. [Claude API skill, cached 2026-10-06; no public URL]

### 3. Subagents, skills, hooks, MCP

- **Subagents** (`AgentDefinition`): own system prompt, `tools` / `disallowedTools`, `model`, `effort`, `maxTurns`, `permissionMode`, `skills`, `mcpServers`. Each starts with a fresh context and only its final message returns to the parent. Subagents can be resumed via `agentId`. They default to background execution. `omitClaudeMd` is TypeScript-only. [Subagents](https://code.claude.com/docs/en/agent-sdk/subagents)
- **Dynamic workflows** (`Workflow` tool, TypeScript SDK v0.3.149+): a script orchestrates dozens to hundreds of agents outside the conversation context. This is the natural fit for parallel option testing. [Subagents](https://code.claude.com/docs/en/agent-sdk/subagents). I did not read the workflows page, so its limits are UNVERIFIED.
- **Hooks:** 33 events (for example `PreToolUse`, `PermissionRequest`, `PostToolUse`, `Stop`, `SubagentStop`, `TaskCompleted`, `PreCompact`, `SessionEnd`). Handler types are `command`, `http`, `mcp_tool`, `prompt`, and an experimental `agent` type. Exit code 2 blocks. A `Stop` hook that returns `decision: block` forces the agent to keep working. This is how "do not stop until CI is green" is enforced deterministically. [Hooks](https://code.claude.com/docs/en/hooks). Only the first 100K characters of that page were read; hook semantics beyond the list are UNVERIFIED.
- **Skills:** on-demand instruction bundles that load only when invoked. They can be invoked in `-p` and GitHub Actions via `/skill-name`, and packaged as plugins. They are the right place for per-goal-type playbooks instead of a bloated CLAUDE.md. [Costs](https://code.claude.com/docs/en/costs), [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- **MCP:** servers are deferred-loaded by default (tool search), so only names enter context until used. CLI tools such as `gh` are still more context-efficient than MCP. [Costs](https://code.claude.com/docs/en/costs)

### 4. Permission modes and unattended safety

- Modes: `default`, `acceptEdits`, `plan`, `auto` (a classifier model reviews actions), `dontAsk` (deny anything not pre-approved, intended for locked-down CI), and `bypassPermissions`, which the docs say is for isolated containers or VMs only and should be run as a non-root user. Deny rules apply in every mode. Allow rules have no effect in `bypassPermissions`. [Permission modes](https://code.claude.com/docs/en/permission-modes)
- Auto mode blocks by default several action classes, including force-push-like history rewrites of pushed commits, writing to session transcripts, reading host credentials and touching sibling containers. `claude auto-mode defaults` prints the rules and `autoMode.environment` adds trusted repos and services. It is the built-in default from v2.1.283 for interactive sessions; for `-p`, pass it explicitly. [Permission modes](https://code.claude.com/docs/en/permission-modes), [Headless](https://code.claude.com/docs/en/headless)
- Auto mode needs a supported model (Sonnet 5 or later, Opus 4.7 or later, Haiku 5.5, Fable). [Permission modes](https://code.claude.com/docs/en/permission-modes)
- Recommended unattended combination: `--permission-mode auto --permission-prompts none` plus explicit allow/deny rules, inside a container.

### 5. GitHub Actions and self-hosted CI

- `anthropics/claude-code-action@v1`: interactive mode (`@claude` mentions) and automation mode (when `prompt` is set, on any event including `schedule`). Inputs include `claude_args` (any CLI flag, for example `--max-turns`, `--model`, `--allowedTools`), `plugins`, `settings`, and `use_bedrock` / `use_vertex` / `use_foundry`. [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Auth options: `ANTHROPIC_API_KEY`, `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`), or workload identity federation through OIDC (`anthropic_federation_rule_id` and related inputs), which avoids a static secret. For org-wide shared secrets the docs recommend an API key over an OAuth token. [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Gotcha: commits pushed with the default `GITHUB_TOKEN` do not trigger CI. Let the action authenticate as the Claude GitHub App, or pass an app token. Bots are rejected unless listed in `allowed_bots`, and write access is required for issue/PR triggers. [GitHub Actions](https://code.claude.com/docs/en/github-actions)
- Self-hosted runners: the docs say the action "runs on GitHub-hosted runners" for billing. I found no official statement of self-hosted runner support or requirements. The action's FAQ exposes `path_to_claude_code_executable` and `path_to_bun_executable` for custom environments, which implies it can be made to work. Self-hosted support is UNVERIFIED; test it. [FAQ](https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md)
- Alternative that avoids the action entirely: call `claude -p --bare` or the SDK from your own runner job. That gives full control and no dependence on the action's runner assumptions.

### 6. Managed Agents (hosted harness, beta)

- Beta header `managed-agents-2026-04-01`; public beta launched 2026-04-08. Objects: Agent (versioned config), Environment, Session, Events (SSE stream). Features: persisted agents, outcomes, multiagent threads, memory stores, vaults, webhooks, scheduled deployments. [Overview](https://platform.claude.com/docs/en/managed-agents/overview), [reference](https://platform.claude.com/docs/en/managed-agents/reference). Launch date is from search results ([Claude Platform release notes](https://platform.claude.com/docs/en/release-notes/overview)).
- **Self-hosted sandbox:** orchestration and the model loop run on Anthropic's side, and tool execution runs in your environment via a worker (`ant beta:worker poll`). The host needs `/bin/bash` and outbound HTTPS only. Tool inputs and outputs still flow to Anthropic. Only `memory_store` resources are supported there, not `file` or `github_repository` resources. This is a direct fit for a home server. [Self-hosted sandboxes](https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes)
- **Outcomes:** `user.define_outcome` takes a markdown rubric (text or file) and `max_iterations` (default 3, max 20). A separate-context grader returns `satisfied`, `needs_revision`, `max_iterations_reached`, `failed` or `interrupted`. This is the closest built-in match to "machine-checkable success criteria, factory proposes DoD". The grader judges by rubric, so it is not a replacement for running tests. [Outcomes](https://platform.claude.com/docs/en/managed-agents/define-outcomes)
- **Session budgets:** hard dollar caps set at session creation (in cents). They are priced at list rates, including $0.08 per session-hour and $10 per 1,000 web searches. Enforcement is between model requests, so overshoot is bounded by one request per thread. The session goes idle with `budget_reached` and resumes if you raise the cap. Deployments copy the budget to each run. [Budgets](https://platform.claude.com/docs/en/managed-agents/budgets)
- Limits: 300 create requests per minute and 1,200 read requests per minute per org. Not eligible for Zero Data Retention or a BAA. Memory stores and "dreaming" are in limited research preview, or UNVERIFIED for access. [Reference](https://platform.claude.com/docs/en/managed-agents/reference), [Overview](https://platform.claude.com/docs/en/managed-agents/overview)
- Beta risk: the API surface can change between releases. Treat it as a replaceable executor behind the factory's own interface.

### 7. Context management, compaction, caching

- Claude Code auto-compacts near the context limit. Native 1M-context models compact at about 967K tokens by default. The window is tunable (100K-1M) through `CLAUDE_CODE_AUTO_COMPACT_WINDOW`, `/autocompact` or the `autoCompactWindow` setting. Compact instructions can go in CLAUDE.md. `PreCompact` / `PostCompact` hooks exist. [Model config](https://code.claude.com/docs/en/model-config), [Costs](https://code.claude.com/docs/en/costs)
- Long runs should not rely on compaction for fidelity. Externalize progress/decisions logs and checkpoints (the goal design already does this) and start fresh sessions per phase. Compaction of a large context is itself a large request. [Costs](https://code.claude.com/docs/en/costs)
- API-level tools: server-side compaction (beta `compact-2026-01-12`), context editing that clears old tool results (`context-management-2025-06-27`), and a memory tool. These are for custom API loops, not the Claude Code harness. [Claude API skill, cached 2026-10-06; no public URL]
- Caching: the Agent SDK and Claude Code use prompt caching automatically. The default TTL is 5 minutes on API key auth. Set `ENABLE_PROMPT_CACHING_1H=1`, or `CLAUDE_CODE_PROMPT_CACHE_TTL=1h` for the main loop, when gaps between turns exceed 5 minutes, which is common with CI waits. One-hour writes cost more. `/usage` shows cache hit and miss stats. [Cost tracking](https://code.claude.com/docs/en/agent-sdk/cost-tracking), [Costs](https://code.claude.com/docs/en/costs)
- Preserved thinking is a new constraint on custom harnesses: Fable 5.1, Opus 5.5 and Sonnet 5.5 thinking blocks are bound to the producing model and conversation, and edited history invalidates them. Keep transcripts append-only. This is mainly relevant if the factory writes its own Messages API loop. [Claude API skill, cached 2026-10-06; no public URL]

### 8. Models and pricing (per MTok, first-party API)

| Model | ID | In / Out | Notes |
|---|---|---|---|
| Fable 5.1 | `claude-fable-5-1` | $10 / $50 | Most capable. Thinking always on. Not available under zero data retention. |
| Opus 5.5 | `claude-opus-5-5` | $4 / $20 | Claude Code default. Effort defaults to `medium`. Cache reads $0.20. |
| Sonnet 5.5 | `claude-sonnet-5-5` | $2 / $10 | Effort defaults to `high` in the API (`medium` in Claude Code). Cache reads $0.20. |
| Haiku 5.5 | `claude-haiku-5-5` | $0.10 / $0.50 | Applies up to 100K-token prompts; $0.50 / $2.50 beyond. |

Source for the table: pricing and IDs from the Claude API reference cached 2026-10-06 (no public URL); see the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing) to re-verify. Aliases in Claude Code (`opus`, `sonnet`, `haiku`, `fable`) resolve to the current models on the Anthropic API but to older versions on other providers. Pin full IDs in factory config. [Model config](https://code.claude.com/docs/en/model-config)

- Effort levels: `low` to `max`. The Claude Code docs set the default to `medium` on Opus 5.5, Sonnet 5.5 and Haiku 5.5, `high` on most others. Thinking cannot be disabled on Opus 5.5, Sonnet 5.5, Haiku 5.5 or Fable. Forced `tool_choice` returns 400 on Fable 5.1, Opus 5.5 and Sonnet 5.5. Anthropic's guidance is to test the best model at lower effort before building multi-model cascades, because caches are per model. [Costs](https://code.claude.com/docs/en/costs), [Claude API skill, cached 2026-10-06]
- Per-subagent model: alias or full ID in `AgentDefinition.model`; `opusplan` runs Opus in plan mode and Sonnet to execute. `--fallback-model a,b` and a `fallbackModel` chain (max 3) apply on overload or server errors, not on rate-limit, auth or billing errors. [Model config](https://code.claude.com/docs/en/model-config)
- Anthropic's own cost guidance: average about $13 per developer per active day across enterprise deployments. Agent teams use roughly 7x the tokens of a standard session. Use Sonnet for teammates and Haiku for simple subagents. [Costs](https://code.claude.com/docs/en/costs)

### 9. Rate limits and cost control

- API: org-level spend limits and TPM/RPM tiers apply. Claude Code traffic lands in an auto-created "Claude Code" workspace where you can set a workspace rate limit and spend limit to protect other workloads. [Costs](https://code.claude.com/docs/en/costs)
- Layers available to the factory: per-run `--max-turns` and `--max-budget-usd`, per-session budgets in Managed Agents, per-workspace Console limits, subagent caps, plus OTel cost metrics for alerting. There is no global cross-run budget primitive, so the factory must keep its own ledger (read `total_cost_usd` per run).
- Under a subscription, limits are rolling 5-hour and weekly windows shared across models. Hitting them pauses `/goal` and long runs. Claude Code can wait and auto-continue after a reset (v2.1.234+). [Costs](https://code.claude.com/docs/en/costs), [Goal](https://code.claude.com/docs/en/goal)

### 10. Auth, subscription vs API, terms

- Credential precedence: cloud provider, then `ANTHROPIC_AUTH_TOKEN`, then `ANTHROPIC_API_KEY` (always used in `-p` when set), then `apiKeyHelper`, then `CLAUDE_CODE_OAUTH_TOKEN`, then Anthropic profiles/WIF, then `/login` OAuth. [Authentication](https://code.claude.com/docs/en/authentication)
- `claude setup-token` makes a one-year OAuth token tied to a Pro/Max/Team/Enterprise subscription. It can make model requests only (no Remote Control, no claude.ai connectors). It is not read in `--bare` mode. [Authentication](https://code.claude.com/docs/en/authentication)
- Terms: OAuth is "intended exclusively for purchasers of ... subscription plans" for "ordinary use of Claude Code". Developers building products or services, "including those using the Agent SDK", should use API keys. Advertised Pro/Max limits assume "ordinary, individual usage of Claude Code and the Agent SDK". Anthropic may enforce this without notice. [Legal and compliance](https://code.claude.com/docs/en/legal-and-compliance)
- A first-party, single-owner factory running on the owner's own home server is not a third party offering Claude.ai login to users, so it is not the main prohibited case. But a 24/7 autonomous fleet is arguably not "ordinary individual usage", so the subscription path is a risk. UNVERIFIED: no Anthropic page I found states whether a personal always-on automation is permitted. Ask Anthropic or use an API key.
- Policy timeline: a planned June 15, 2026 change moving SDK and `claude -p` usage to a separate credit was paused the same day. As of 2026-10-07, Agent SDK, `claude -p` and third-party apps still draw on subscription limits, and Max and Team plans now include monthly API credits that cover the Agent SDK, `claude -p`, the API and Managed Agents. Reported amounts are $100 (Max 5x), $200 (Max 20x), up to $500 pooled (Team). They land in a linked Console org. Whether they cover the $0.08/hr Managed Agents runtime is UNVERIFIED. [Support article](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan), [API credits for subscribers](https://platform.claude.com/docs/en/about-claude/api-credits-for-subscribers). Amounts come from search results; the support article text I fetched did not list them.
- Session and OAuth login expiry: a login within 3 days of expiry warns at startup, and unattended sessions stop making progress when it expires. Another reason to use API keys or WIF for unattended work. [Authentication](https://code.claude.com/docs/en/authentication)

### 11. Observability

- OpenTelemetry (off by default): `CLAUDE_CODE_ENABLE_TELEMETRY=1`, with `OTEL_METRICS_EXPORTER` and `OTEL_LOGS_EXPORTER` set to `otlp`, `prometheus` (scrape on :9464) or `console`. Metrics include `claude_code.cost.usage`, `claude_code.token.usage`, `commit.count`, `pull_request.count` and `code_edit_tool.decision`. Events include `tool_result`, `tool_decision`, `api_request`, `api_error`, `api_refusal`, `api_retries_exhausted`, `skill_activated`, `hook_registered` and `mcp_server_connection`. `prompt.id` groups all events from one prompt. [Monitoring](https://code.claude.com/docs/en/monitoring-usage)
- Traces are beta (`CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`, `OTEL_TRACES_EXPORTER`), with span hierarchy `interaction > llm_request / tool / hook`, subagent spans nested. An inbound `TRACEPARENT` in `-p` and SDK runs parents Claude's spans under the caller's trace, so the factory orchestrator can own one trace per goal round. [Monitoring](https://code.claude.com/docs/en/monitoring-usage)
- Prompts, responses and tool details are redacted by default; opt in with `OTEL_LOG_USER_PROMPTS`, `OTEL_LOG_TOOL_DETAILS` and so on. Repo-level settings cannot enable or redirect telemetry. Only the first 100K characters of that page were read. [Monitoring](https://code.claude.com/docs/en/monitoring-usage)
- Managed Agents exposes `session.usage`, `span.model_request_*` and `span.outcome_evaluation_*` events on its stream. [Reference](https://platform.claude.com/docs/en/managed-agents/reference)

## Recommended use per goal type

| Goal type | Primary mechanism | Why |
|---|---|---|
| 1. Dependency/upstream update | SDK or `claude -p --bare` in a worktree per attempt, driven by the factory's own loop. CI is the machine-checkable criterion. A `Stop` hook or the orchestrator re-prompts until CI is green or a bound is hit. | Deterministic gate. The cheap model class is enough for most bumps. |
| 1b. CLI contract tests across harnesses | Same, plus a test matrix job that runs each harness. | Contract tests are pass/fail. Keep model use to diagnosing failures. |
| 2. Long-running problem solving | SDK with fresh sessions per phase, external progress/decision logs, `/goal` or a Stop hook for continuation, `PreCompact` hook to flush notes. | Compaction is lossy; the state lives outside the context. |
| 3. Parallel option testing | Dynamic workflows or orchestrator-spawned parallel SDK runs, one worktree per option. Automated scores first, then a separate-context judge on a rubric. | Parallel fan-out is cheap to bound with `maxBudgetUsd` per branch. |
| 4. Greenfield from vision | Opus planner proposes DoD as rubric plus tests. User approves. Builder runs until tests pass. Managed Agents Outcomes is a candidate for the rubric loop. | The rubric is the acceptance artifact; rejection creates a new round. |

## Implications for the dark factory

1. **Make the factory's own state the source of truth, not Claude sessions.** Sessions are local `.jsonl` files and are lossy under compaction. Persist goal, logs, constraints registry and checkpoints in git or a database. Treat each Claude invocation as a stateless worker given a goal slice. Use `--resume` only within one round.
2. **Use API key auth for the factory.** The documented terms push developers building on the Agent SDK to API keys. OAuth tokens expire, are blocked in `--bare`, and 24/7 use may breach "ordinary individual usage". Put keys in a secrets store and set up an `apiKeyHelper` or WIF. Claim the Max/Team API credits if the account has them.
3. **Run `claude -p --bare` (or the SDK) from your own runner jobs, with explicit `--mcp-config`, `--settings`, `--agents` and `--append-system-prompt`.** This yields reproducible runs. It also avoids loading hooks and MCP servers from checked-out dependency repos. Treat `claude-code-action` on a self-hosted runner as optional until tested (UNVERIFIED).
4. **Autonomy per repo maps to permission config.** High autonomy: `--permission-mode auto --permission-prompts none` in a rootless container with deny rules (secrets paths, `git push --force`, branch protection bypass). Low autonomy: `plan` mode plus a human gate. `bypassPermissions` only inside disposable containers. Let GitHub branch protection plus required CI be the enforcement for "auto-merge on green".
5. **Hard bound every run:** `--max-turns`, `maxBudgetUsd`, subagent depth set to 1 or 2, concurrency set low, plus a factory-level daily ledger keyed off `total_cost_usd` and the Console workspace spend limit. Remember the SDK cost field is only an estimate.
6. **Put success criteria in code, not prose.** Gate order: deterministic checks (tests, lints, contract matrix), then a separate-context model judge against a rubric (Outcomes or a SDK verifier subagent with read-only tools), then human acceptance for greenfield. Never let the worker self-certify.
7. **Pilot Managed Agents with a self-hosted sandbox, behind an executor interface.** It gives hosted orchestration, session budgets, Outcomes and scheduled deployments while tool execution stays on the home server. But it is beta, sends tool I/O to Anthropic, cannot take file or repo resources on self-hosted environments, and is outside ZDR. Do not make it load-bearing in phase 1.
8. **Cache and cost hygiene.** Set the 1-hour cache TTL for loops that wait on CI. Keep prompts and tool sets stable per run, since caches are per model and a cascade forfeits reuse. Move playbooks into skills, keep CLAUDE.md short, and prefer `gh` and CLI tools over MCP servers.
9. **Observability from day one.** Run an OTel collector on the server. Export metrics and logs (traces optional), tag with `OTEL_RESOURCE_ATTRIBUTES` (goal id, round, role, repo), and pass `TRACEPARENT` so one goal round is one trace. Alert on `api_refusal`, `api_retries_exhausted` and cost-per-goal. The `system/init` check on `mcp_server_errors` and `plugin_errors` should fail the run early.
10. **Handle refusals and fallbacks.** Fable 5.1, Opus 5.5 and Sonnet 5.5 can return `refusal` stop reasons from safety classifiers (for example in security-adjacent dependency work). Configure fallback models and treat a refusal as a distinct failure class that escalates instead of retrying.

## Recommended reference stack and role-to-model mapping

**Stack:** home-server orchestrator (Python or TypeScript) holds the goal store (git plus SQLite/Postgres). It launches Agent SDK queries (or `claude -p --bare`) inside rootless containers, one git worktree per attempt, on the self-hosted CI runners. Hooks (Stop, PreToolUse, PreCompact) enforce gates. Skills carry per-goal-type playbooks. Auth is an API key via `apiKeyHelper`. An OTel collector feeds Prometheus/Grafana or equivalent. Managed Agents with a self-hosted worker is a later, optional executor for scheduled and outcome-style goals.

| Role | Model (pin full ID) | Effort | Rationale |
|---|---|---|---|
| Planner / goal definer / DoD proposer (type 4), architectural escalation triage | `claude-opus-5-5` (`claude-fable-5-1` for hardest greenfield planning) | high | Plans are few and expensive to get wrong; Opus 5.5 is cheaper than Fable. |
| Worker (edits, builds, dependency adaptation) | `claude-sonnet-5-5` default, escalate to Opus 5.5 on repeated failure | medium to high | Anthropic positions Sonnet 5.5 for everyday coding and agent work at about half Opus's price. |
| Verifier / judge (rubric grading, review) | `claude-opus-5-5`, read-only tools, separate context from the worker | high | A judge at least as strong as the worker reduces self-grading bias. This is a design choice; I found no Anthropic benchmark for it. |
| Cheap subagents (log triage, changelog summarizing, file search, CI-log parsing) | `claude-haiku-5-5` | low to medium | $0.10 / $0.50 per MTok. Keep prompts under 100K tokens to stay on the lower price. |
| `/goal` evaluator | small fast model (default); override via `ANTHROPIC_DEFAULT_HAIKU_MODEL` | n/a | Billed on the fast model. That env var also moves other background work, so set it deliberately. |
| Fallback chain | Opus 5.5 to Sonnet 5.5 to Haiku 5.5 via `--fallback-model` | n/a | Covers overload and server errors only. |

Test the "one strong model at lower effort" option before committing to a multi-model cascade (Anthropic's own guidance, cached 2026-10-06).

## Open questions

1. Is a personal, always-on, autonomous factory on a Max subscription permitted? No Anthropic page states it. Ask Anthropic or default to API billing.
2. Does `claude-code-action@v1` work on self-hosted runners (requirements, Bun/Node, security)? Not documented in what I read. Test it, or skip the action.
3. Do the new Max/Team API credits cover Managed Agents session-hour fees, and how are they metered? UNVERIFIED.
4. What are the dynamic-workflow limits and cost behavior (`/docs/en/workflows`)? Not read.
5. How well does the Stop-hook plus CI-gate loop converge compared with the `/goal` evaluator on real upgrade tasks? Needs a pilot with logged iterations.
6. Managed Agents beta stability: how often do event schemas or behaviors change? Watch the release notes.
7. Rate-limit tier needed for N parallel workers (TPM/RPM per tier) was not researched. Check the Console limits page before sizing parallel option-testing.

## Sources

- Headless / `claude -p`: https://code.claude.com/docs/en/headless (fetched 2026-10-07)
- Agent SDK overview: https://code.claude.com/docs/en/agent-sdk/overview (2026-10-07)
- Agent SDK subagents, caps: https://code.claude.com/docs/en/agent-sdk/subagents (2026-10-07)
- Agent SDK cost tracking: https://code.claude.com/docs/en/agent-sdk/cost-tracking (2026-10-07)
- Permission modes: https://code.claude.com/docs/en/permission-modes (2026-10-07; partially read)
- Hooks: https://code.claude.com/docs/en/hooks (2026-10-07; first 100K characters read)
- /goal: https://code.claude.com/docs/en/goal (2026-10-07)
- GitHub Actions: https://code.claude.com/docs/en/github-actions (2026-10-07)
- claude-code-action FAQ: https://github.com/anthropics/claude-code-action/blob/main/docs/faq.md (2026-10-07)
- Costs: https://code.claude.com/docs/en/costs (2026-10-07)
- Model config: https://code.claude.com/docs/en/model-config (2026-10-07; last 16K characters not read)
- Authentication: https://code.claude.com/docs/en/authentication (2026-10-07)
- Legal and compliance: https://code.claude.com/docs/en/legal-and-compliance (2026-10-07)
- Monitoring (OTel): https://code.claude.com/docs/en/monitoring-usage (2026-10-07; first 100K characters read)
- Managed Agents overview: https://platform.claude.com/docs/en/managed-agents/overview (2026-10-07)
- Managed Agents self-hosted sandboxes: https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes (2026-10-07)
- Managed Agents outcomes: https://platform.claude.com/docs/en/managed-agents/define-outcomes (2026-10-07)
- Managed Agents budgets: https://platform.claude.com/docs/en/managed-agents/budgets (2026-10-07)
- Managed Agents reference (rate limits): https://platform.claude.com/docs/en/managed-agents/reference (2026-10-07)
- Agent SDK with your plan (policy, dated 2026-10-07 and 2026-06-15 updates): https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- API credits for Max and Team (amounts, from search result only): https://platform.claude.com/docs/en/about-claude/api-credits-for-subscribers
- Pricing page (not fetched; for re-verification): https://platform.claude.com/docs/en/about-claude/pricing
- Claude API skill reference, cached 2026-10-06: model IDs, pricing, API behavior (thinking, `tool_choice`, preserved thinking, compaction, context editing). It has no public URL; re-verify against the pricing and models pages.
