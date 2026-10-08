# Worked example: keeping a multi-harness messaging tool compatible with its upstream CLIs

This is an example of goal type `update`, not part of the factory's core design. The factory itself is use-case agnostic (see `../DESIGN.md` and `../BUILD_PLAN.md`). The example is useful because it exercises the contract-testing, constraints-registry and escalation paths against upstreams that change often and document their interfaces poorly.

Scenario: the owner's messaging tool talks to several coding-agent CLIs (Claude Code, Codex, OpenCode, Kimi, and possibly others). Each upstream release might change flags, output events, exit codes or config in a way that breaks the tool. The factory watches the releases, runs the tool's conformance suite against each new version, and decides: adapt the tool, pin/exclude the version, or escalate.

Evidence base: `../research/wave2/W2-01-harness-cli-contracts.md` (source and registry reading; no binary was run) and its audit `../research/wave2/audit/A-01-harness-cli-contracts.md`. Runtime behaviour is unverified.

## What the research found (source-level, audited)

- **Event schemas differ a lot.** Claude Code and Codex have typed event schemas. OpenCode's `run --format json` is an ad-hoc envelope `{type,timestamp,sessionID,part}` around internal parts. Kimi emits OpenAI-style `{role,content,tool_calls}` lines. So each upstream needs its own event grammar; "terminal" means a terminal event or process exit.
- **Kimi has two incompatible generations.** `kimi-cli` (PyPI) was archived with its last release, 1.52.0, on 2026-09-22; the successor `kimi-code` (npm `@moonshot-ai/kimi-code`, 2.1.1) has different flags (`-p` takes the prompt) and a different data root. Use two adapters, with the legacy checks capped at 1.51.0.
- **Hazard.** In `kimi-cli` 1.52.0 every entry point (`-p`, `--print`, `acp`, `--help`) prints a deprecation notice and exits 0, so tests record false successes, and a bare run executes a shell string fetched from a CDN with no confirmation. The version is banned in the constraints registry, harnesses are invoked by absolute path with explicit arguments, and a liveness check fails any run that exits 0 without protocol output.
- **Past breakage (counted from changelogs and git history; small numbers):** Claude Code had 2 breaking headless-output changes, all before 2.0, and later output changes are additive, though default behaviour still shifts. Codex had 4 wire-level schema renames in 2 commits in its first 8 days of JSON events, then additive changes, and removed `--full-auto` 93 days after deprecation (one data point), so the contract covers invocation flags, not only event fields. OpenCode's JSON envelope has been stable since 2025-09-24. Kimi changed print-mode exit codes in 1.27.0, then rewrote the CLI.
- **Release cadence.** In the 90 days before research, Claude Code had about 78 releases, Codex 39 and OpenCode 41 (counts depend on the window start), so a live smoke test within 24 hours of each new stable release is the practical cadence.
- **Self-updaters.** `kimi-code` auto-installs updates by default and Claude Code has `DISABLE_UPDATES`; the test environment sets updaters off (`DISABLE_UPDATES=1`, `KIMI_CODE_NO_AUTO_UPDATE=1`), an explicit permission mode and per-run config/home directories.
- **Standards in flux.** ACP runs natively in OpenCode and Kimi and through third-party adapters for Claude Code and Codex, and is mid-migration (schema v1.24.1, v2 draft). MCP's 2026-07-28 revision removed sessions and `initialize`. A2A reached v1.0.1.
- **Not yet covered:** Gemini CLI, Cursor CLI and Copilot CLI (all speak ACP). They need a short research pass before adapters are written.

## How the generic factory pieces apply

- Each upstream is an entry in the constraints registry with `kind: cli`, `tested_range` and `known_bad`. Pins are expressed both as dev-time pins and as runtime compatibility ranges, because the user's machine, not the factory, controls the installed CLI version.
- The canary installs the new version in a sandbox with an API key (not a throwaway account), runs recorded-fixture contract tests with fake-CLI shims, then a live smoke test. Claude `stable` and `latest` channels are tracked separately.
- A new enum value in an event is classified `ambiguous` or `breaking`.
- A failing contract triggers the decision procedure: adapt the adapter to accept both old and new fields when the change is clear and mechanical; constrain or exclude the version when it is an upstream regression; escalate when the change forces a protocol or design decision (for example, a config schema change that affects every adapter).

## Decisions specific to this example

- **Known-bad versions in the messaging tool:** bundled in each release, generated from the factory's constraints registry. No runtime fetch for now (the factory may itself improve the messaging tool, so the delivery path stays simple). Default is warn; safety-class entries (such as `kimi-cli==1.52.0`) block. The preflight reads the installed version from package metadata or the install path, not by running the harness.
- **Messaging tool as an escalation channel:** later, as an adapter (conditional on it exposing MCP or ACP); ntfy, the web answer page and PR comments come first.
- **OPEN: harness scope.** Which CLIs the first conformance suite covers. This is an example-level choice and does not block the factory's phases; any harness that needs adapters must first be researched like the four above.
