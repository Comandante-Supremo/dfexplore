# A-01: Audit of W2-01 (harness CLI headless contracts)

Auditor: independent model, 2026-10-07. Method: fresh blobless bare clones (scratchpad, not executed) of anthropics/claude-code (HEAD 79babc37, 2026-10-07), openai/codex (f73a478c, 2026-10-07), sst/opencode dev (a697115b, 2026-10-07), MoonshotAI/kimi-cli (9ab1286b = tag 1.52.0), MoonshotAI/kimi-code (21406fb4, 2026-09-30; tag `@moonshot-ai/kimi-code@2.1.1` present), agentclientprotocol/agent-client-protocol, modelcontextprotocol/modelcontextprotocol, a2aproject/A2A; npm registry JSON for the four npm packages; PyPI JSON for kimi-cli; code.claude.com `headless.md` and `agent-sdk/typescript.md`. No binary, installer or harness CLI was run. GitHub API for kimi-cli returned 403 (not routed around), so the repo's "archived" flag itself was not read; the archive claim rests on the CHANGELOG and source.

## Verdict: TRUST WITH CAVEATS

The factual core is solid: 13 of 15 checked claims are supported from source, and the doc is honest about not running anything. Three problems must be fixed before BUILD_PLAN uses it:
1. **Kimi legacy is misdescribed in a way that matters for safety.** In 1.52.0, *every* entry point is short-circuited, not only bare `kimi`. `-p`, `--print`, `acp` and `--help` all print a one-line deprecation notice and **exit 0**. So K1-K3 ("kimi-cli <=1.52.0") cannot pass on 1.52.0. Proposed edit 7 ("always pass `--version`/`-p` explicitly") would also produce silent false successes.
2. **"4 Codex schema renames" is overstated.** There are 4 wire-level renames, but they come from only 2 commits. Of the other two commits the doc lists, #4482 renamed only Rust/TS type names and #4309 added the exit code. "14 additive commits" after the renames is really 12, and one of them (#13850) adds an enum value (`Interrupted`), which breaks strict-enum consumers.
3. **Self-updaters are ignored.** Pinning and the canary model fail if harnesses update themselves. kimi-code auto-installs by default with staged rollout, and Claude Code has an updater too.

## Claims table

| # | Claim | Rating | Evidence |
|---|---|---|---|
| 1 | kimi-cli archived; last release 1.52.0 on 2026-09-22 | SUPPORTED (archive flag itself not read: API 403) | kimi-cli `CHANGELOG.md` "## 1.52.0 (2026-09-22) ... archived and no longer maintained, and this is the final release"; tag 1.52.0 = 9ab1286b (2026-09-22 09:29Z); PyPI latest 1.52.0, upload 2026-09-22T10:03Z; 1.51.0 uploaded 2026-09-21. |
| 2 | Successor on npm as `@moonshot-ai/kimi-code` 2.1.1, where `-p` takes the prompt | SUPPORTED | npm `dist-tags.latest`=2.1.1, time 2026-09-24T07:27Z; 83 versions since 2026-05-21. kimi-code `docs/en/reference/kimi-command.md:21` "`--prompt <prompt>` \| `-p`"; `:43` "`--prompt` cannot be used with `--yolo`, `--auto`, or `--plan`"; `apps/kimi-code/src/cli/options.ts:80-86` (OptionConflictError). Not in W2-01: `KIMI_MODEL_OUTPUT_FORMAT` env sets the `-p` output format (`options.ts:5`), an ambient-environment risk for the contract. |
| 3 | 1.52.0 runs a CDN-fetched installer when `kimi` is run bare | SUPPORTED, and the doc understates it | `src/kimi_cli/__main__.py` @1.52.0 `_deprecation_gate`: `if not args: return run_kimi_code_installer()`, and every other argument list prints the notice and returns 0. `src/kimi_cli/deprecation.py` runs `bash -o pipefail -c <install_sh>` with no confirmation. `install_sh` is a **full shell command string** read from `https://cdn.kimi.com/kimi-code-tips/kimi_cli/migration.json` (`ui/shell/update.py:33,183,284`), cached to disk, with a default of `curl -fsSL https://code.kimi.com/kimi-code/install.sh \| bash`. The CDN can therefore change the command that runs. 1.51.0 also runs the CDN-supplied string, but only from the interactive `/upgrade` path (`update.py:459` @1.51.0). |
| 4 | Claude Code `result`, `system/init` and `api_retry` event shapes | SUPPORTED | `headless.md:229-240` api_retry table (attempt, max_retries, retry_delay_ms, error_status, no_response? v2.1.261+, 12-value `error` enum, uuid, session_id). `typescript.md:1556-1612`: result subtypes (success plus the 4 error subtypes) and the 19-value `terminal_reason`. `:1758-1804`: init `apiKeySource`, `claude_code_version`, `plugin_errors?`, `capabilities` (v2.1.205+). Missed by W2-01: `startup_failure_reason` on `error_during_execution` (SDK 0.3.274+), and `plugin_install` events that can precede `system/init` (`headless.md`, "Read session metadata"). |
| 5 | 3 pre-2.0 breaking headless-output changes | PLAUSIBLE, mislabelled | CHANGELOG: 0.2.117 "Breaking change: --print JSON output now returns nested message objects" (real, labelled); 1.0.22 "SDK: Renamed `total_cost` to `total_cost_usd`" (breaking but unlabelled). 0.2.66 "Print mode (-p) now supports streaming output" is a feature addition, not a break. Correct count: 2 breaks. "No Breaking entry since 2.0 except 2.1.75" is SUPPORTED: the only "Breaking change:" lines are at CHANGELOG lines 6322 (2.1.75), 8367 (0.2.125, Bedrock ARN) and 8372 (0.2.117). |
| 6 | `-p` defaults to auto mode from 2.1.285 | SUPPORTED (scoped) | CHANGELOG 2.1.285 (line 769): "Changed `claude -p` and Python Agent SDK sessions on third-party providers or with telemetry off to start in auto mode when no permission mode is configured". The scope is third-party providers or telemetry off, as W2-01 says. A factory that disables telemetry is in scope, so it must always pass `--permission-mode`. |
| 7 | Codex ThreadEvent shape; exit 1 on failure | SUPPORTED | `codex-rs/exec/src/exec_events.rs` @HEAD: `serde(tag="type")` with variants thread.started{thread_id}, turn.started, turn.completed{usage incl. cache_write_input_tokens and reasoning_output_tokens}, turn.failed{error}, item.started, item.updated, item.completed, error. Items use tag `type`: agent_message, reasoning, command_execution (status in_progress/completed/failed/declined), file_change, mcp_tool_call, collab_tool_call, web_search, todo_list, error. `exec/src/lib.rs`: 13+ `std::process::exit(1)` sites. Commit cc1b21e47 (2025-09-26) "ensures we return 1 on failures" adds test `exits_non_zero_when_server_reports_error`. |
| 8 | 4 Codex schema renames, 2025-09-25 to 2025-10-02 | PLAUSIBLE, overstated | Wire renames: 4a80059b (#4478, 2025-09-29) `session.created{session_id}` -> `thread.started{thread_id}`; c405d8c0 (#4610, 2025-10-02) tag `item_type` -> `type` and `AssistantMessage` -> `AgentMessage`. That is 4 wire renames in 2 commits. ea82f866 (#4482) has no serde/string diff, so the rename is Rust/TS type names only. cc1b21e4 (#4309) is the exit-code fix, not a rename. The file has 20 commits in total; 12 come after 2025-10-02, not 14. |
| 9 | `--full-auto` deprecated 2026-04-28, removed 93 days later | SUPPORTED | 3d10ba9f "chore(cli) deprecate --full-auto (#20133)", 2026-04-28; 1c5f336c "Remove legacy `--full-auto` handling from `codex exec` (#36054)", 2026-07-30: "Callers must now select the sandbox mode explicitly". 2026-04-28 to 2026-07-30 is 93 days. Current `exec/src/cli.rs` has `long="json", alias="experimental-json"` and no full-auto. |
| 10 | OpenCode `run --format json` envelope {type, timestamp, sessionID, part} | SUPPORTED | `packages/opencode/src/cli/cmd/run.ts:678-690`: `emit()` writes `{type, timestamp: Date.now(), sessionID, ...data}`; calls at :725-767 pass `{part}`; `error` passes `{error: props.error}`. Origin 449994f1 (#2471, 2025-09-24); step_start/step_finish added 8a28d34f (2025-09-27); rewrite 1275c71a (2026-02-02). `--fork` without `--continue`/`--session` calls `process.exit(1)` (:425-427). |
| 11 | Release counts in the last 90 days: Claude Code 78, Codex 39, OpenCode 41 | SUPPORTED (boundary-sensitive) | With the window starting 2026-07-09T00:00Z, my count of stable x.y.z versions gives 78, 39 and 41. With a strict 90x24h window back from 2026-10-07 23:59Z it gives 77, 37 and 38. Totals 539, 200 and 944 match. Missed by W2-01: Claude Code's npm `stable` dist-tag is **2.1.285** while `latest` is 2.1.293. A distinct stable channel exists, and the canary should record both. |
| 12 | ACP schema v1.24.1 and v2 alpha.7 | SUPPORTED | tags `schema-v1.24.1` and `schema-v2.0.0-alpha.7`, both 2026-09-30; 14 v1 tags since 2026-06-16; 7 v2 alpha tags. Note: `docs/get-started/agents.mdx` still links the archived kimi-cli, not kimi-code. kimi-code's own `kimi acp` is documented (`kimi-command.md:148`; changelog says ACP 0.23). |
| 13 | MCP 2026-07-28 revision removes sessions and `initialize` | SUPPORTED | `docs/specification/2026-07-28/changelog.mdx` major changes 1 (sessions, `Mcp-Session-Id`), 2 (initialize handshake), 3 (`server/discover`), 4 (`subscriptions/listen`), 5 (ping), 7 to 9 (MRTR replaces server-initiated sampling and elicitation; required `resultType`; no SSE resumability). Tag 2026-07-28 (RC 2026-05-29). W2-01 omits MRTR, which also breaks any server using sampling or elicitation. |
| 14 | A2A v1.0.1 | SUPPORTED | tag v1.0.1 2026-05-28; v1.0.0 2026-03-12; v1.0.0-rc 2026-01-29. |
| 15 | Kimi legacy exit codes 0/1/75 (1.27.0) are usable for K1 on "<=1.52.0" | CONTRADICTED for 1.52.0 | As in row 3: 1.52.0 never reaches the Typer CLI (`run_original_cli` is "intentionally not called"). Any argument list, including a failing one, exits 0 after printing the notice. The 0/1/75 contract holds only for 1.27.0 to 1.51.0. |

Spot checks also SUPPORTED: Claude 2.1.205 json-schema behaviour (`headless.md:160`), SIGTERM gives 143 (`:85`), 10MB stdin cap (`:115`), `--bare` "will become the default for `-p`" (`:70`), `mcp_server_errors` v2.1.219 (`:271`), `--permission-prompts none` in 2.1.259, 2.1.290 json-schema exit fix, kimi-code 0.28.0 "**Breaking:**" (changelog:550-554), kimi-code 2.0.0 dated 2026-09-17.

## Tag-integrity findings

- No runtime behaviour is presented as observed. Section 7's last bullet ("I did not execute any harness binary") and section 2.2's note ("runtime behaviour not run") are correct, and source-derived behaviour (OpenCode auto-reject, Codex exit(1)) is attributed to files.
- Untagged inferences that should be [U]:
  - 2.2: exec on app-server is a "likely future replacement for `--json`" (speculation).
  - Summary: Kimi "emits OpenAI-style JSONL" stated for kimi-code, but `PromptJsonWriter` was not read (section 7 admits this).
  - 2.5: "Afterwards additive" for Codex. #13850 adds an enum value, and 66447d5d (#10349) swaps the MCP types crate. Neither was shown to be wire-neutral.
- Mislabelled: 2.5 counts 0.2.66 (a feature) as a "headless-output break", and the summary's "4 schema renames" counts a type-name-only commit and an exit-code fix.
- Incomplete [V]: 2.3 tags the 1.52.0 behaviour [V] from the CHANGELOG but misses the source behaviour for every non-bare invocation (row 15). The CHANGELOG line it quotes is accurate; the conclusions drawn from it in section 5 item 7 and K1 are wrong.
- "Stable" is overloaded. W2-01 means non-prerelease semver, but Claude Code has an npm `stable` dist-tag (2.1.285) that differs from `latest`.

## Safety findings

Confirmed hazard, larger than stated:
- **1.52.0 with no args** fetches a JSON from `cdn.kimi.com` and runs its `install_script.sh` field through `bash -c`, unconfirmed. This is remote code execution controlled by a CDN payload, not a fixed script URL. On CDN failure it falls back to a cached copy and then to `curl ... install.sh | bash`.
- **1.52.0 with any args** exits 0 with no work done. Conformance or factory runs would record false success.
- **1.51.0 and earlier**: the interactive `/upgrade` path (since about 1.47.0) also runs the CDN-supplied string. Headless `--print` is not affected, but a TTY session in a sandbox could trigger it.
- **kimi-code** auto-update (`[upgrade].auto_install`) is on by default, with staged rollout (`docs/en/configuration/data-locations.md:64,97`). It is disabled by `KIMI_CODE_NO_AUTO_UPDATE` (`env-vars.md:184`), and `kimi upgrade` exists.
- **Claude Code** has `DISABLE_AUTOUPDATER` and the stricter `DISABLE_UPDATES` (CHANGELOG line 5334).
- Both generations install a binary named `kimi`, so PATH order decides which one runs.

Controls that follow (for the conformance suite and the factory sandbox):
1. Ban `kimi-cli==1.52.0` in the constraints registry (reason: non-functional plus unconfirmed CDN exec). The legacy adapter, if kept, pins `kimi-cli==1.51.0` by hash (`uv`/`pip --require-hashes`).
2. The adapter refuses to launch any harness with an empty argv, and invokes harnesses by absolute path, never by `kimi` on PATH.
3. Add a liveness assertion to every live smoke run: a success exit with no protocol lines on stdout is a contract failure. This catches the 1.52.0 pattern generally.
4. Run harness sandboxes with default-deny egress, allowing only the provider API hosts. Block `cdn.kimi.com`, `code.kimi.com` and the npm, PyPI and GitHub release hosts at run time; installs happen only in a separate, pinned install step.
5. Set `DISABLE_UPDATES=1` (Claude Code) and `KIMI_CODE_NO_AUTO_UPDATE=1` (kimi-code) in every harness environment, and make the suite assert `--version` before and after each run. A Codex/OpenCode updater equivalent was not checked [U].
6. Use a read-only harness install and a throwaway HOME per run (`CLAUDE_CONFIG_DIR`, `CODEX_HOME`, `KIMI_SHARE_DIR`), so cached tips files and updater state cannot persist.
7. Always pass an explicit permission mode (Claude `--permission-mode`, Codex `--sandbox`). Claude 2.1.285 silently changes the default when telemetry is off.

## Overreach findings

BUILD_PLAN edits (W2-01 section 5):

| # | Edit | Decision | Reason |
|---|---|---|---|
| 1 | Mark section 4 item 1 DONE | MODIFY | Write "source/doc-verified in W2-01; runtime unverified". Exit codes per subtype, the `--verbose` requirement, kimi-code exit codes and stream schema are still open. |
| 2 | Harness set plus contract scope (flags, exit codes, sessions, config, version gates) | ACCEPT, with a fix | The legacy adapter is capped at `kimi-cli==1.51.0`; 1.52.0 is banned. |
| 3 | Phase 3 exit: planted flag removal is `breaking`, additive field is `additive` | ACCEPT | Directly evidenced by `--full-auto` (#36054). Also plant a new enum value (Codex #13850 pattern) and require `ambiguous` or `breaking`, not silent pass. |
| 4 | Live smoke on every stable release within 24h | MODIFY | The cadence is evidenced. "Throwaway account" conflicts with the API-key auth decision; use a dedicated low-cap API key. For Claude Code, track the `stable` and `latest` dist-tags separately. Batch same-day releases. |
| 5 | OPEN decision: CLI-JSON vs ACP vs Codex app-server | ACCEPT | Evidence supports both ACP's reach (agents.mdx) and its churn. The recommendation (CLI-JSON baseline, ACP behind a flag) is reasonable. |
| 6 | `deprecation_window` field; watch for "deprecated" | MODIFY | Accept the field and the watcher. Do not use 93 days as a default: it is n=1. |
| 7 | Kimi supply-chain note | MODIFY | It has the wrong safe-usage rule ("pass `--version`/`-p`" makes 1.52.0 exit 0 silently). Replace it with Controls 1 to 6 above. |
| 8 | `tested_range` per protocol revision (MCP 2026-07-28, ACP v2) | ACCEPT, conditional | Applies only if the messaging tool exposes MCP or ACP. Add MRTR (sampling and elicitation) to the MCP item. |

Contract assertions (section 6, about 45). Most restate source or docs. These are guesses or need rewording:
- **Wrong or will false-fail:**
  - X2 "every item.completed id appeared in item.started or is file_change". `event_processor_with_jsonl_output.rs` pushes bare `ItemCompleted` at lines 382, 411, 434, 466, 495 and 515 (several item kinds). Rewrite as: ids are unique, and a started id is eventually completed.
  - K1-K3 "<=1.52.0". Must be "<=1.51.0".
  - C2 "first non-hook event is system/init". `plugin_install` events can also precede it.
- **Guesses (snapshot as baseline, do not assert):**
  - C5 non-zero exit for a bogus `--resume` id (the message is documented; the exit code is not).
  - C8 "SIGINT yields a result" (the docs say the turn ends; emission of a `result` line is inferred).
  - C9 an unreachable base URL produces `api_retry` (it may instead give `error_status: null` with `no_response`, or no retry).
  - X5 bad key gives exit 1 (exit sites are verified; the auth path is not traced).
  - X7 `$CODEX_HOME` redirects sessions.
  - O4 bad model string gives exit 1.
  - O5 401 without credentials, and the `/event` schema.
  - O6 "deprecated keys accepted with a warning".
  - K3 "negotiated >= 1.7" (an arbitrary threshold).
  - K4 kimi-code exit codes (W2-01 already says snapshot).
  - K5 "`kimi migrate` leaves `~/.kimi` untouched".
- **Missing:** a liveness assertion (exit 0 with protocol output); an updater-off assertion (version unchanged after a run); `KIMI_MODEL_OUTPUT_FORMAT` and other ambient-env overrides cleared in the harness env.

## Consistency with BUILD_PLAN.md and doc 03

- Doc 03 layout `harnesses/{claude-code,codex,kimi,opencode}` should become `kimi-code/` plus an optional `kimi-cli-legacy/` (frozen at 1.51.0, no `latest` matrix leg, since 1.52.0 is final and banned).
- Doc 03 `test_event_grammar.py` ("init -> events* -> single terminal") does not fit all harnesses. OpenCode `run --format json` has no init or terminal event (only part events, ending at idle and process exit; `run.ts:678-767`). Kimi stream-json is bare role messages. Codex ends with `turn.completed` or `turn.failed`, possibly after `error` events. Make the grammar a per-harness parameter, and use process exit plus exit code as the terminal signal where no terminal event exists. W2-01 does not flag this.
- The doc 03 line 29 claim "needs `--verbose`" stays [U]. W2-01 correctly declines to confirm it, and C1 tests it.
- BUILD_PLAN 1 "Execution: `claude -p --bare`" is consistent. But 2.1.286 changed `--bare` semantics (only CLI-named MCP servers; no system reminders; no background tasks), so record a version gate on `--bare` in `contract.yaml`.
- BUILD_PLAN Phase 3 "live smoke on new versions only" is compatible with edit 4. Doc 03's "nightly at most" needs the same 24h wording.
- Harnesses not covered: Gemini CLI, Cursor CLI and GitHub Copilot CLI are all ACP-listed (`agents.mdx`), which makes an ACP adapter cover them cheaply. Goose is also listed. Aider is not ACP-listed and has no event stream in scope. W2-01 should at least state that the four-harness scope is the owner's choice and that ACP is the cheapest path to these extras. Also missing: harness self-updaters (Safety, item 5) and server-side model changes behind a fixed CLI version (doc 03 open question; still unaddressed).

## Recommended edits

BUILD_PLAN.md:
1. Section 4 item 1: "Source/doc-verified in W2-01 (audited A-01); runtime contract unverified; see W2-01 section 7." (MODIFY of W2-01 edit 1.)
2. Phase 3: accept W2-01 edits 2 and 3. Add "planted new enum value classified `ambiguous` or `breaking`" and "liveness: exit 0 without protocol output fails".
3. Phase 1 (sandbox runner): add "harness env sets `DISABLE_UPDATES=1`, `KIMI_CODE_NO_AUTO_UPDATE=1`, an explicit permission mode, per-run config/home dirs; default-deny egress except provider APIs; harnesses invoked by absolute path, never with empty argv." (Replaces W2-01 edit 7.)
4. Constraints registry seed: `kimi-cli==1.52.0` banned (reason: non-functional, unconfirmed CDN shell exec; evidence: `src/kimi_cli/deprecation.py`@1.52.0).
5. Accept W2-01 edits 5 and 8 (8 conditional on MCP/ACP exposure). Accept edit 6 without the 93-day default. Accept edit 4 with an API key instead of a "throwaway account" and with Claude `stable`/`latest` tracked separately.

Doc 03:
1. Line 29: replace "complete schema is UNVERIFIED" with a pointer to W2-01 section 2.1 (verified); keep `--verbose` as UNVERIFIED.
2. Line 122 open question: answer per W2-01 correction 3 (ACCEPT).
3. Section 7 layout: split `kimi` into `kimi-code` and `kimi-cli-legacy`. The `test_event_grammar.py` comment becomes "per-harness grammar; terminal = terminal event or process exit".
4. Section 3, live smoke: "within 24h of each new stable version" instead of "nightly at most"; add the liveness check.

W2-01 itself (for the researcher): fix the claims in rows 5, 8 and 15 and the "14 additive" count, and add the MRTR and `startup_failure_reason` omissions.
