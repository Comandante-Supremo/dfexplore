# W2-01: Harness CLI headless contracts (Claude Code, Codex, Kimi, OpenCode, ACP/MCP/A2A/AGENTS.md)

Date of research: 2026-10-07. Tags: [V] verified from a primary source I read (URL or repo path + commit/tag given); [U] unverified. Repo evidence comes from bare partial clones of the upstream repos in the scratchpad (`git log`/`git show` on upstream history) and from registry JSON (npm/PyPI), because several doc hosts were unreachable (see Fetch failures).

## 1. Summary

- All four harnesses have a non-interactive mode with machine-readable output, but only two have a documented, typed event schema: Claude Code (SDK message types, documented) and Codex (`exec_events.rs` plus a TypeScript SDK that spawns `codex exec --experimental-json`). OpenCode `run --format json` emits an ad-hoc envelope around internal "Part" objects; Kimi emits OpenAI-style `{role, content, tool_calls}` JSONL.
- Wave one's "Kimi CLI" target is dead. `MoonshotAI/kimi-cli` (Python) was archived; its final release is 1.52.0 on 2026-09-22. The successor is `MoonshotAI/kimi-code` (Node.js, npm `@moonshot-ai/kimi-code`, 2.1.1 on 2026-09-24) with a different CLI surface (`-p` takes the prompt, no `--print`/`--wire`/`--input-format` in `apps/kimi-code/src`, headless exit semantics unverified). The conformance suite needs two Kimi adapters or a "kimi-cli legacy" tombstone.
- Breaking-change base rates differ sharply. Claude Code: 3 explicitly breaking headless-output changes, all pre-2.0 (2025-04 to 2025-06); since then output changes are additive or bug-fix. Codex: 4 schema renames in the first 8 days after the JSONL schema appeared (2025-09-25 to 2025-10-02), then 14 additive commits to the event file in 12 months, but CLI flags are removed on a deprecate-then-remove path (`--full-auto`: deprecated 2026-04-28, removed 2026-07-30). Kimi: one output-semantics change (print mode exit codes, 1.27.0) plus a full rewrite (2.0). OpenCode: JSON envelope stable since 2025-09-24; payload (`part`) tracks internal types and the OpenAPI surface grows fast (85 -> 113 -> 161 paths, Mar/Jun/Oct 2026).
- Release cadence is high everywhere: Claude Code 78 stable npm releases in the last 90 days, Codex 39, OpenCode 41. A per-release live canary is needed, not a quarterly one.
- ACP is now a real cross-harness option: OpenCode (`opencode acp`) and Kimi (`kimi acp`) are native; Claude Code and Codex go through adapters (`zed-industries/claude-agent-acp`, `agentclientprotocol/codex-acp`). But ACP itself is mid-migration (v1 schema releases ~every 1-2 weeks; v2 draft alpha.7 on 2026-09-30), so it moves the churn rather than removing it.
- MCP's 2026-07-28 revision is itself a large breaking spec change (stateless, no `initialize`, no sessions). Relevant if the messaging tool exposes an MCP server.

## 2. Verified findings

### 2.1 Claude Code

Invocation and output formats [V] https://code.claude.com/docs/en/headless (fetched 2026-10-07):
- `claude -p "<prompt>"` (alias `--print`); stdin is read and piped input is capped at 10MB (non-zero exit with error if exceeded). `--output-format text|json|stream-json`; `--input-format stream-json` exists (docs mention it for SDK hosts; changelog 2.1.251, 2.1.257, 2.1.274 [V]).
- stream-json is newline-delimited; the docs' streaming example uses `--output-format stream-json --verbose --include-partial-messages`; "The last line of the stream is a `result` message". Whether `--verbose` is strictly required is not stated on the page [U] (wave one asserted it).
- `--json-schema` with `--output-format json` returns the object in `structured_output`. Invalid schema: exits with `Error: --json-schema is not a valid JSON Schema`. Before v2.1.205 an invalid schema was silently ignored (a behaviour change that turned silent success into a hard error).
- Exit codes: 0 on success, non-zero on failure; invalid flags go to stderr before the run; in-run failures such as missing auth are "printed as the result on stdout"; SIGTERM yields 143 with no result recorded; SIGINT or SDK `interrupt()` ends the turn cleanly. Per-error-subtype exit codes are not documented [U].
- `--bare`: skips hooks, skills, plugins, MCP, auto memory, CLAUDE.md; never reads OAuth/keychain (needs `ANTHROPIC_API_KEY` or `apiKeyHelper`); "will become the default for `-p` in a future release" (a pending behaviour change the suite should pin both ways). Before v2.1.286 its limits "held only partly".
- Without `--bare`, `-p` runs project hooks and `.mcp.json` servers with no trust dialog.
- Permissions: `--allowedTools`, `--permission-mode` (`auto`, `dontAsk`, `acceptEdits`, ...), `--permission-prompts none` (v2.1.259+; earlier versions "reject it with an unknown-option error"). From 2.1.285 `-p` on third-party providers or telemetry-off starts in auto mode when no mode is configured (changelog [V]) - a silent default change.

Event schema [V] https://code.claude.com/docs/en/agent-sdk/typescript (type definitions quoted on the page; I read chars 0-200000 of 277762):
- `SDKMessage` is a union of 36 member types (assistant, user, result, system, stream_event partials, hook_*, task_*, rate_limit, permission_denied, api_retry, ...). CLI stream-json emits these same objects one per line.
- `system/init`: `session_id, uuid, claude_code_version, cwd, tools, mcp_servers[{name,status,source?}], model, permissionMode, slash_commands, output_style, skills, plugins, plugin_errors?, apiKeySource, capabilities?` (the `capabilities` string array is an explicit feature-detection mechanism, "ignore values you don't recognize", v2.1.205+; named values: `interrupt_receipt_v1`, `interrupt_cancel_queued_v1`, `sdk_mcp_manifests`, `sdk_mcp_tools_list_changed`). `mcp_server_errors` is documented on the headless page (v2.1.219+).
- `result`: discriminated on `subtype`: `success` | `error_max_turns` | `error_during_execution` | `error_max_budget_usd` | `error_max_structured_output_retries`. Common fields: `session_id, uuid, duration_ms, duration_api_ms, is_error, num_turns, stop_reason, total_cost_usd, usage, modelUsage, permission_denials`; success adds `result, structured_output?`; error arms add `errors[]`. Optional `terminal_reason` enumerates ~19 values (completed, max_turns, budget_exhausted, ...).
- `system/api_retry` fields (`attempt, max_retries, retry_delay_ms, error_status, no_response?, error, uuid, session_id`) and the `error` enum are documented on the headless page [V]. Subagent messages carry `parent_tool_use_id`; forked skills started via `/skill` carry a `forked-command-` prefix value.
- Stability guarantee: no explicit semver/stability promise for the stream was found on either page [U-negative]. The de facto policy is "additive plus version-gated" (every new field is annotated with "requires v2.1.x"); consumers are told to feature-detect via `capabilities`.

Sessions [V] https://code.claude.com/docs/en/sessions:
- Transcripts: `~/.claude/projects/<project>/<session-id>.jsonl`; "The entry format is internal to Claude Code and changes between versions, so scripts that parse these files directly can break on any release." Relocate with `CLAUDE_CONFIG_DIR`; `CLAUDE_CODE_PROJECT_DIR_NAME` (v2.1.234+) needs `CLAUDE_CONFIG_DIR`; retention 30 days (`cleanupPeriodDays`); `--no-session-persistence` for `-p`.
- `-p`/SDK sessions are hidden from the picker and `claude --continue` (but `claude -p --continue` includes them). `--resume <id>` finds sessions in any project on the machine since v2.1.223 (exactly one match required; a copied duplicate gives not-found). Resume does not restore `--mcp-config`, `--settings`, `--plugin-dir`, `--add-dir`, `--fallback-model`. Resuming a session that is still running in the background: with output-format flags, `--max-turns` etc. it "sends nothing and exits with status 1" (v2.1.285+). Failure to load a picked session exits 1. Not-found message: `No conversation found with session ID: <id>`.
- Config/hook locations [V] https://code.claude.com/docs/en/settings-reference and /hooks: `~/.claude/settings.json`, `.claude/settings.json`, `.claude/settings.local.json`, managed settings, global `~/.claude.json`; hooks live in the `hooks` key (and plugin `hooks/hooks.json`); 33 hook events; hook stdin fields `session_id, prompt_id, transcript_path, cwd, permission_mode, hook_event_name`; exit 2 blocks, other non-zero is non-blocking; JSON output `continue/stopReason/systemMessage/decision/hookSpecificOutput`. Managed-settings OS paths and credential storage location were not confirmed on the page I read [U].

Release channel and cadence [V] https://registry.npmjs.org/@anthropic-ai/claude-code (time map) and https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md: npm `@anthropic-ai/claude-code`, latest 2.1.293 (2026-10-07); 539 stable semver releases since 2025-02-24; 78 in the 90 days before 2026-10-07 (about 6 per week). Agent SDK package `@anthropic-ai/claude-agent-sdk` 0.3.293 (318 releases since 2025-09-27, 78 in 90 days), version-locked to the CLI. Announcement channel: the in-repo CHANGELOG.md only; entries carry no dates, and "Breaking change:" is used sparingly (see 2.5).

### 2.2 OpenAI Codex CLI

Source: bare clone of https://github.com/openai/codex at main (2026-10-07), files `codex-rs/exec/src/cli.rs`, `codex-rs/exec/src/exec_events.rs`, `codex-rs/utils/cli/src/shared_options.rs`, `sdk/typescript/src/exec.ts`. [V] for everything below unless marked.
- Invocation: `codex exec [OPTIONS] [PROMPT]`; prompt from stdin if omitted or `-` (stdin is appended as a `<stdin>` block if both). Subcommands `resume [SESSION_ID|--last] [PROMPT]`, `fork`, `review`. `--json` (alias `--experimental-json`, which the TS SDK still passes) prints JSONL to stdout; `--output-schema <FILE>` constrains the final message; `-o/--output-last-message <FILE>`; `--ephemeral` (no session files); `--skip-git-repo-check`; `--ignore-user-config`, `--ignore-rules`; `--strict-config` (errors on unknown config fields); `--color`; `-C/--cd`, `--add-dir`, `-m`, `-p/--profile` (loads `$CODEX_HOME/<name>.config.toml`), `--oss`.
- Sandbox/approval: `-s/--sandbox <mode>`; `--approve-for-me` (alias `--not-so-yolo`: auto-review with workspace-write); `--dangerously-bypass-approvals-and-sandbox` (alias `--yolo`). `--ask-for-approval/-a` exists in the TUI CLI (`codex-rs/tui/src/cli.rs`) but is not declared in the exec option set I read; the TS SDK passes `--config approval_policy=...` instead [exec option set read directly; runtime behaviour not run]. `--full-auto` no longer exists for `exec` (see 2.5).
- Event schema (`ThreadEvent`, serde `tag = "type"`): `thread.started{thread_id}` (first event; "Can be used to resume the thread later"), `turn.started`, `turn.completed{usage{input_tokens,cached_input_tokens,cache_write_input_tokens,output_tokens,reasoning_output_tokens}}`, `turn.failed{error{message}}`, `item.started|updated|completed{item}`, `error{message}`. Items (`type` snake_case + `id`): `agent_message{text}`, `reasoning{text}`, `command_execution{command,aggregated_output,exit_code?,status in_progress|completed|failed|declined}`, `file_change{changes[{path,kind add|delete|update}],status}`, `mcp_tool_call{server,tool,arguments,result?,error?,status}`, `collab_tool_call{...}`, `web_search{id,query,action,results?}`, `todo_list{items[{text,completed}]}`, `error{message}`. The file is the TS-RS source for exported types, so it is a machine-readable schema.
- Exit codes: failures call `std::process::exit(1)` in `exec/src/lib.rs` (many sites); commit cc1b21e47 (#4309, 2025-09-26): "ensures we return 1 on failures" with a test `exits_non_zero_when_server_reports_error`. No documented retryable code.
- Config/auth: `$CODEX_HOME/config.toml` (from `--ignore-user-config` help text); `docs/authentication.md` and `docs/exec.md` in the repo are one-line pointers to developers.openai.com (unreachable, see failures), so the auth file path and `CODEX_API_KEY` env var are [U].
- Architecture shift: commit "Finish moving codex exec to app-server (#15424)" 2026-03-24: exec now runs on top of the app-server JSON-RPC layer, which is a richer interface and a likely future replacement for `--json`.
- Release channel: npm `@openai/codex` latest 0.161.0 (2026-10-07), 200 stable 0.x releases since 2025-04-16, 39 in the last 90 days; still pre-1.0 [V registry]. Announcements: GitHub releases/commit messages (the repo has no CHANGELOG file at the paths I listed [U]).
- Stability statement: none found; the `--experimental-json` alias suggests the JSON stream was once labelled experimental, and the `--json` flag help carries no stability promise.

### 2.3 Kimi (Moonshot): two generations

Legacy `kimi-cli` (Python) [V] bare clone of https://github.com/MoonshotAI/kimi-cli, `CHANGELOG.md`, `docs/en/customization/print-mode.md`, `wire-mode.md`, `data-locations.md`:
- `kimi --print -p "..."` (or stdin); `--output-format text|stream-json`, `--input-format text|stream-json`, `--final-message-only`, `--quiet`, `--afk`/`--yolo`, `--continue/-C`, `--session|--resume [ID]` (creates new if ID not found), `--wire` (experimental JSON-RPC 2.0 over stdio, protocol version `1.10`, per-message "Added/Changed in Wire 1.x" annotations, `initialize` handshake negotiating `protocol_version`), `kimi acp`.
- stream-json lines are OpenAI-style messages: `{"role":"assistant","content":...,"tool_calls":[{type:"function",id,function:{name,arguments}}]}`, `{"role":"tool","tool_call_id","content"}`; thinking is not emitted. Print mode implicitly enables `--afk` (all approvals auto-accepted).
- Exit codes (documented): 0 success, 1 permanent failure (config/auth/quota), 75 retryable (429, 5xx, timeouts). Introduced in 1.27.0 (2026-03-28): "Fix `--print` mode returning exit code 0 on errors".
- Data: `~/.kimi/` (`config.toml`, `mcp.json`, `credentials/<provider>.json`, `sessions/<work-dir-hash>/<session-id>/{context.jsonl,wire.jsonl,state.json}`); `KIMI_SHARE_DIR` relocates it.
- Status: CHANGELOG 1.51.0/1.52.0 "Kimi CLI is archived and no longer maintained, and this is the final release"; 1.52.0 (2026-09-22) makes bare `kimi` download and run the Kimi Code install script from a CDN with no confirmation (a supply-chain-relevant behaviour for an unattended factory). PyPI `kimi-cli` latest 1.52.0.

Successor `kimi-code` (Node.js) [V] bare clone of https://github.com/MoonshotAI/kimi-code, `docs/en/reference/kimi-command.md`, `docs/en/guides/migration.md`, `docs/en/release-notes/changelog.md`, `docs/en/reference/server-api.md`; npm registry `@moonshot-ai/kimi-code`:
- Headless: `kimi -p "<prompt>" [--output-format text|stream-json]`; `-p` now carries the prompt; `--output-format` "can only be used together with `--prompt`"; `--prompt` cannot be combined with `--yolo/--auto/--plan` ("non-interactive mode uses `auto` permission by default; static deny rules remain in effect"). `--session [id]`, `--continue/-c` (note `-c` meaning changed: legacy `-C`), `--agent`, `--add-dir`. Text-mode puts assistant text on stdout and thinking/progress/"resuming session" on stderr. stream-json: "an Assistant message with `tool_calls` is emitted first, followed by the corresponding Tool message" (same message family as legacy). A grep of `apps/kimi-code/src` found no `--print`, `--wire`, or `--input-format` strings [absence check on a partial clone; a runtime test is needed].
- Other surfaces: `kimi acp` (ACP stdio), `kimi web` (REST `/api/v1`, `/api/v2/sessions`, WebSocket `/api/v1/ws`, bearer-token auth, live `/openapi.json` and `/asyncapi.json`). The Server API page states: "The REST and WebSocket APIs ... are experimental: interface stability is not guaranteed".
- Data root moved to `~/.kimi-code/` (server lock/instances path shown in docs); migration via `kimi migrate`; OAuth credentials are not migrated.
- Exit codes for `-p` in kimi-code: `headless-exit.ts` documents a force-exit timer that reads `process.exitCode` lazily and a goal-turn mapping (`goalExitCode`); the legacy 0/1/75 contract is not restated in the new docs I read [U].
- Cadence: npm 83 versions (including prereleases) between 2026-05-21 and 2026-09-24; changelog documents `**Breaking:**` for 0.28.0 (2026-07-20: `kimi server` deprecated, `kimi web` foreground) and a major 2.0.0 on 2026-09-17.

### 2.4 OpenCode

Source: bare clone of https://github.com/sst/opencode (dev branch, 2026-10-07), `packages/opencode/src/cli/cmd/run.ts`, `packages/web/src/content/docs/{server,acp,config}.mdx`; registry https://registry.npmjs.org/opencode-ai.
- Headless: `opencode run [message..]` with `--format default|json`, `--continue`, `--session <id>`, `--fork`, `--model provider/model`, `--agent`, `--file`, `--title`, `--dir`, `--attach <url>` (to a running server; `--password/--username` basic auth), `--port`, `--variant`, `--thinking`, `--command`, `--auto`/`--yolo`/`--dangerously-skip-permissions` (added 2026-04-06, #21266).
- JSON envelope (`emit()` in run.ts): each line is `{"type", "timestamp": Date.now(), "sessionID", ...data}` with `type` in `tool_use | step_start | step_finish | text | reasoning | error` and a `part` object for all but `error` (`error` carries `props.error`). The `part` is the internal message-part model (tool state, text, time); it is not a separately versioned public schema. First added 2025-09-24 (#2471) as `outputJsonEvent`, renamed to `emit` in the 2026-02-02 rewrite (#11814, "make run non-interactive"); event type names were unchanged across that rewrite (verified by diff), `reasoning` added with `--thinking` (#12013, 2026-02-03).
- Behaviours that matter for a harness wrapper: without `--auto`, a permission request is auto-rejected (`reply: "reject"`) and a message printed; the run ends on `session.status` idle; `session.error` sets `process.exitCode = 1`; "Session not found" and invalid-flag combinations `process.exit(1)`; there is no retryable-vs-permanent distinction. `--format json` cannot combine with `--mini`. Commit "Fix run JSON output draining" (2026-05-11) fixed truncation.
- Server: `opencode serve [--port 4096] [--hostname 127.0.0.1] [--cors]`; auth via `OPENCODE_SERVER_PASSWORD` (user `OPENCODE_SERVER_USERNAME`, default `opencode`); OpenAPI 3.1 at `/doc`; the JS SDK is generated from it. `packages/sdk/openapi.json` path count grew 85 (2026-03-01) -> 113 (2026-06-01) -> 161 (2026-10-07); `message.part.updated` event is present at all three points. `opencode acp` = ACP over JSON-RPC stdio.
- Config: `$schema: https://opencode.ai/config.json`; merge order global `~/.config/opencode/opencode.json` -> `OPENCODE_CONFIG` -> project `opencode.json` -> `.opencode/` dirs -> `OPENCODE_CONFIG_CONTENT`; TUI settings split out into `tui.json`; plural subdirectory names with singular accepted "for backwards compatibility". A repo commit (2026-06-10) "accept deprecated reference config key" shows deprecated-key handling in practice.
- Release channel: npm `opencode-ai` latest 1.18.35 (2026-10-06); 944 stable semver versions since 2025-05-31, 41 in the last 90 days; dist-tags include `beta`, `next`, `dev`, and `latest-0`/`latest-1` (old majors kept alive). `run.ts` has had 129 commits since mid-2025. Announcements: GitHub releases only; no changelog file found [U].
- Stability statement: none for `--format json`.

### 2.5 Quantified breaking-change history

Claude Code (CHANGELOG.md keyword mining; counts are keyword-based, so treat as bounds):
- Explicit headless-output breaks: 0.2.117 (2025-05-17) "Breaking change: --print JSON output now returns nested message objects"; 1.0.22 (2025-06-12) "SDK: Renamed `total_cost` to `total_cost_usd`"; 0.2.66 (2025-04-09) stream-json introduced. No `Breaking` entry touching `-p`/stream-json/exit codes since 2.0 (only 2.1.75's Windows managed-settings path removal).
- Behavioural-default changes without a "Breaking" label (all [V] in CHANGELOG): 2.1.205 invalid `--json-schema` now errors; 2.1.285 `-p` default permission mode becomes auto in some setups; 2.1.280 effort defaults; 2.1.90 `--resume` picker hides `-p` sessions; 2.1.267 spurious "Continue from where you left off." turn removed; 2.1.290 `--json-schema` runs no longer exit non-zero with `is_error: true` on a success result.
- Reliability fixes that bit consumers: truncated stream-json at exit (2.1.208, 2.1.214), hangs on stdin/EOF in stream-json (2.1.153), `--print` hanging with teammates (2.1.71), blank CRLF input killing a session (2.1.208), headless session hung by a bad `set_model` payload (2.1.208).
- Volume: 151 of 407 dated versions (37%) mention headless/SDK/stream-json/resume items; 128 of 289 (44%) in the last 12 months.

Codex (git history of `codex-rs/exec/src/exec_events.rs`, 20 commits 2025-09-25 to 2026-09-17): breaking schema changes: `session.created{session_id}` -> `thread.started{thread_id}` (#4478, 2025-09-29), "conversation" -> "thread" (#4482, 2025-09-29), item tag `item_type` -> `type` and `assistant_message` -> `agent_message` (#4610, 2025-10-02), exit code on error added (#4309, 2025-09-26). Afterwards additive: `declined` status (2025-11-21), MCP args/results (2025-10-29), collab tools (2026-01-26), `web_search` actions/results (2026-01-26, 2026-09-17), reasoning tokens in usage (2026-04-24), `cache_write_input_tokens` (2026-07-16), MCP `_meta` (2026-05-16). Flag-level breaks: profile flag renamed to `--profile` (#23883, 2026-05-21) and profile v1 plumbing removed (#23886); `--full-auto` deprecated (#20133, 2026-04-28) then "Stop accepting the hidden, deprecated `--full-auto` flag in `codex exec`... Callers must now select the sandbox mode explicitly" (#36054, 2026-07-30), 93 days later.

Kimi: legacy CLI `--command/-c` -> `--prompt/-p` and `--query` removed (0.77, 2026-01-15); `--acp` deprecated for `kimi acp` (0.74, 2026-01-09); print exit codes (1.27.0, 2026-03-28); `--print` switched from yolo to AFK semantics (1.40.0, 2026-04-28); wire protocol bumped 1.1 -> 1.10 with documented per-field changes (e.g. `task_tool_call_id` -> `parent_tool_call_id` in 1.6) and "old name still accepted" aliasing. Generational break: kimi-cli -> kimi-code 2.0 (2026-09-17), CLI flags and data directory changed.

OpenCode: JSON envelope names stable since 2025-09-24; breaking items are mostly internal/event names (e.g. `vcs.changed` -> `vcs.branch.updated`, 2025-11-26, #4771) and config layout (`tui.json` split; old singular dir names tolerated).

## 3. Cross-harness standards

- ACP [V] clone of https://github.com/agentclientprotocol/agent-client-protocol (tags, `docs/announcements/*`, `docs/get-started/agents.mdx`): JSON-RPC over stdio between editor/client and agent. Listed agents include Claude Agent "via Zed's SDK adapter", Codex CLI "via ACP's adapter", Gemini CLI, Kimi CLI, OpenCode, Cursor, Copilot (public preview), Goose and many more. Governance: RFD process with "Draft -> Preview -> Completed (the only state that can represent a 1-way door)". Cadence: 14 `schema-v1.*` tags from 1.13.7 (2026-06-16) to 1.24.1 (2026-09-30), 7 `schema-v2.0.0-alpha.*` tags; "ACP v2 is available in Draft" announced 2026-07-20, with deliberate breaking changes (prompt lifecycle decoupled from turns, ID-patched messages, structured diffs replacing `oldText/newText`, permission request shape). Rust and TypeScript SDKs hit 1.0 on 2026-06-25. Implication: ACP gives one wire format for session/prompt/permission across four harnesses, but the Claude and Codex paths depend on third-party adapters that track two moving upstreams.
- MCP [V] https://github.com/modelcontextprotocol/modelcontextprotocol tags and `docs/specification/2026-07-28/changelog.mdx`: dated spec revisions 2024-11-05, 2025-03-26, 2025-06-18, 2025-11-25, 2026-07-28. The 2026-07-28 revision removes protocol-level sessions, the `initialize` handshake, `ping`, `resources/subscribe`, SSE resumability, and moves tasks to an extension; adds required `resultType`, `server/discover`, `subscriptions/listen`. An MCP server in the messaging tool must negotiate both generations.
- A2A [V] tag list of https://github.com/a2aproject/A2A: v0.1.0 (2025-05-19) ... v0.3.0 (2025-07-31), v1.0.0-rc (2026-01-29), v1.0.0 (2026-03-12), v1.0.1 (2026-05-28); canonical spec is `specification/a2a.proto`. It is an agent-to-agent network protocol, not a coding-CLI headless interface; none of the four harnesses was verified to speak A2A [U].
- AGENTS.md [V] https://github.com/agentsmd/agents.md README: "a simple, open format for guiding coding agents" (a README for agents); Codex's repo ships `docs/agents_md.md`. It is a convention for instructions, not an event or process contract; Claude Code reads CLAUDE.md (AGENTS.md support in Claude Code, Kimi and OpenCode loading behaviour not verified [U]; kimi-code changelog mentions loading AGENTS.md files [V]).

## 4. Corrections to wave-one docs

1. research/03 line 29: "(needs `--verbose`; ...) ... the complete schema is UNVERIFIED here". Now verified: the full `result`, `system/init`, `api_retry` schemas are in section 2.1. The `--verbose` requirement is still not stated by the docs page (only used in its example) [U].
2. research/03 line 29: "A docs page also mentions a `--json-schema` change in v2.1.205 (UNVERIFIED...)". Verified: headless docs say invalid schemas were silently ignored before v2.1.205 and `format` was rejected.
3. research/03 line 122: "Which of Codex, Kimi, OpenCode expose published event schemas vs needing inference? (Not researched)". Answer: Codex has a typed, TS-RS-exported `ThreadEvent` (use the source/TS SDK `events.ts`); Kimi legacy documents message and Wire 1.10 types in prose; kimi-code documents message shape in prose and the web API via live OpenAPI/AsyncAPI; OpenCode has no schema for `run --format json` (only the generated OpenAPI for the server), so infer.
4. research/05 section B: "Failure: Kimi CLI release hangs on non-interactive mode... Constrain `kimi-cli != X`". The package is archived (final 1.52.0, 2026-09-22), so no future bad-version list for it is meaningful; the live upstream is `@moonshot-ai/kimi-code`, with a different CLI. Also "npm/GitHub versions" for bisecting kimi-cli is wrong: legacy is on PyPI.
5. research/05 section B: "(npm for some, GitHub releases for others, [U: channels vary])". Now verified: Claude Code npm `@anthropic-ai/claude-code`; Codex npm `@openai/codex`; OpenCode npm `opencode-ai`; kimi-code npm `@moonshot-ai/kimi-code`; kimi-cli PyPI `kimi-cli`. Changelog locations differ (in-repo CHANGELOG.md for Claude Code and Kimi; commit messages/GitHub releases for Codex and OpenCode).
6. research/05 section B: "Failure: Codex's new release changes an event field name. Class A... adapter parses both old and new field names". Real precedent exists (thread.started vs session.created) but occurred inside 8 days of introduction; the more recent real Codex break was flag removal (`--full-auto`), which an adapter-field-alias strategy would not catch. The contract must include the invocation flags, not only event fields.
7. research/02 line 19: "`stream-json` ... use with `--verbose --include-partial-messages`". `--include-partial-messages` only adds `stream_event` deltas; it is not needed for complete messages [partly V: the docs call it the way to receive tokens as generated].
8. research/01 line 54: "`claude -p` runs the full agent loop and exits with a status" cited to third-party blogs (SFEIR, hidekazu-konishi). Primary source is https://code.claude.com/docs/en/headless; exit status detail is limited to 0/non-zero/143 [V].
9. BUILD_PLAN section 4 item 1 groups "Kimi" as one CLI. Split into legacy kimi-cli (tombstone) and kimi-code.

## 5. Implications and concrete edits to /home/user/dfexplore/docs/BUILD_PLAN.md

1. Section 4, item 1 (second-pass research): replace with "DONE in W2-01: contracts verified for Claude Code, Codex, OpenCode, Kimi (legacy archived; kimi-code successor). Remaining: live runtime checks (see W2-01 section 7)."
2. Phase 3 (line 42): after "per-harness consumer contracts", add: "harness set is Claude Code, Codex, OpenCode, kimi-code (kimi-cli is archived; keep one frozen legacy adapter only if users still run it). Each contract covers invocation flags (not just event schema), exit codes, session round-trip, config/auth paths, and a version-gated feature table (see W2-01 section 6)."
3. Phase 3 exit criterion: add "planted flag removal (e.g. a shim that rejects a previously valid flag) is classified `breaking`, and a planted additive event field is classified `additive` without failing the build."
4. Canary schedule: change "live smoke on new versions only" to "live smoke on every new stable version of each harness within 24 hours of publication" given 39-78 releases per 90 days per CLI; use cheap prompts and a throwaway account; record the dist-tag and version.
5. Open decision list (section 3): add "OPEN: Messaging-tool transport per harness: native CLI JSON streams (4 adapters) vs ACP (native for OpenCode and Kimi; adapters for Claude Code and Codex) vs Codex app-server. Recommendation: keep CLI-JSON adapters as the baseline, add an ACP adapter behind a feature flag and run the same conformance scenarios against both."
6. Constraints registry (doc 05): add per-harness `deprecation_window` field; Codex precedent is 93 days between deprecation (2026-04-28) and removal (2026-07-30), so a watcher on "deprecated" in release notes/commit messages should open a goal at deprecation time.
7. Supply-chain note (infrastructure/escalation): kimi-cli 1.52.0 executes a CDN-fetched installer when run with no arguments. The factory must never invoke bare `kimi`; pin `kimi-cli` at <=1.51.0 if used at all and always pass `--version`/`-p` explicitly.
8. Section 5 tensions: add "MCP spec 2026-07-28 and ACP v2 are both breaking generations in flight: the messaging tool needs `tested_range` per protocol revision, not just per CLI."

## 6. Observable-contract checklist per harness (conformance suite seed)

Common to all (assertions on structure, not text): (a) `--version` parses to semver; (b) run with a deterministic trivial prompt (e.g. "reply with the single word OK") and assert event grammar order; (c) first/last event types; (d) session id present and resumable in a second process with a follow-up that references round-trip state; (e) exit code for: success, bad auth, bad flag, nonexistent session, SIGTERM/SIGINT; (f) unknown keys tolerated, required keys present; (g) stdout contains only protocol lines (stderr carries logs); (h) output flush on slow reader (Claude 2.1.214, 2.1.208 precedent); (i) `--help` flag set diff, with a "flags we use still exist" assertion; (j) no hang: process exits within N seconds of final event (Claude 2.1.153, Kimi `headless-exit.ts` precedent); (k) permission behaviour under non-interactive mode (denied vs auto-approved) because defaults changed in Claude 2.1.285 and Kimi 1.40.0.

Claude Code:
- C1 `claude -p X --output-format stream-json --verbose` (record whether `--verbose` is required by running without it).
- C2 first non-hook event is `system/init` with `session_id`, `claude_code_version`, `tools`, `mcp_servers`, `plugins`, `permissionMode`; absent optional fields (`capabilities`, `plugin_errors`, `mcp_server_errors`) tolerated; if `capabilities` present, log unknown values.
- C3 last line `type=result` with `subtype`, `is_error`, `session_id`, `total_cost_usd`, `num_turns`, `permission_denials`; error subtype set within the 4 known values plus unknown tolerated.
- C4 `--output-format json` returns one object with `result` and `session_id`; `--json-schema` returns `structured_output`; invalid schema exits non-zero with the documented message.
- C5 `--resume <session_id>` from a different cwd works (>=2.1.223); `--resume` bogus id yields `No conversation found with session ID` and non-zero.
- C6 `--permission-prompts none` accepted (>=2.1.259); `permission_denied` events emitted and `permission_denials` populated.
- C7 `--bare` with `ANTHROPIC_API_KEY`: no hooks/MCP from project; `system/init` `mcp_servers` empty.
- C8 SIGTERM exit 143 with no result line; SIGINT yields a result.
- C9 `api_retry` field set (simulate with an unreachable base URL: `error`, `attempt`, `retry_delay_ms`).
- C10 transcript path `~/.claude/projects/<project>/<id>.jsonl` exists, but parsing it is out of contract (docs say internal).
- C11 hook contract: a `Stop` hook with exit 2 keeps the agent going; stdin contains `session_id`, `transcript_path`, `cwd`, `hook_event_name`.

Codex:
- X1 `codex exec --json --skip-git-repo-check --ephemeral -s read-only "X"`: first event `thread.started` with `thread_id`; then `turn.started`; ends with `turn.completed{usage}` or `turn.failed`/`error`.
- X2 item type set subset of {agent_message, reasoning, command_execution, file_change, mcp_tool_call, collab_tool_call, web_search, todo_list, error}; every `item.completed` has an `id` that appeared in `item.started` or is a single-shot (file_change).
- X3 `--output-schema` + `-o file` writes the final message that validates against the schema.
- X4 `codex exec resume <thread_id> "follow-up"` and `resume --last`; `fork` exists.
- X5 exit 1 on server error (auth failure with a bad key), 0 on success.
- X6 flags we depend on exist in `codex exec --help`: `--json`, `--sandbox`, `--output-schema`, `--cd`, `--skip-git-repo-check`, `--ephemeral`, `--model`, `--profile`; assert `--full-auto` is absent in >= the removal version and that adapter never emits it.
- X7 `--strict-config` rejects an unknown `config.toml` key; `$CODEX_HOME` redirects config and sessions.
- X8 `--experimental-json` alias still accepted (or adapter uses `--json`).

OpenCode:
- O1 `opencode run --format json "X"`: every line parses as JSON with `type`, `timestamp`, `sessionID`; type in {tool_use, step_start, step_finish, text, reasoning, error}; `text` lines have `part.text`.
- O2 permission request without `--auto` auto-rejects (no hang); with `--auto` proceeds.
- O3 `--session <id>` and `--continue` reuse the session; `--fork` requires one of them (exit 1 otherwise).
- O4 error path exits 1 (bad model string, missing session).
- O5 `opencode serve` on `127.0.0.1:4096` serves `/doc` as OpenAPI 3.1, auth with `OPENCODE_SERVER_PASSWORD` returns 401 without credentials; `/event` stream (SSE) delivers `message.part.updated` and `session.status`; diff path/schema count against previous baseline and report removals only.
- O6 config merge order test: global < `OPENCODE_CONFIG` < project < `.opencode/` < `OPENCODE_CONFIG_CONTENT`; `$schema` validates; deprecated keys accepted with a warning.
- O7 `opencode acp` handshake with a minimal ACP client (initialize, session/new, session/prompt).

Kimi (two adapters):
- K1 (kimi-cli <=1.52.0, legacy): `kimi --print -p X --output-format stream-json` lines match `{role, content[, tool_calls]}`; tool message has `tool_call_id`; exit 0; bogus auth gives exit 1; simulated 429 gives 75 (mock provider).
- K2 legacy `--input-format stream-json` multi-turn loop terminates on stdin close.
- K3 legacy `--wire` `initialize` returns `protocol_version` and `server.version`; assert negotiated version >= 1.7 and unknown events tolerated.
- K4 (kimi-code): `kimi -p X --output-format stream-json`; text mode keeps assistant text on stdout and thinking on stderr; `--output-format` without `-p` is rejected; `-p` with `--yolo` is rejected; exit codes recorded (unknown, so snapshot them as baseline rather than assert).
- K5 `kimi -c` and `--session <id>` resume; `kimi acp` handshake; `kimi migrate` leaves `~/.kimi` untouched.
- K6 data root: sessions/config under `~/.kimi-code/` (and `~/.kimi/` for legacy).
- K7 canary guard: never run bare `kimi` for legacy 1.52.0.

Protocol layer: ACP conformance via a minimal client against `opencode acp`, `kimi acp`, the claude-agent-acp and codex-acp adapters (report ACP `protocolVersion` negotiated); MCP: `server/discover` and `initialize` both handled depending on negotiated revision.

## 7. Remaining unverified items

- Claude Code: exit code per `result` error subtype; whether `--verbose` is required for stream-json; managed-settings paths and credential store locations; hooks beyond the first 100,000 characters of the hooks page; whether `--include-partial-messages` is accepted without `--verbose`.
- Codex: auth file path and `CODEX_API_KEY`/login flow (docs on developers.openai.com unreachable); sandbox mode enum values; exec's runtime treatment of `approval_policy` when unset; release notes format on GitHub Releases.
- Kimi: kimi-code stream-json exact message schema (`PromptJsonWriter` source was not read; partial clone fetch timed out), kimi-code exit codes, whether `--print` is accepted as a hidden alias in 2.x, kimi-code retry exit semantics.
- OpenCode: GitHub release-note format, whether `run --format json` is documented anywhere as stable, SSE `/event` schema details, full list of `part` types per version.
- A2A support in any of the four harnesses; AGENTS.md handling in Claude Code/OpenCode/Kimi legacy; ACP adapter version-compatibility matrix (claude-agent-acp, codex-acp).
- Empirical runs: I did not execute any harness binary, so every runtime behaviour above is from source/docs, not observation.

## 8. Sources

Fetch failures: developers.openai.com (proxy 403 and DNS failure), opencode.ai (ENOTFOUND), `gh api` and GitHub MCP for openai/codex (access denied for session), github.com HTML (403). Worked: code.claude.com, raw.githubusercontent.com, `git clone --bare --filter=blob:none` of public repos, registry.npmjs.org, pypi.org.

- https://code.claude.com/docs/en/headless
- https://code.claude.com/docs/en/agent-sdk/typescript
- https://code.claude.com/docs/en/sessions
- https://code.claude.com/docs/en/settings-reference
- https://code.claude.com/docs/en/hooks
- https://raw.githubusercontent.com/anthropics/claude-code/main/CHANGELOG.md
- https://registry.npmjs.org/@anthropic-ai/claude-code , /@anthropic-ai/claude-agent-sdk , /@openai/codex , /opencode-ai , /@moonshot-ai/kimi-code ; https://pypi.org/pypi/kimi-cli/json
- https://github.com/openai/codex (files: codex-rs/exec/src/exec_events.rs, codex-rs/exec/src/cli.rs, codex-rs/utils/cli/src/shared_options.rs, codex-rs/tui/src/cli.rs, sdk/typescript/src/exec.ts; commits cc1b21e47, 4a80059b1, ea82f8666, c405d8c06, #20133, #36054, #15424, #23883)
- https://github.com/sst/opencode (packages/opencode/src/cli/cmd/run.ts; packages/web/src/content/docs/server.mdx, acp.mdx, config.mdx; packages/sdk/openapi.json; commits 449994f120, 1275c71a6)
- https://github.com/MoonshotAI/kimi-cli (CHANGELOG.md; docs/en/customization/print-mode.md, wire-mode.md; docs/en/configuration/data-locations.md; docs/en/reference/kimi-command.md)
- https://github.com/MoonshotAI/kimi-code (docs/en/reference/kimi-command.md, server-api.md; docs/en/guides/migration.md; docs/en/release-notes/changelog.md; apps/kimi-code/src/cli/options.ts, headless-exit.ts, v2/run-v2-print.ts)
- https://github.com/agentclientprotocol/agent-client-protocol (tags; docs/announcements/acp-v2-draft.mdx, sdk-1-0-releases.mdx; docs/get-started/agents.mdx; docs/rfds/about.mdx)
- https://github.com/modelcontextprotocol/modelcontextprotocol (tags; docs/specification/2026-07-28/changelog.mdx)
- https://github.com/a2aproject/A2A (tags; specification/a2a.proto path)
- https://github.com/agentsmd/agents.md (README.md)
