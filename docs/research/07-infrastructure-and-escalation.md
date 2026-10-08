# 07 - Infrastructure and Escalation (home server, CI, sandboxes, queue, triggers, HITL)

Researched 2026-10-07. Research tooling note: WebFetch could not resolve hosts in this environment, so claims rest on WebSearch result summaries. Many sources are third-party blogs; anything not backed by an official doc is flagged UNVERIFIED. Verify against official docs before building.

## Summary

- Use one forge with a GitHub-compatible workflow dialect and ephemeral runners. If the repos live on GitHub, keep GitHub Actions with JIT ephemeral self-hosted runners. If you want a local forge, Forgejo Actions is the closest drop-in. Woodpecker is a separate system with its own YAML and more moving parts.
- Agent workloads are not CI jobs. Run the agent in its own sandbox (rootless Podman container by default, microVM or gVisor for riskier goals) orchestrated by the control plane, not inside a CI runner. CI stays the judge of green/red.
- Sandbox ladder at home-server scale: git worktree (filesystem isolation only) < rootless Podman < gVisor < Firecracker microVM. Parallel option testing needs worktree + container per option, with per-sandbox cgroup limits.
- Control plane: a single daemon with a SQLite (WAL) goal/event/question store, systemd timers for schedules, and a small authenticated HTTP endpoint for webhooks. Temporal is viable on one node but is a heavy dependency; defer it.
- Triggers: poll (releases/feeds/GitHub API) as the source of truth, webhooks as an optimization. GitHub does not auto-retry failed deliveries, and a home server is often offline, so polling reconciliation is mandatory.
- Escalation: a durable question table is the source of truth; notification channels (ntfy first) are mere doorbells with deep links. Every question carries context, options, a recommendation, and a default-after-timeout. Approvals for code live on the PR; non-code forks live in the question queue.

## Findings

### A1. Forge and CI for self-hosted runners

| Option | Fit for dark factory | Notes |
|---|---|---|
| GitHub Actions + self-hosted runners | Best if repos are on GitHub | Actions Runner Controller (ARC) runners use `--ephemeral` and JIT config by default; the controller retries failed runner pods up to 5 times and GitHub unassigns a job after 24h if no runner accepts it ([GitHub docs, ARC](https://docs.github.com/en/actions/concepts/runners/actions-runner-controller), current). ARC needs Kubernetes, which is likely overkill for one box. A non-ARC path exists: `POST /repos/{owner}/{repo}/actions/runners/generate-jitconfig` plus `run.sh --jitconfig` so the host never holds a long-lived registration token ([third-party guide](https://privatedevops.com/articles/ephemeral-ci-runners-on-your-own-hardware), UNVERIFIED against GitHub REST reference). Ephemeral hosts lose logs, so forward them externally (same source). |
| Forgejo Actions (or Gitea act_runner) | Best if you want a fully local forge | Runs many GitHub-designed workflows; default runner image is minimal Debian + node vs GitHub's large Ubuntu image; some keys ignored (e.g. `permissions`, `continue-on-error`); OIDC uses a different key ([Forgejo docs v17](https://forgejo.org/docs/v17.0/user/actions/github-actions/), via search summary). Other runners can serve Forgejo if they speak the same protocol ([Forgejo admin docs](https://forgejo.org/docs/v14.0/admin/actions/)). Ephemeral/one-shot runner support: could not verify, check the forgejo-runner docs. |
| Woodpecker | Only if forge independence matters | Separate server, agents, DB and UI; own YAML, not GitHub-compatible; works with many forges; stronger matrix/caching claims are from vendor content ([RamNode](https://ramnode.com/guides/series/gitea-devops/ci-cd-pipelines), [sumguy](https://www.sumguy.com/gitea-actions-vs-woodpecker); UNVERIFIED, undated). |
| GitLab runner | Not researched in depth | UNVERIFIED. Heavy forge if GitLab itself must be hosted. |

Tradeoffs that matter for agents: (1) workflow-dialect compatibility determines how much existing upstream CI you can reuse unchanged; (2) ephemeral runners prevent cross-run contamination, which matters when agents modify workflow files; (3) agent-authored PRs can edit CI config, so branch protection and required checks must be defined outside the PR's reach (integration point only; security is out of scope).

Claude invocation: headless `claude -p` with `--output-format json` returns a `session_id` that can be passed to `--resume`; `--max-turns` guards runaway loops; `stream-json` helps for long runs ([SFEIR guide](https://institute.sfeir.com/en/claude-code/claude-code-headless-mode-and-ci-cd/), third-party, undated). Session state lives under `~/.claude/projects/`, so ephemeral containers must mount or restore it (inference, UNVERIFIED). Prefer the Agent SDK for the factory's own driver; not researched here (see other researchers).

### A2. Sandboxing concurrent agent runs

- Isolation ladder ([amux](https://amux.io/guides/ai-agent-sandboxing/), [SoftwareSeni](https://www.softwareseni.com/firecracker-gvisor-containers-and-webassembly-comparing-isolation-technologies-for-ai-agents/), 2026 vendor-adjacent blogs; figures indicative only):
  - Rootless Podman/Docker: shared kernel, lowest overhead (<1s start, <2% CPU claimed). Rootless limits blast radius of escape but adds no kernel boundary.
  - gVisor: user-space kernel, <1s start, 5-15% syscall overhead claimed; middle ground, little infra.
  - Firecracker: own guest kernel under KVM, ~125-150ms boot claimed, strongest, but you build the fleet/scheduler. Needs KVM on the host. Daytona/microsandbox-style tools wrap this (Daytona startup 5-15s claimed).
  - Git worktrees: isolate the working tree/branch only; no process, network or port isolation. Use them for source isolation inside a container, not as the sandbox.
  - Devcontainers: a convenient per-repo environment definition (toolchain, services); can be the image source for the Podman sandbox. Not researched in detail.
- One source: containers are probably fine for internal logic that passed CI; use microVMs for untrusted/prompt-injectable code; always add egress, filesystem and resource limits, timeouts and a watchdog ([morphllm](https://www.morphllm.com/ai-agent-sandbox); UNVERIFIED).
- Concurrency at home scale: bound by RAM and CPU, not by tool. N parallel options means N (worktree + container + port range + DB/service instances). Docker-in-sandbox for integration tests is the hard part; rootless Podman with per-sandbox socket, or a microVM, handles it.

### A3. Queue and scheduler

- SQLite in WAL mode handles one-writer-claims-row queues well on one machine; caveat: at-least-once semantics, so handlers must be idempotent; Litestream replication is asynchronous so recent writes can be lost on disk failure ([vardiya/DEV](https://dev.to/zulwatha/i-got-tired-of-running-redis-for-one-background-job-so-i-built-a-sqlite-job-queue-4flk), [Obelisk 2026](https://obeli.sk/blog/sqlite-is-all-you-need-for-durable-workflows/); blogs).
- Temporal self-hosted: single-node Docker Compose with Postgres is described as fine for low volume; needs several containers and ~512MB per role ([automationatlas 2026](https://automationatlas.io/guides/tutorial-temporal-self-hosted-deploy-2026/), blog, UNVERIFIED). Gives replay, long timers, UI. Costly for a one-person home server.
- Redis: no advantage at this scale. systemd timers/cron: right for fixed schedules and pollers; they only enqueue rows.
- Recommendation: goals are long-lived state machines; model them as rows with explicit states and lease/heartbeat columns rather than adopting a workflow engine.

### A4. Trigger plumbing

- GitHub webhooks: verify `X-Hub-Signature-256` (HMAC-SHA256 over the raw body, constant-time compare); no header is sent if no secret is set ([GitHub docs, validating deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries); search-summary).
- Failure behavior: no automatic redelivery; a 10s response limit; redelivery can be scripted via REST by listing deliveries with non-OK status ([GitHub docs, failed deliveries](https://docs.github.com/webhooks/using-webhooks/handling-failed-webhook-deliveries)). A reconciliation job should call this on a schedule.
- Tunnels: Cloudflare Tunnel can expose only the webhook endpoint ([ri-forward-webhook example](https://github.com/RiskIdent/ri-forward-webhook)). Zero Trust/Access policies and tunnel timeout behavior: UNVERIFIED. Alternative: no inbound exposure at all, pure polling (simplest, fits "upstream update" goals where minutes of latency are fine).
- Release polling: GitHub `releases`/`tags` API with ETag conditional requests, Atom feeds (`/releases.atom`), registry APIs (npm, PyPI, crates, container registries). ETag behavior is standard HTTP; rate-limit specifics UNVERIFIED here.
- Normalize every trigger into an `event` row with a dedupe key (source + id); goals subscribe to event patterns.

### A5. Artifacts, logs, observability

- Store per-run: transcript (stream-json), diff/patch, test output, screenshots, as files under a content-addressed directory on local disk, indexed in SQLite. Back up the DB and artifact dir (restic) since Litestream is async.
- Dashboard: a single read-only web page over SQLite (goals, state, last event, open questions, spend, CI status). Grafana/Prometheus is optional; a status page plus an event timeline gets 90% of value. Ephemeral runner logs must be shipped out before teardown.
- Claude Code hooks exist (e.g. Notification with `permission_prompt|idle_prompt` matchers) but have had delay and reliability bugs, notably in the VS Code extension ([hooks guide](https://code.claude.com/docs/en/hooks-guide), [issue 43530](https://claudeissues.com/issue/43530-notification-hook-fires-8-seconds-after-permission-prompt-appears), closed 2026-04-08). Headless runs should not rely on interactive permission prompts at all; the factory should use explicit allowlists and its own question tool.

### B1. Escalation transports

- ntfy: supports priority 1-5, and `view`, `http` and `copy` action buttons; an `http` button calls your endpoint from the phone, so the endpoint must be reachable from the device ([ntfy publish docs](https://docs.ntfy.sh/publish), via search summaries; exact `http` action syntax UNVERIFIED). Self-hostable.
- Telegram: bot API from a script is simple; Claude Code Channels (announced 2026-03-20, research preview, needs v2.1.80+ and Bun) forward Telegram/Discord messages into a running local session ([MindStudio guide](https://www.mindstudio.ai/blog/how-to-set-up-claude-code-channels-telegram); third-party). It is session-bound and local; unsuitable as the factory's durable queue, but fine as a chat UI. Whether permission prompts are forwarded: UNVERIFIED.
- Pushover, Slack: not researched; both are standard webhook-in, push-out doorbells (UNVERIFIED specifics).
- GitHub issues/PR comments: durable, threaded, mobile app notifications, and a natural place for code-adjacent questions; not suitable for generic goals not tied to a repo.
- User's own multi-harness messaging tool: not inspected by me (no access in this research). Treat as a transport adapter behind the same interface (`send(question)`, `on_reply(question_id, answer)`); decide after inspecting its API.

### B2. Escalation design

Use a questions table as source of truth; channels are replaceable. Per question:

```
id, goal_id, kind (design|fork|ambiguity|acceptance|blocked)
context (what was being done, evidence links)
options[]  (label, consequence, effort)
recommendation + confidence
default_action + default_after (timeout)  -- executed if unanswered
blocks[] (which work items are parked) ; unblocked work continues
status: open|answered|defaulted|withdrawn ; answered_via, answered_at
```

Rules:
- Ask only after exhausting unblocked work; park the branch, not the goal.
- Default-after-timeout should be the reversible, conservative option (e.g. keep current behavior, open draft PR instead of merging). Irreversible defaults are forbidden.
- Batch low-urgency questions into a digest; use high-priority push only for blockers that stall everything.
- Answers arrive as free text or option id; an LLM step parses and records an interpretation back for confirmation if confidence is low.
- Withdraw questions automatically if upstream state makes them moot, and notify.

Approvals: code changes are approved on the PR (review/approve, or a `/approve`-style comment, or label). Design forks are answered in chat with option buttons. Never have two sources of truth for the same decision; the PR comment thread should link to the question id, and the factory ingests PR review events as answers.

### B3. Acceptance review UX for goal type 4 (greenfield)

Present a single review page per goal: criteria checklist with evidence per item (test results, screenshots, run command), a live preview URL for the built app (ephemeral, home-network or tunnel), diff summary vs. spec, list of decisions the agent made and open assumptions, and three actions: accept, accept with follow-ups (creates goals), reject with notes (creates revision goal). Phone push links to this page. This UX is my design, not drawn from a source.

## Implications for the dark factory

1. Pick GitHub-compatible CI. If code is on GitHub, use plain GitHub Actions with JIT ephemeral runners from a small supervisor script (no Kubernetes). If self-hosting the forge, Forgejo. Do not adopt Woodpecker unless forge independence becomes a goal.
2. Separate "agent sandbox" from "CI runner". Controller launches agent in rootless Podman, one container per (goal, attempt, option), with worktree bind-mounted, cgroup limits and egress rules provided by the existing security layer. Reserve Firecracker/gVisor for a per-repo flag `isolation: microvm` (greenfield builds that run arbitrary installers; options needing Docker-in-Docker).
3. Build the control plane as one daemon + SQLite. No Temporal, no Redis. Goals are state machines with leases; handlers are idempotent; systemd timers only enqueue.
4. Poll first, webhooks second. Pollers run on timers and write events; a tunneled webhook endpoint is added later, with a reconciliation sweep that redelivers failed GitHub deliveries.
5. Escalation is its own subsystem with a transport interface. Start with ntfy (self-hosted) + a web question page + PR comments; add the user's messaging tool as an adapter after reading its API.
6. Per-repo autonomy config lives in the goal's repo file (e.g. `.factory/policy.yaml`): auto-merge-on-green, escalate-on classes, default-after-timeout, isolation level.
7. Make the dashboard the same web app that serves question pages, so a notification deep-link lands on both context and action.

### Minimal architecture

```
 triggers                         control plane (one host, one daemon)                 workers
 --------                         -------------------------------------                -------
 systemd timers --poll-->  +---------------------------------------------+
 (releases, feeds, cron)   |  event ingest -> dedupe -> goal matcher     |
 GitHub webhook (tunnel) ->|                                             |
 PR/issue events ---------->|  SQLite (WAL): goals, events, runs,        |-- lease --> sandbox launcher
                           |  work_items, questions, artifacts index     |           (rootless Podman /
                           |                                             |            microVM) + worktree
                           |  scheduler loop (leases, heartbeats, retry) |              |
                           |  escalation service (question lifecycle,    |              +-> claude -p / Agent SDK
                           |   timeouts, defaults, withdraw)             |
                           |  web UI: dashboard + question/accept pages  |<-- results, logs, artifacts
                           +--------+---------------------+--------------+
                                    |                     |
                          notifier adapters         forge + CI
                          ntfy | Telegram |        (GitHub Actions JIT runners
                          user's msg tool |         or Forgejo) -> green -> auto-merge
                          PR comments               artifacts dir + restic backup
```

### Build order

1. SQLite schema + event ingest + one poller (release polling for one dependency) + goal state machine (no agent yet).
2. Sandbox launcher: rootless Podman + worktree + `claude -p` with JSON output; capture transcript/artifacts; resume by `session_id`.
3. CI integration: PR creation, JIT ephemeral runners, read green/red, auto-merge per repo policy.
4. Dashboard (read-only), then question table + escalation service with ntfy and web answer page; timeout/default executor.
5. PR-review-as-answer ingestion; webhook endpoint via tunnel + redelivery sweep.
6. Parallel option testing (N sandboxes, resource accounting, comparison report as an acceptance/choose question).
7. Greenfield acceptance page; optional microVM isolation tier; messaging-tool adapter; Temporal only if state machines get painful.

## Open questions

- Are repos on GitHub, local Forgejo, or both? This decides the CI path (not stated in design context).
- Does Forgejo runner support true ephemeral/one-job runners? (Not verified.)
- Host has KVM and enough RAM for N concurrent sandboxes? How many parallel options are wanted?
- What does the user's messaging tool expose (inbound replies, buttons, threading, auth)?
- Is the phone reachable from home-server endpoints (VPN such as Tailscale vs public tunnel) for ntfy `http` buttons and review pages?
- Agent SDK vs `claude -p` for the driver, and how session state is persisted across sandboxes (see other reports).
- Cloudflare Tunnel/Access behavior with GitHub webhook IPs and the 10s limit: unverified.

## Sources

- https://docs.github.com/en/actions/concepts/runners/actions-runner-controller (ARC, ephemeral/JIT; current)
- https://privatedevops.com/articles/ephemeral-ci-runners-on-your-own-hardware (JIT config endpoint, third-party, undated)
- https://forgejo.org/docs/v17.0/user/actions/github-actions/ and https://forgejo.org/docs/v14.0/admin/actions/ (Forgejo Actions; read via search summary)
- https://www.devroom.io/2026/03/15/using-github-actions-in-self-hosted-forgejo/ (2026-03-15)
- https://ramnode.com/guides/series/gitea-devops/ci-cd-pipelines and https://www.sumguy.com/gitea-actions-vs-woodpecker (undated)
- https://amux.io/guides/ai-agent-sandboxing/ , https://www.softwareseni.com/firecracker-gvisor-containers-and-webassembly-comparing-isolation-technologies-for-ai-agents/ , https://www.morphllm.com/ai-agent-sandbox (2026 sandbox comparisons, vendor-adjacent)
- https://dev.to/zulwatha/i-got-tired-of-running-redis-for-one-background-job-so-i-built-a-sqlite-job-queue-4flk , https://obeli.sk/blog/sqlite-is-all-you-need-for-durable-workflows/ , https://automationatlas.io/guides/tutorial-temporal-self-hosted-deploy-2026/
- https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries , https://docs.github.com/webhooks/using-webhooks/handling-failed-webhook-deliveries , https://github.com/RiskIdent/ri-forward-webhook
- https://docs.ntfy.sh/publish (via search summaries only)
- https://code.claude.com/docs/en/hooks-guide , https://claudeissues.com/issue/43530-notification-hook-fires-8-seconds-after-permission-prompt-appears , https://www.mindstudio.ai/blog/how-to-set-up-claude-code-channels-telegram , https://institute.sfeir.com/en/claude-code/claude-code-headless-mode-and-ci-cd/
