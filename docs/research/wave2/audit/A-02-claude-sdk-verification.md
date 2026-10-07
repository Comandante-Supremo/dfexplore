# A-02: Audit of W2-02 (Claude SDK / Claude Code verification)

Auditor: independent model, 2026-10-07. Method: fetched the raw Markdown of the cited pages myself (`code.claude.com/docs/en/<page>.md`, `platform.claude.com/docs/en/<page>.md`), plus the support article HTML, two engineering posts, and three files from `anthropics/claude-code-action@main` through raw.githubusercontent.com. All were reachable, so most claims could be checked against primary text rather than summaries. I did not fetch the workflows, agent-loop, session-storage, Managed Agents budgets/self-hosted/outcomes pages, or the other two engineering posts.

## Verdict

**TRUST WITH CAVEATS.** Every decision-relevant claim I re-checked appears in the primary page. That covers prices, caps, hook semantics, bare-mode auth, the resume budget rule, rate limits, credit coverage and the terms quotes. The quotes are verbatim in the raw Markdown, even though the researcher fetched them through a summarizing tool. All five corrections to doc 02 are confirmed. There are three weaknesses:
1. The doc calls "SDK via API key against the linked Console org" the safe reading of credit coverage, but it missed a line on the credits page. That line says a Claude Code request in a credits-only org fails with "Credit balance too low". Coverage for `claude -p --bare` is therefore likely *no*, not just unclear. Whether SDK traffic counts as "SDK" or as "Claude Code" is untested.
2. It omits a Legal-page sentence that bears directly on the auth question.
3. Proposed edit 6 turns an owner decision (OPEN #2) into a "Decision". The other edits are proportionate.

## Claims table

| # | Claim (from W2-02) | Rating | Evidence |
|---|---|---|---|
| 1 | Sonnet 5.5 cache read "$0.10" (doc 02 said $0.20); Opus 5.5 $0.20; Fable 5.1 $0.25 | SUPPORTED | pricing.md row: "Claude Sonnet 5.5 \| $2 \| $2.50 \| $4 \| $0.10 / MTok<sup>2</sup> \| $10"; footnote: "0.05x the base input price". The full table (Haiku tiers, 1h writes) also matches. |
| 2 | `--max-budget-usd`: "totals restored from earlier runs don't count toward it"; subagent spend counts; v2.1.217+ | SUPPORTED | cli-reference.md, line 105, is word for word. |
| 3 | `--bare` skips auto-discovery, never reads OAuth/keychain, "recommended mode for scripted and SDK calls, and will become the default for `-p`" | SUPPORTED | headless.md lines 37, 49, 70. |
| 4 | Bare mode does not read `CLAUDE_CODE_OAUTH_TOKEN` or profiles/WIF | SUPPORTED | authentication.md line 310 ("Bare mode does not read `CLAUDE_CODE_OAUTH_TOKEN`") and line 257 (profiles/federation not read in bare mode). |
| 5 | Non-bare `-p` runs project hooks and `.mcp.json` "even in a folder you've never trusted" | SUPPORTED | headless.md line 41. |
| 6 | Stop hook: 8-consecutive-continuation cap, reset by a tool call, `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` | SUPPORTED | hooks.md line 2579 is word for word. |
| 7 | PreToolUse: non-2 non-zero exit does not block; a timed-out command/http/mcp_tool hook "doesn't block the call"; an SDK callback that times out does block | SUPPORTED | hooks.md lines 840, 849, 850, 1592. Line 837 adds something the doc missed: a mistyped hook path "leaves the gate silently disabled". |
| 8 | Auto-mode models on the Anthropic API: "Opus 4.6 or later, Sonnet 4.6 or later, Haiku 5.5, or a Fable model"; the stricter list applies to cloud providers | SUPPORTED | permission-modes.md line 322. The doc's correction 3 to doc 02 stands. |
| 9 | Thinking: Fable 5.1 and Opus 5.5 "Adaptive (always on)", Sonnet/Haiku 5.5 "Adaptive" | SUPPORTED | models/overview.md line 34. The inference that thinking can be disabled on Sonnet/Haiku 5.5 is correctly left [U]. |
| 10 | Logs exporters `otlp\|console\|none` (no Prometheus); port 9464 [U] | SUPPORTED (and the [U] can be cleared) | monitoring-usage.md line 108 for the logs exporters. Line 464: "scraped from `http://localhost:9464/metrics`". Doc 02's :9464 was right for metrics. |
| 11 | Credits: Max 5x $100, 20x $200, Team pool capped at $500, claim after 7 days; coverage table "Claude Code: No" | SUPPORTED, but the doc's reading is incomplete | api-credits page lines 18-20, 35, 40, 86-89. **Missed:** line 122: "If an organization holds only these credits and sends a request they don't cover, such as a Claude Code session, Claude Code displays: Credit balance too low". |
| 12 | Support article (2026-10-07): credits "cover the Claude Agent SDK, `claude -p`, the Claude API, and Claude Managed Agents"; the 2026-06-15 pause | SUPPORTED | Raw HTML of support.claude.com/.../15036540 matches. The two Anthropic pages really do conflict on `claude -p`. |
| 13 | Legal terms quotes (OAuth "exclusively for purchasers ... ordinary use"; Pro/Max limits "assume ordinary, individual usage"; enforcement "without prior notice") | SUPPORTED | legal-and-compliance.md lines 48, 54, 55, 59, verbatim. **Omitted:** line 57: "Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription". |
| 14 | Spend caps Start $500 / Build $1,000 / Scale $200,000; 429 `enforced_spend_limit_reached` with no `retry-after`; a self-set limit gives 400; Start tier 1,000 RPM / 2M ITPM / 400K OTPM per 5.5 model; Fable 500K ITPM | SUPPORTED | rate-limits.md lines 39-62, 83, 141-149. Opus 5.5 and Sonnet 5.5 each have their own bucket (footnotes 2-3). |
| 15 | SIGTERM gives exit 143 with no result; use SIGINT/`interrupt()` first | SUPPORTED | headless.md line 85. |
| 16 | `/goal` "is a wrapper around a session-scoped prompt-based Stop hook"; condition up to 4,000 chars | SUPPORTED | goal.md lines 64, 120. |
| 17 | `claude --bg` cannot combine with `-p`; reboot stops background sessions; the 48h failed/stopped rule | SUPPORTED | agent-view.md lines 483, 871-874. |
| 18 | Main-conversation 1h cache TTL only on a subscription within plan usage; API key gets 5m | SUPPORTED | prompt-caching.md line 268. |
| 19 | Safety-classifier fallback: Opus 5.5 → Opus 4.8 (cyber), → Opus 5 (bio); Sonnet 5.5 bio → refusal | SUPPORTED | model-config.md lines 519-522. |
| 20 | Managed Agents $0.08 per session-hour; worked example $0.705 | SUPPORTED | pricing.md lines 435, 452. |
| 21 | "Don't rely on session resume ... often more robust" | SUPPORTED | agent-sdk/sessions.md line 408. |
| 22 | claude-code-action self-hosted handling in source; docs say the action "runs on GitHub-hosted runners" | SUPPORTED | action.yml line 227; create-prompt/index.ts line 947; modes/agent/index.ts line 79; github-actions.md line 305. |
| 23 | Engineering posts: "80% of the variance", 4x/15x tokens, 90.2%, "most coding tasks", "a poor QA agent", "a strong lever", $9 vs $200 | SUPPORTED | Raw HTML of both posts. One nuance: the 90.2% figure is for Opus 4 lead + Sonnet 4 subagents, and the doc drops the model versions. |
| 24 | SessionStore best-effort mirror (3 attempts, `mirror_error`, a store resume deletes the local copy); workflow limits (16/1,000/4,096) | PLAUSIBLE-UNCHECKED | session-storage and workflows pages were not fetched by me. |
| 25 | Table row: "Managed Agents leases (about 60 s)" | UNSUPPORTED (in this doc) | The doc gives no source and no tag, and none of the Managed Agents pages it lists is cited for it. |
| 26 | Table row: "`/schedule` and routines need a claude.ai login, which API-key/profile auth doesn't have" | PLAUSIBLE-UNCHECKED | authentication.md says such features are unavailable under profile/federation. Extending that to API keys is an inference, though a reasonable one. |

## Tag integrity

- **Quotes are actually verbatim.** The doc admits that WebFetch returns model-summarized text. Even so, every quote I checked (about 25) matches the raw Markdown or HTML character for character. The [V] tags are justified for those items.
- **Partial reads are disclosed** (hooks 0-200K, agent-view 0-100K, and so on), and no claim I checked depends on an unread range. On line 57, "cli-reference (read characters 0-100000 of 133683)" mislabels the agent-view length: cli-reference is about 47K. This is cosmetic.
- **Support article [V]** is tagged with the caveat "only these excerpts verified". That is honest, and the excerpts check out.
- **[V] on "SDK-via-API-key ... is the safe reading" (correction 4)** is untagged reasoning, and it is weaker than presented. The credits page's "such as a Claude Code session" failure line suggests that `claude -p --bare` with an API key, in a credits-only org, fails outright. The Agent SDK runs the Claude Code binary underneath, so it is untested which bucket SDK traffic is billed under. Retag as [U].
- **[U] that can now be cleared:** Prometheus port 9464 (monitoring-usage.md line 464).
- **Untagged claims in the "provided vs build" table** (Managed Agents ~60 s leases) have no source. Tag them [U] or drop them.

## Overreach

- **Edit 6 (Auth)** replaces the "OPEN" label with "Decision: API key for all unattended runs". The evidence supports API key as the *recommendation*. Who decides billing is the owner's call (BUILD_PLAN section 3 lists it as an owner decision). The edit also says credits "can fund SDK/API usage", which overstates things given the credits-only "Claude Code session" failure line. It also leaves out Legal line 57, which explicitly permits an end user to use the unmodified Claude Code binary with their own subscription. That sentence weakens, but does not resolve, the "ordinary, individual usage" concern.
- **Edit 7** puts Start-tier numbers into the plan. They are verified today, but they depend on the tier and change over time, and a new org may sit in the lower Evaluation tier. Keep the edit's task wording, "record the org tier"; give the numbers as an example only.
- **Edit 10** targets "Models table in the plan (if present)". BUILD_PLAN has no models table; only line 26 names model roles. The retirement-floor calendar item and the classifier-fallback route class are good regardless of evidence.
- **Edit 4** (supervisor loop, Stop hook as backstop) and **edit 5** (deny rules plus exit 2, startup self-test) are sound architecture independent of the exact cap number. Hooks.md line 837 (a mistyped path silently disables the gate) strengthens the self-test.
- **Edit 2(b)** (prefer a fresh seeded session over `--resume`) follows directly from Anthropic's own "Don't rely on session resume". It is good regardless of evidence.
- **Edit 8** sets a 1h TTL for both cache buckets by default. On API-key billing, a 1h write costs 2x versus 1.25x for 5m, so it only pays when attempts actually idle for more than 5 minutes (waiting on CI). Make it conditional per attempt type and measure it in Phase 1, as the edit half-says.
- **Edit 14** says to avoid multi-agent for coding. That generalizes a 2025 post about research agents. It is fine as a default but should not be a rule.

## Consistency

- **Doc 07** (line 132: "resume by `session_id`") conflicts with W2-02's fresh-seeded-session preference. BUILD_PLAN's Phase 1 exit ("resumes from its last commit") already matches W2-02. Note that doc 07 is superseded on this point.
- **BUILD_PLAN line 26** (Opus 5.5 plans and verifies) interacts with the classifier fallback. A security-related dependency update (CVE context) can be silently re-run on Opus 4.8. That changes the verifier tier, the cost and the cache. W2-02 raises the route class but not the verifier-independence angle.
- **Edits 2 and 3 conflict on permission mode.** Edit 2 uses `--permission-mode dontAsk`; edit 3 specifies non-root for `bypassPermissions`. Pick `dontAsk` plus allow and deny rules as the default. The non-root note still applies if bypass is ever used (the root check is skipped inside a "recognized sandbox", which Podman may or may not count as).
- No conflicts found with doc 04. The doc's corrections to doc 02 are all confirmed.

## Missing

1. **Credits-only org failure mode.** A `claude -p` call, or possibly an SDK call, against an org holding only subscriber credits gets "Credit balance too low". Phase 1 needs a test: one SDK call and one `claude -p --bare` call against the linked org, with purchased credits present and absent.
2. **SDK callback hooks fail closed** on timeout (hooks.md line 1592). That is a real alternative to command hooks for policy gates if the runner uses the Agent SDK. The doc notes it but does not recommend it.
3. **Claude Code version churn.** Many behaviors are gated on v2.1.2xx (bare MCP behavior at v2.1.286, the auto-mode classifier at v2.1.281, and others), and upgrading invalidates the cache. The plan should pin the Claude Code version in the runner image and treat Claude Code upgrades as an `update` goal of the factory itself, checked against a captured stream-json fixture.
4. **429 classification.** There are two different 429s: a spend cap (no `retry-after`, a stop) and a Claude Code workspace limit (with `retry-after`, a retry). There is also the 400 from a self-set limit. Edit 7 covers only the first.
5. **Haiku 5.5 above 100K tokens costs 5x.** Cheap subagents with large contexts can silently cross that line. Add a guard or a metric.
6. The 2026-10-07 policy change is one day old. Re-check the credits and support pages at the start of Phase 1.

## Recommended edits to BUILD_PLAN.md

| W2-02 edit | Action | Reason |
|---|---|---|
| 1 Re-issue caps on every invocation; ledger is authoritative; pin full model IDs | ACCEPT | Verbatim support (cli-reference line 105). |
| 2 Phase 1 acceptance (stream-json init gate, fresh-session resume rule, SIGINT then SIGTERM, `RESUME_INTERRUPTED_TURN`) | ACCEPT | All verified. Add a pinned Claude Code version and a captured stream-json fixture. |
| 3 Sandbox: non-root, always `--bare` with explicit config, API key via env | MODIFY | Make `dontAsk` plus explicit allow and deny rules the default mode. Keep non-root regardless. |
| 4 CI-green loop in the supervisor; Stop hook as backstop | ACCEPT | Verified cap; sound design either way. |
| 5 Safety in deny rules and exit-2; startup self-test | ACCEPT | Add: prefer SDK callback hooks (fail closed) for policy gates if the SDK is used; the self-test also catches mistyped hook paths. |
| 6 Auth "Decision" | MODIFY | Keep it **OPEN** with "Recommendation: API key". Add the Legal line 57 quote. Say credits may not cover `claude -p` *or* Claude-Code-backed SDK traffic ("Credit balance too low" in credits-only orgs). Require purchased credits, or a Phase 1 test, before relying on them. |
| 7 Rate-limit/tier task | MODIFY | Keep "record actual tier" and the TPM-based concurrency cap. Give the numbers as dated examples. Classify all three limit responses (spend-cap 429, workspace 429 with retry-after, self-limit 400). |
| 8 Cache TTL env | MODIFY | Make 1h TTL conditional on attempts that wait more than 5 minutes; measure the hit rate before making it the default. |
| 9 OTel | ACCEPT | Verified. Drop the [U] on 9464. |
| 10 Models table / retirement floors / classifier fallback | MODIFY | There is no models table, so apply it to the line 26 model roles. Add the retirement floors as calendar items, and a fallback route class that also flags the verifier tier change. |
| 11 Mark second-pass item 2 resolved; Managed Agents facts | ACCEPT | Leave credit coverage of `-p` and the runtime line as open sub-items. |
| 12 SessionStore only for cross-host; no `claude --bg` runner | ACCEPT | `--bg` rejection of `-p` and the reboot behavior are verified. |
| 13 claude-code-action not on the core path; ephemeral self-hosted runners | ACCEPT | Source evidence verified; docs are silent. |
| 14 Eval/judge design rules from the posts | ACCEPT with softening | State "no multi-agent for coding" as a default, not a rule. |
| 15 Outcomes grader is still Claude | ACCEPT | Correct, and relevant to OPEN #6. |
