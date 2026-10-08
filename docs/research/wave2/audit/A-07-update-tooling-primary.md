# A-07 - Audit of W2-07 (update tooling and infrastructure, primary sources)

Auditor: independent model, 2026-10-07. Method: fresh shallow/sparse clones (HEAD 2026-10-06/07) of `renovatebot/renovate`, `github/docs`, `ossf/osv-schema`, `google/osv.dev`, `binwiederhier/ntfy`, `pypi/warehouse`, `golang/website`, `rust-lang/cargo`, `astral-sh/uv`, `pypa/pip`, `npm/cli`, `pnpm/pnpm.io`, `cloudflare/cloudflare-docs`, `containers/podman`; the GitHub REST OpenAPI JSON; live calls to PyPI Simple JSON, crates.io sparse index, proxy.golang.org, index.golang.org, registry.npmjs.org and static.rust-lang.org channel manifests. I ran `renovate-config-validator` (renovate 44.145.1) on the doc's Renovate example, plus a negative control. Not re-checked: Forgejo (host blocked), gVisor, Firecracker, Homebrew API, Go proposal #76485, the Dependabot docs-commit date.

## Verdict

**TRUST WITH CAVEATS.** I re-checked 15 decision-relevant claims. Twelve are supported nearly word for word by the primary text. The rest are either not proven by the cited source or wrong on timing. The researcher's sourcing is real: line numbers and quotes match the clones. The tagging is mostly honest. There are three material caveats:

1. **Cargo native cooldown is not in stable Rust today.** The options are documented as "Respected as of 1.100+". Live stable is rustc 1.99.0 (2026-10-01), and cargo 0.101 (Rust 1.100) is still in beta. The doc's "native cooldown now exists in every major tool ... Cargo" is wrong until about mid-November 2026.
2. **The Renovate minimal example probably stalls lock-file maintenance.** It sets a top-level `minimumReleaseAge` together with `:maintainLockFilesWeekly` and `internalChecksFilter: strict`, but has no `lockFileMaintenance` null rule. Renovate's own security preset exists precisely because such updates "will never come". The validator accepts the config, so this is a semantic trap, not a syntax error.
3. **Correction 10 (Go `.info` time) overstates the case and misses the obvious fix.** `index.golang.org` publishes a proxy first-seen `Timestamp` (verified live).

## Claims table

| # | Claim | Rating | Evidence |
|---|---|---|---|
| 1 | Renovate `minimumReleaseAge` string/default null; `minimumReleaseAgeBehaviour` default `timestamp-required` (opt-in 41.150.0, default in 42); `minimumReleaseAgeBuffer` 30 min; per-version wait | SUPPORTED | `lib/config/options/index.ts` L2198-2217; key-concepts L105-108, timeline L275-289 |
| 2 | Lock-file maintenance, replacement and pin are unsupported for age checks; the `security:minimumReleaseAge*` presets null them | SUPPORTED, incomplete | `security.preset.ts` `unsupportedUpdateTypeRules` also nulls `bump`, `lockfileUpdate` and `rollback`. Preset rationale: applying the age check "results in updates that will never come" |
| 3 | npm `--before` passthrough (stricter of the two dates, ETARGET retry); Poetry `POETRY_SOLVER_MIN_RELEASE_AGE` (ceil days); "specify in **both**" | SUPPORTED | key-concepts L66-100 verbatim |
| 4 | Security updates bypass `minimumReleaseAge`; `best-practices` extends `security:minimumReleaseAgeNpm` + `:maintainLockFilesWeekly`; `allowedVersions` cannot coexist with `matchUpdateTypes` | SUPPORTED | key-concepts L269; `config.preset.ts` L4-16; configuration-options L3270. Validator rejects the combination: "packageRules cannot combine both matchUpdateTypes and allowedVersions" |
| 5 | The doc's Renovate example is a valid working config | SYNTAX SUPPORTED / SEMANTICS LIKELY WRONG | `renovate-config-validator --strict` passes. But `update/branch/index.ts` L437-489 marks any upgrade with `minimumReleaseAge` and no `releaseTimestamp` as pending under `timestamp-required`, and lock-file maintenance has no timestamp. I could not show it end to end: the `--platform=local` dry run does not reach the branch stage |
| 6 | Dependabot default cooldown is 3 days for version updates only, not security; `cooldown` is version-updates only; semver keys only for some ecosystems; `exclude` wins | SUPPORTED | `dependabot-options-reference.md` L185-275; `data/features/dependabot-cooldown-default-days.yml` (fpt/ghec `*`, ghes `>3.21`); "even when `cooldown` is not configured" |
| 7 | Dependabot `ignore` also applies to security updates | SUPPORTED | heading L388 carries both the version and security icons |
| 8 | Docs commit for the default is 2026-07-15 (PR #62269); the 2026-07-14 enablement date is [U] | PLAUSIBLE-UNCHECKED | shallow clone; history not deepened |
| 9 | PyPI 14-day rule rejects new files on releases older than 14 days; it is not a cooldown; merged 2026-07-08, posted 2026-07-22; "should not yet rely" | SUPPORTED | warehouse blog post L2-49. The doc's gloss that "per-file `upload-time` is trustworthy for old releases" leans on behaviour the post says not to rely on |
| 10 | Go `.info` = `{Version, Time}`, and `Time` is commit time, not proxy first-seen time | SUPPORTED | mod.md L165 "In Git, this is the commit time", L2947; live `v0.9.1.info` Time 2020-01-14 |
| 11 | OSV: withdrawn records are excluded from POST queries and still returned by GET `/v1/vulns/<ID>` and the exports; no rate limit; querybatch returns id and modified only; version plus versioned purl gives 400 | SUPPORTED | osv.dev `docs/faq.md` L169-193; `post-v1-querybatch.md` L9, L31-32 |
| 12 | Homebrew became an OSV ecosystem in schema 1.9.0; latest 1.9.1 (2026-09-24); 52 ecosystems | SUPPORTED | osv-schema CHANGELOG L48-49 (1.9.0 = 2026-08-06); `ecosystems.json` has 52 keys including `Homebrew`; `validation/schema.json` L422 |
| 13 | Cooldown option names: uv `exclude-newer` (+`-package`, durations, no months), pip `--uploaded-prior-to` (26.0) with `P3D` (26.1), npm `min-release-age`/`-exclude` (11.10.0, 2026-02-11, `before` wins, audit fix exits non-zero), pnpm `minimumReleaseAge` in minutes (1440 default since v11, non-strict unless set), Cargo `registry.global-min-publish-age` / `resolver.incompatible-publish-age` | SUPPORTED (names/semantics) | uv resolution.md L815-930; pip NEWS L176-188, L261; npm definitions.js L1541-1594, CHANGELOG 11.10.0; pnpm dependency-resolution.md L303-398; cargo config.md L1213-1226, L1364-1381 |
| 14 | "Native cooldown now exists in every major tool ... Cargo (Rust 1.100)" | CONTRADICTED (timing) | cargo docs say "MSRV: Respected as of 1.100+". Live `channel-rust-stable.toml`: rustc 1.99.0, date 2026-10-01; beta cargo is 0.101.0 (= Rust 1.100). Not on stable until about 2026-11-12 |
| 15 | GitHub JIT: `POST .../actions/runners/generate-jitconfig` (repo and org); required `name`, `runner_group_id`, `labels` (1-100); optional `work_folder`; 201 `{runner, encoded_jit_config}`; 404/409/422; classic token needs `repo` | SUPPORTED | OpenAPI. The doc omits that the caller must have repo **admin** access |
| 16 | Webhook redelivery: `GET .../hooks/{id}/deliveries` (`per_page`, `cursor`, `status`) and `POST .../deliveries/{delivery_id}/attempts`; no auto-redelivery; 3-day window (7 on GHES); same `X-GitHub-Delivery` on redelivery; 10 s response limit | SUPPORTED | OpenAPI paths and params; `handling-failed-webhook-deliveries.md` L4/L18; `data/variables/webhooks.yml` retention `3`/`7`; best-practices L41-60 |
| 17 | ntfy: up to 3 actions; `http` short format with `headers.<H>=`, `body=`, `clear=true`; default POST; `X-Actions`/`Actions`/`Action`; quoting rules; supported on Android, iOS and web | SUPPORTED | ntfy `docs/publish.md` L1185-1211, L2047-2054, L2333-2335 |
| 18 | ntfy `http` action is executed by the phone or browser, not the server | PLAUSIBLE-UNCHECKED (tagged [V]) | publish.md says only "sends a HTTP request when the action button is tapped". It is true by design, but not stated in the cited text |
| 19 | Cloudflare Access actions Allow/Block/Bypass/Service Auth; Bypass disables enforcement and logging and has no identity selectors; documented for "webhook receivers" | SUPPORTED | `policies/index.mdx` L20, L55-77; `common-policies.mdx` L1084 |
| 20 | Rootless Podman: only delegated controllers can be limited; check `cgroup.controllers`; `Delegate=memory pids cpu cpuset` drop-in; pids-limit default 2048 | SUPPORTED | `troubleshooting.md` L712-737; `options/pids-limit.md` L11 |
| 21 | ARC requires a Kubernetes cluster and Helm 3 | SUPPORTED | `use-actions-runner-controller/get-started.md` L22-27 |
| 22 | Live registry shapes: PyPI Simple JSON api-version 1.4, `upload-time`, ETag, max-age=600; crates `pubtime`; npm `deprecated` | SUPPORTED | live curl 2026-10-07. `pubtime` is backfilled: all 316 serde versions have it |
| 23 | Forgejo ephemeral runners need v15 plus a runner newer than 12.5 | PLAUSIBLE-UNCHECKED (honestly tagged V-secondary) | host blocked for me too |

## Tag integrity

- Overall honest. [U] is used where the source was unreachable (Dependabot date, Atom feeds, Yarn/Bun, Homebrew fields, Go gating proxies). Forgejo is correctly marked "V-secondary", not [V].
- **Over-tagged:** the ntfy claim "executed by the phone/browser" is an inference, tagged [V]. Cargo "stabilised Rust 1.100" is tagged [V] without saying 1.100 is unreleased. Go #76485 "[V: WebFetch of the issue page]" conflicts with the doc's own fetch-failure list (`api.github.com` blocked; github.com unclear); I could not reproduce it.
- **Under-stated:** the "Remaining unverified" list says the Renovate "44.x" preset claim is unverifiable. The `unsupportedUpdateTypeRules` block exists at HEAD, and Renovate is now at 44.145.1, so the claim is substantially true. Only the exact version that introduced it is unverified.

## Overreach findings (the 11 corrections to doc 05)

| # | Correction | Assessment |
|---|---|---|
| 1 | Header note obsolete | Accept |
| 2 | Lock-file gap narrower (npm `--before`, Poetry env) | Accept. Add: the presets also null `bump`, `lockfileUpdate` and `rollback`. Lock-file maintenance for npm **does** run with `--before` (key-concepts L93). For uv, pip, cargo and go nothing is passed |
| 3 | Downgrade CVE-2026-88884 row to [U] | Accept. Renovate 44.3.1 exists (npm, 2026-07-30), but nothing in the repo ties a CVE to it |
| 4 | best-practices preset is 3-day npm | Accept |
| 5 | Dependabot 3-day default [V], date [U] | Accept |
| 6 | pip durations need 26.1 | Accept (NEWS L176-188) |
| 7 | npm deprecate / crates yank semantics | Accept |
| 8 | PyPI 14-day meaning | Accept the definition. Reject the add-on claim that `upload-time` is now "trustworthy": the post says not to rely on it, and files can still be added within the 14 days |
| 9 | OSV withdrawn only visible via GET/export | Accept. Design consequence: periodic GET re-fetch of cached IDs |
| 10 | Go `.info` time "cannot serve" for cooldown | **Modify.** Commit time makes a version look older than it is (cooldown too weak), so "weaker" is the right word, not "cannot". The fix is to use `index.golang.org/index?since=` `Timestamp` (proxy first-seen, verified live) or the factory's own first-seen store. Also note that Renovate's `go` datasource is listed as having timestamps (key-concepts L349), so Renovate's Go cooldown inherits the commit-time weakness |
| 11 | crates `pubtime` removes the need for an extra call | Accept, but doc 05 never claimed otherwise, so this is an addition, not a correction |

Other overreach in the document:

- "Native cooldown now exists in every major tool": false for Cargo on stable today (see claim 14).
- The proposed `native_defence: {cargo: global-min-publish-age}` will silently do nothing on Rust 1.99 or older. The doc flags only "confirm installed toolchain".
- Minimal Renovate example: see claim 5. Add `{matchUpdateTypes: ["lockFileMaintenance","pin","replacement","bump","lockfileUpdate","rollback"], minimumReleaseAge: null}`, or extend `security:minimumReleaseAgePypi`/`Npm` instead of a bare top-level value.
- Dependabot example: `cooldown.exclude: ["claude-agent-sdk"]` means 0 days, while the Renovate example and doc 05 policy use 1 day. Dependabot has no per-package days without a separate `updates` block using `include`. That is a unit-compile gap the policy compiler must handle.

## Consistency with BUILD_PLAN.md and docs 05/07

- Consistent with BUILD_PLAN s1 Detection: Renovate for mechanics, cooldown in the factory plus natively. W2-07 strengthens the "factory resolver owns cooldown" rule, because transitive and lock paths for uv, pip, cargo and go get no Renovate passthrough.
- Consistent with doc 07 on JIT without Kubernetes, no auto-redelivery plus a reconciler, the 10 s limit, and ntfy `http` buttons needing device reachability (BUILD_PLAN Open Q4).
- The W2-07 ask to "mark Verification items 3 and 4 done" is too broad. Item 4 (Forgejo ephemeral) remains secondary-only.
- BUILD_PLAN Open Q1 (forge) is unaffected. The Forgejo >= v15 requirement is a new constraint the owner should see, but it is unverified.

## Missing risks and alternatives

1. **Go first-seen timestamps from `index.golang.org`.** It is the free, primary alternative to commit time, and the doc does not mention it.
2. **Cargo cooldown timing.** Until Rust 1.100 ships (about 2026-11-12) and the host toolchain is upgraded, crates need factory-side gating (sparse index `pubtime`).
3. **ntfy web client and CORS.** If an answer is tapped in the ntfy web app, the browser makes the `http` call, so the answer endpoint must send CORS headers or the web button fails. Unchecked; test it in Phase 4.
4. **Bypass has no logging.** Bypass on `/hooks/github` turns off Access logs. Log at the receiver, and consider a Cloudflare WAF rule allowing only GitHub `/meta` hook CIDRs. Alternatives not mentioned: Tailscale Funnel, or no inbound at all (poll only), which doc 07 already prefers.
5. **The JIT endpoint needs repo admin rights,** which matters for token scoping. A fine-grained PAT or GitHub App with `administration: write` is the least-privilege form (unchecked).
6. **Within-window wheel uploads.** A version can gain new files during its first 14 days. uv's per-file `upload-time` gate then hides the new wheel while older files pass, which can change the resolved artifact (sdist vs wheel) between runs. Pin artifact hashes in the lock.
7. **Renovate requires Node `^24.11.0` and pnpm `^11`** (npm engines for 44.145.1). The self-hosted image avoids this; an npm install on the host needs Node 24. I confirmed that Node 22 crashes with `RegExp.escape is not a function`.

## Recommended edits

### docs/research/05-dependency-updates-and-constraints.md

| W2-07 proposal | Decision | Reason |
|---|---|---|
| Verified cooldown table (tool, option, unit, default) | **Modify** | Accept, but mark the Cargo row "Rust >= 1.100 (beta on 2026-10-07; stable about 2026-11-12)". Add a pnpm `minimumReleaseAgeStrict` column note |
| Security updates bypass cooldown in Renovate and Dependabot | Accept | Verified |
| Go caveat: use the factory's own first-seen time | **Modify** | Prefer `index.golang.org` `Timestamp`, with the factory's first-seen store as fallback. Say "weaker", not "cannot" |
| OSV withdrawn fix; Homebrew ecosystem | Accept | Verified (schema 1.9.0, 2026-08-06) |
| PyPI Simple JSON preferred; 14-day note; crates `pubtime` | Accept | Drop "upload-time trustworthy for old releases" |
| Constraint projection syntaxes; Dependabot `ignore` can suppress security fixes; independent advisory scan | Accept | Verified; an important catch |
| Policy `native_defence` incl. cargo; require pnpm strict | **Modify** | Cargo entry conditional on toolchain >= 1.100. Add a Renovate rule nulling `minimumReleaseAge` for the unsupported update types |
| Downgrade CVE-2026-88884 row to [U] | Accept | No primary support |
| Retag verified items [V] | Accept | |

### docs/BUILD_PLAN.md

| W2-07 proposal | Decision | Reason |
|---|---|---|
| L29: generated-config contract (constraints into `allowedVersions`/`ignore` plus native cooldowns); self-hosted Renovate image | Accept | Consistent with s1. Add "generated Renovate config must null `minimumReleaseAge` for lockFileMaintenance and the other unsupported types" |
| Phase 1/2: JIT via `generate-jitconfig` and a small launcher; ARC only with k3s/kind; Forgejo >= v15 | Accept / modify | JIT and ARC verified. Mark Forgejo v15 as unverified (secondary) and keep Open Q1 open |
| Open Q3 host checks (`/dev/kvm`, cgroup delegation drop-in) | Accept | Podman part verified. Add "Rust toolchain >= 1.100 if cargo native cooldown is relied on" |
| Open Q4 / Phase 4: ntfy `http` actions with bearer header, max 3, always a `view` link; Bypass on the webhook path plus HMAC, 10 s, dedupe on delivery GUID, 3-day redelivery reconciler | Accept | Verified. Add "receiver-side logging (Bypass is unlogged)" and "test the web-client CORS path" |
| Poll budget (ETag, 5,000/h, 900 points/min) | Accept (not re-checked by me) | Matches doc 07 intent |
| Mark Verification items 3 and 4 done | **Modify** | Item 3 done except the Dependabot rollout date. Item 4 partially done: ntfy and Cloudflare done, Forgejo still open |
