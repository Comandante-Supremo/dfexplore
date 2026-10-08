# W2-07 - Update tooling and infrastructure: primary-source verification

Research date: 2026-10-07. Tags: [V] verified by reading the primary source named (a docs-site source repo, spec, source code, or a live registry response); [U] not verified. Scope: every [U] in doc 05 plus the infrastructure items listed for doc 07 (doc 07 contains no literal [U] tags; I verified the items BUILD_PLAN lists as open).

## Method and fetch failures

The sandbox egress proxy allow-lists hosts. WebFetch failed with DNS errors on `docs.github.com` and `docs.renovatebot.com`. curl got 403 CONNECT-rejected on `peps.python.org`, `osv.dev`, `api.osv.dev`, `formulae.brew.sh`, `ntfy.sh`, `docs.ntfy.sh`, `developers.cloudflare.com`, `gvisor.dev`, `forgejo.org`, `codeberg.org`, `gitea.com`, `github.blog`, `blog.pypi.org`, `github.com/*/releases.atom` and `api.github.com/repos/*` (the latter says to `add_repo`). Not blocked: `git clone` of public GitHub repos, `raw.githubusercontent.com`, and the PyPI, npm, crates.io and Go proxy registries (live calls). So I read the docs sites through their **source repositories**, shallow and sparse-cloned, at HEAD dated 2026-10-07 or a few days earlier. That is the primary text, pre-render:

| Source clone | Used for |
|---|---|
| `renovatebot/renovate` | `lib/config/options/index.ts`, `docs/usage/*`, presets |
| `github/docs` | Dependabot options reference, Actions/webhook/REST docs; `data/features/*.yml` |
| `python/peps` | PEP 592, 691, 700 |
| `pypi/warehouse` | PyPI blog source (14-day rule) |
| `astral-sh/uv`, `pypa/pip`, `npm/cli`, `pnpm/pnpm.io`, `rust-lang/cargo`, `golang/website` | cooldown options and Go module reference |
| `google/osv.dev`, `ossf/osv-schema` | OSV API and schema |
| `Homebrew/brew`, `Homebrew/formulae.brew.sh` | livecheck, JSON API |
| `binwiederhier/ntfy` | `docs/publish.md` |
| `cloudflare/cloudflare-docs` | Access policies, service tokens |
| `go-gitea/gitea` | runner registration code |
| `containers/podman`, `google/gvisor`, `firecracker-microvm/firecracker` | sandbox prerequisites |

`github/rest-api-description` (OpenAPI JSON) gave the JIT-runner and webhook-delivery endpoints. Forgejo ephemeral support is from WebSearch result summaries only (forgejo.org was unreachable).

## Summary

- Renovate `minimumReleaseAge` semantics in doc 05 are right but incomplete. It has more knobs (`minimumReleaseAgeBehaviour`, `minimumReleaseAgeBuffer`), a Renovate-42 behaviour change (missing timestamp now blocks), and for npm and Poetry Renovate passes native cooldown flags during lockfile generation. It is not a blanket "bypass".
- Dependabot: the 3-day default cooldown for version updates is [V] in GitHub's own docs source (not security updates). The exact date 2026-07-14 is not verified; the docs commit is 2026-07-15.
- The PyPI "14-day" item is [V] and it is **not a cooldown**. It rejects new files on releases older than 14 days (merged 2026-07-08, blog 2026-07-22). Its effect on us is minor but real: a version's age no longer changes after 14 days, so per-file `upload-time` is trustworthy for old releases.
- Native cooldown now exists in every major tool the factory touches: uv, pip (>=26.0/26.1), npm >=11.10, pnpm (default 1440 min since v11), Cargo (Rust 1.100, `pubtime`), Renovate, Dependabot. Go has only an open, on-hold proposal. Units differ (days / minutes / duration strings), confirming doc 05's "unit chaos".
- OSV: `withdrawn` records are excluded from POST query responses, so the factory only sees withdrawal if it re-fetches cached IDs via GET. `MAL-` is the malicious-packages prefix. Homebrew is an OSV ecosystem as of schema 1.9.0. No API rate limit is documented.
- Infra: JIT runner API, ARC (requires Kubernetes), ntfy actions (phone-side HTTP), Cloudflare Access Bypass for webhook paths, webhook redelivery API (3-day window), rootless Podman cgroup delegation, and gVisor/Firecracker prerequisites are all [V]. Forgejo v15 ephemeral runners are [V-secondary].

## Verified findings

### Renovate

- **`minimumReleaseAge`**: type string, default `null`. Renovate 42.19.5+ treats `"0 days"` as null. Waits per version; it does not wait for a quiet period. [V: renovate `lib/config/options/index.ts` L2198-2217; `docs/usage/configuration-options.md` "minimumReleaseAge"]
- **`minimumReleaseAgeBehaviour`**: `timestamp-required` (default; added 41.150.0, default from 42) or `timestamp-optional`. Before 42 a missing timestamp counted as passed. **`minimumReleaseAgeBuffer`** default `30 minutes`. **`internalChecksFilter`**: `strict` (default) / `flexible` / `none`. Status check name `renovate/stability-days`. [V: same files, `docs/usage/key-concepts/minimum-release-age.md`]
- The release timestamp must come from the registry, not the publisher (Docker `org.opencontainers.image.created` rejected for that reason). Timestamp support per registry: crate, Docker Hub, github-releases yes; Docker on GHCR/Quay/non-Hub no. [V: key-concepts page]
- **Update types**: major/minor/patch supported; `lockFileMaintenance`, `lockfileUpdate`, `rollback`, `bump`, `pin`, `replacement` NOT supported; `digest`/`pinDigest` conditional. [V: key-concepts table]. The `security:minimumReleaseAge{Crate,Npm,Pypi}` presets (3 days each) add a rule that sets `minimumReleaseAge: null` for `lockFileMaintenance`, `replacement`, `pin`. [V: `lib/config/presets/internal/security.preset.ts`]
- **Native passthrough**: for npm, Renovate passes `--before=<now - minimumReleaseAge>` during lock generation (stricter of that and `.npmrc` `before`/`min-release-age`; retries without `--before` on `ETARGET`). For Poetry it sets `POETRY_SOLVER_MIN_RELEASE_AGE` (integer days, ceiling). Docs: "We recommend specifying minimum release age in both your Renovate and package manager configuration". Transitive deps are not managed by Renovate. [V: key-concepts L66-100, L291-293]
- **Security updates bypass `minimumReleaseAge`** [V: key-concepts "What happens to security updates?"]. The `best-practices` preset extends `security:minimumReleaseAgeNpm` and `:maintainLockFilesWeekly`; upgrade-best-practices recommends `"14 days"` when automerging third-party deps. [V: `config.preset.ts`, `upgrade-best-practices.md` L132]
- **`lockFileMaintenance`** defaults: `enabled: false`, `schedule: ['before 4am on monday']`, `branchTopic: 'lock-file-maintenance'`. [V: options/index.ts L2640-2659]
- **`packageRules.allowedVersions`**: a semver range or `/regex/`; supports templates such as `<={{add major 1}}`; "`allowedVersions` and `matchUpdateTypes` cannot be used in the same package rule". [V: configuration-options.md L3230-3284]. Matchers present: `matchPackageNames`, `matchDepNames`, `matchManagers`, `matchUpdateTypes`, `matchCurrentVersion`, `matchDatasources`. [V]
- **Dependency Dashboard**: `dependencyDashboard` default `false` in the options table, but enabled by `config:recommended` (docs: "Starting from version v26.0.0"). Dashboard lists pending/open/closed/error PRs and lets a human force a pending update; pending-status-check updates appear there. [V]
- **Self-hosting**: npm package `renovate` or images `renovate/renovate` and `ghcr.io/renovatebot/renovate` (amd64+arm64); platforms include `github`, `gitea`, `forgejo`; `RENOVATE_TOKEN`, `autodiscover`, `onboarding` options exist; config schema URL `https://docs.renovatebot.com/renovate-schema.json` (referenced from docs; the schema file itself not fetched). [V: running.md, config-validation.md]

Minimal working example for the factory's native-defence layer:

```json5
{
  $schema: "https://docs.renovatebot.com/renovate-schema.json",
  extends: ["config:recommended", ":maintainLockFilesWeekly"],
  internalChecksFilter: "strict",
  minimumReleaseAge: "3 days",
  packageRules: [
    { matchUpdateTypes: ["major"], minimumReleaseAge: "14 days" },
    // factory-generated hold (from constraints.yaml); one rule per constraint
    { matchPackageNames: ["fastapi"], allowedVersions: "<0.115.0" },   // not combinable with matchUpdateTypes
    { matchDatasources: ["pypi"], matchPackageNames: ["claude-agent-sdk"], minimumReleaseAge: "1 day" }
  ]
}
```

### Dependabot

- **`cooldown`** (version updates only; "not available for security updates"): `default-days`, `semver-major-days`, `semver-minor-days`, `semver-patch-days`, `include` (max 150, wildcards), `exclude` (max 150; wins over include). If a semver-level key is unset, `default-days` applies. Semver keys are supported only for some ecosystems (npm/yarn, pip, uv, cargo, gomod, bundler, maven, etc.); Docker, Actions, Terraform, Helm and others only get `default-days`. [V: `github/docs` `content/code-security/reference/supply-chain-security/dependabot-options-reference.md` L183-291]
- **Default**: "By default, Dependabot applies a cooldown period of 3 days to version updates ... This default cooldown does not apply to security updates." Gated by feature file `dependabot-cooldown-default-days` (fpt `*`, ghec `*`, ghes `>3.21`); without it "Consider all new versions immediately". [V: options reference L191-196; `data/features/dependabot-cooldown-default-days.yml`; `data/reusables/dependabot/default-cooldown-period.md`]. The commit that documents it is dated **2026-07-15** ("Document Dependabot's default 3-day version-update cooldown", PR #62269). The claimed 2026-07-14 enablement date is [U] (the changelog on github.blog was unreachable).
- **`groups`**: first match wins; keys `applies-to` (`version-updates`|`security-updates`), `dependency-type`, `patterns`, `exclude-patterns`, `update-types`, `group-by: dependency-name`. **`ignore`**: `dependency-name`, `versions`, `update-types` (`version-update:semver-major|minor|patch`); documented for both version and security updates (the heading carries both icons) and "a dependency that is matched by both an allow and an ignore will be ignored". **`multi-ecosystem-groups`** exists (version updates). [V: same file L315-535]
- uv docs recommend pairing `exclude-newer = "1 week"` with `cooldown: {default-days: 7}` for ecosystem `uv`, else Dependabot PRs fail to lock. [V: `astral-sh/uv` `docs/guides/integration/dependabot.md`]

```yaml
version: 2
updates:
  - package-ecosystem: "uv"
    directory: "/"
    schedule: {interval: "daily"}
    cooldown: {default-days: 3, semver-major-days: 14, exclude: ["claude-agent-sdk"]}
    ignore:
      - {dependency-name: "fastapi", versions: [">=0.115.0"]}   # generated from constraints.yaml
```

### PyPI

- **Yanking (PEP 592)**: HTML `data-yanked` attribute (optional string reason); installers MUST ignore yanked files if the constraints can be met otherwise; yanked files remain installable only when exactly pinned (`==`/`===`) or locked; yank is reversible ("API users MUST be able to cope with a yanked file being 'unyanked'"). JSON: PEP 691 file key `yanked` = boolean or non-empty reason string. [V: `python/peps` pep-0592.rst, pep-0691.rst L282-303]. Live: `GET /simple/requests/` with `Accept: application/vnd.pypi.simple.v1+json` returns `meta.api-version` `1.4`, `versions`, per-file `upload-time` (`2026-05-14T19:25:27.735762Z`), `yanked`, `size`, `provenance`, `project-status`; `ETag` and `cache-control: max-age=600`. [V: live curl 2026-10-07]
- **PEP 700 (api-version >= 1.1)**: mandatory `versions` list and `size`; optional `upload-time`. [V: pep-0700.rst L49-93]. Legacy JSON API `/pypi/<pkg>/json` gives `releases[ver][i].yanked`, `yanked_reason`, `upload_time_iso_8601`, plus a `vulnerabilities` key. [V: live]
- **"14-day change"**: PyPI "now rejects new files being uploaded to releases that are older than 14 days" (patch merged 2026-07-08, announced 2026-07-22). It is a supply-chain guard against poisoning old releases, not a cooldown. The post warns: "Users should not yet rely on this behavior as there are no defined semantics ... or APIs available to confirm the state of the release." [V: `pypi/warehouse` `docs/blog/posts/2026-07-22-releases-now-reject-new-files-after-14-days.md`]

### npm

- `npm deprecate pkg@"<0.2.3" "message"`; empty string un-deprecates; deprecation is per version and stored in the packument `versions[v].deprecated` (live: left-pad shows `"use String.prototype.padStart()"`). `time` map is `{created, modified, "<version>": ISO}` (live). [V: `npm/cli` npm-deprecate.md; live registry.npmjs.org]
- **`min-release-age`**: number of days; default null; `min-release-age-exclude` (list of names/globs); implemented by setting `before = now - days` unless `before` is set in the same source ("`before` wins"); blocked `npm audit fix` exits non-zero. Added in npm **11.10.0 (2026-02-11)**, exclude list later. [V: `workspaces/config/lib/definitions/definitions.js` L1541-1594; `CHANGELOG.md` L507-511]
- **pnpm**: `minimumReleaseAge` in **minutes**, default **1440 since v11** (0 before), added v10.16.0, applies to transitive deps; `minimumReleaseAgeExclude` (patterns since 10.17); `minimumReleaseAgeStrict` (default true only when explicitly configured; the built-in default is non-strict and falls back to a too-new version); `minimumReleaseAgeIgnoreMissingTime` default true. [V: `pnpm/pnpm.io` `docs/settings/dependency-resolution.md`]. Implication: relying on pnpm's built-in default does not enforce anything strictly; set the value explicitly.
- Yarn `npmMinimalAgeGate` and Bun 1.3: [U] (secondary only in doc 05; not re-verified).

### crates.io and Cargo

- Sparse index `https://index.crates.io/<2>/<2>/<name>` (e.g. `se/rd/serde`), one JSON object per line, `ETag`/`Last-Modified` supported, `config.json` = `{dl, api}`. Lines carry `yanked` (the only mutable field) and now `pubtime` (`2026-07-18T23:05:13Z`, UTC, no fractional seconds, original publish time unaffected by yank). [V: live curl; cargo `doc/book/src/reference/registry-index.md` L225-240]. `pubtime` stabilised in Rust 1.94.
- Yank semantics: no new dependencies on a yanked version; existing `Cargo.lock` keep working; `cargo yank --version 1.0.1 [--undo]`. [V: cargo publishing.md]
- **Native cooldown** (stabilised **Rust 1.100**): `registry.global-min-publish-age = "7 days"` (units seconds..months, default `"0"`), `registries.<name>.min-publish-age`, `resolver.incompatible-publish-age = "deny"|"allow"` (default `deny`: ignore too-new versions unless already in `Cargo.lock`). Env `CARGO_REGISTRY_GLOBAL_MIN_PUBLISH_AGE`. [V: cargo `config.md` L1213-1226, L1364-1381; `resolver.md` L313-322]

### Go

- GOPROXY protocol: `GET $base/$module/@v/list` (no pseudo-versions), `@v/$version.info` (JSON `{Version, Time}`), `.mod`, `.zip`, optional `@latest`; module and version are case-encoded (`!` + lowercase); 404/410 mean "try next", comma vs pipe in `GOPROXY` controls fallback; other 4xx/5xx are errors. [V: `golang/website` `_content/ref/mod.md` L2881-2960; live `proxy.golang.org/github.com/pkg/errors/@v/list` and `.info`]. Caveat: `Time` is documented as "commit time", so it is a **VCS timestamp, not proxy first-seen time**; a cooldown built on it is weaker than npm/PyPI's registry timestamps. [V]
- `retract [v1.9.0, v1.9.5]` in the module's own go.mod; retracted versions are skipped by `@latest`/upgrades, builds pinned to them continue, `go list -m -u -retracted` reveals them; the go command reads retractions from the highest version's go.mod (`go list -m -retracted $mod@latest`), so retraction is visible only after a newer version is published. [V: mod.md L882-915]
- Cooldown: `golang/go#76485` (opened 2025-11-27, proposes `GOCOOLDOWN=15d`) is open, labelled `Proposal-Hold`, no maintainer comments. [V: WebFetch of the issue page]. Third-party gating proxies exist (secondary, search summary only) [U].

### Homebrew

- JSON API: `https://formulae.brew.sh/api/formula.json`, `/api/formula/<name>.json`, `/api/cask.json`, `/api/cask/<name>.json`, plus `/api/analytics/...`. [V: `formulae.brew.sh` `docs/api.md`; host not reachable live]
- `brew livecheck` (dev-cmd) switches: `--json`, `--newer-only`, `--installed`, `--formula`, `--cask`, `--tap=`, `--eval-all`, `--full-name`, `--autobump`, `-q`; livecheck blocks support strategies (`GithubLatest`, `GithubReleases`, `Git`, page match, etc.), `throttle`, `skip`. [V: `Homebrew/brew` `Library/Homebrew/dev-cmd/livecheck.rb`, `docs/Brew-Livecheck.md`]. Homebrew has no per-version publish timestamp in this API, so no native cooldown [U: I did not find one].

### OSV

- `POST https://api.osv.dev/v1/querybatch` body `{"queries":[{package:{name,ecosystem}|{purl}, version, page_token}|{commit}]}`; returns per-query `vulns: [{id, modified}]` only, same order as input; `version` and versioned purl together => 400; pagination via `next_page_token` (query pagination at >1,000 results or >20 s; response may contain only the token). `GET /v1/vulns/{id}` returns the full record. [V: `google/osv.dev` `docs/api/post-v1-querybatch.md`, `post-v1-query.md` L160-187]
- "Is the API rate limited? No." 32 MiB response cap on HTTP/1.1, none on HTTP/2. [V: `docs/faq.md` L191-198]
- **Withdrawn records are excluded from POST query responses** and list/search, but still returned by `GET /v1/vulns/<ID>` and exports (with the `withdrawn` RFC3339 field). [V: faq.md L169-182; osv-schema `docs/schema.md` L737-748]. So withdrawal detection requires re-fetching IDs the factory has cached.
- Schema latest in clone: **1.9.1 (2026-09-24)**; 1.9.0 added the **Homebrew** and WordPress ecosystems and wildcard package name `*`; ids with prefix `MAL` come from the OpenSSF malicious-packages repo. `ecosystems.json` has 52 ecosystems including PyPI, npm, crates.io, Go, Maven, Homebrew, GitHub Actions, Debian, Alpine. Ecosystem names are case-sensitive ("crates.io", "PyPI"). [V: `ossf/osv-schema` CHANGELOG.md, `ecosystems.json`, `docs/schema.md` L486]

```bash
curl -sS https://api.osv.dev/v1/querybatch -d '{"queries":[
  {"package":{"ecosystem":"PyPI","name":"jinja2"},"version":"2.4.1"},
  {"package":{"purl":"pkg:npm/lodash@4.17.20"}}]}'
```

### uv, pip cooldowns

- **uv**: `exclude-newer` takes an RFC 3339 timestamp, a local date, or a duration (`"7 days"`, `"1 week"`, ISO `P7D`; no months/years). Compared against **per-file** `upload-time` (PEP 700), not release date. Missing `upload-time` => distribution unavailable unless opted out. `exclude-newer-package = { setuptools = "30 days" }`, per-index `[[tool.uv.index]] exclude-newer = "7 days" | false`, `--exclude-newer false` to disable. With `uv.lock`, the computed timestamp is stored and is refreshed only on `--upgrade`/`--refresh`. Applies to registry packages only (not git). [V: `astral-sh/uv` `docs/concepts/resolution.md` L805-930, `docs/concepts/indexes.md` L312-334]
- **pip**: `--uploaded-prior-to` added in **26.0** (ISO date/datetime), `PnD` durations (e.g. `P3D`) from **26.1**; override config with `P0D`; fails hard if the index lacks upload-time metadata; ignores local files, `--find-links`, VCS. Docs warn it delays security fixes and recommend pairing with pip-audit/Dependabot. [V: `pypa/pip` `docs/html/user_guide.rst` L360-445, `cmdoptions.py` L486-533]. Doc 05 said only "26.0 `--uploaded-prior-to`"; add the 26.1 duration form.

```toml
# pyproject.toml
[tool.uv]
exclude-newer = "3 days"
exclude-newer-package = { claude-agent-sdk = "1 day" }
```
```ini
# pip.conf
[install]
uploaded-prior-to = P3D
```
```
# .npmrc: min-release-age=3        pnpm-workspace.yaml: minimumReleaseAge: 4320  (minutes)
# ~/.cargo/config.toml: [registry] global-min-publish-age = "3 days"
```

### GitHub release polling and rate limits

- Most REST endpoints return `etag` (many `last-modified`); a conditional `GET` with `If-None-Match` returning **304 does not count against the primary rate limit when authorized**. Primary limits: **60/h unauthenticated, 5,000/h authenticated**. Secondary: max 100 concurrent requests, 900 points/min per REST endpoint (GET=1, POST/PATCH/PUT/DELETE=5), 80 content-creating requests/min. Exceeding either gives 403/429 with `retry-after`/`x-ratelimit-reset`. `GET /rate_limit` is free. [V: `github/docs` `best-practices-for-using-the-rest-api.md` L76-106, `rate-limits-for-the-rest-api.md`, `data/reusables/rest-api/*.md`]
- Atom feeds (`/releases.atom`, `/tags.atom`) and their ETag behaviour: [U] (github.com blocked in this sandbox; no GitHub doc covers it). Use REST `releases/latest` with ETag and an authenticated token.

### Infrastructure items (doc 07 scope)

- **GitHub JIT runners**: `POST /repos/{owner}/{repo}/actions/runners/generate-jitconfig` (also `/orgs/{org}/...`); body `{name, runner_group_id, labels[1..100], work_folder?}`, all but `work_folder` required; 201 returns `{runner, encoded_jit_config}`; 404/409/422 possible; classic token needs `repo` scope. Start with `./run.sh --jitconfig ${encoded_jit_config}`; runner does at most one job then is auto-removed. GitHub recommends ephemeral for autoscaling and warns that reused hardware can leak state. [V: rest-api-description OpenAPI; `content/actions/reference/security/secure-use.md` L267-277; `self-hosted-runners.md` L103-122]
- **ARC**: requires a **Kubernetes cluster** (minikube/kind acceptable for local) and Helm 3; installs `gha-runner-scale-set-controller` (OCI chart on ghcr.io); each job gets an ephemeral runner pod using a JIT config. So on a single host ARC means running k3s/kind; a ~50-line custom JIT launcher is the lighter alternative (design choice, [U] on effort). [V: `get-started.md`, `actions-runner-controller.md`]
- **Forgejo/Gitea ephemeral**: Gitea server accepts an `Ephemeral` flag at runner registration and deletes the runner when its task completes (`DeleteEphemeralRunner`). [V: `go-gitea/gitea` `routers/api/actions/runner/runner.go` L83, `models/actions/task.go` L458]. Forgejo: ephemeral registration (`forgejo forgejo-cli actions register --ephemeral`; `forgejo-runner register --ephemeral` deprecated) requires Forgejo v15.0 and a runner newer than 12.5; one job then credentials invalidated. [V-secondary: WebSearch summary of forgejo.org docs and docs PR #1575; primary pages unreachable]. The `act_runner`/`forgejo-runner` client flags were not read.
- **ntfy publish**: `POST https://ntfy.example/topic` with body as message; headers `Title`, `Priority`, `Tags`, `Click`, `Actions` (aliases `X-Actions`, `Action`); JSON publish via `POST /` with `{"topic","message","actions":[...]}`. Up to **3 actions**; types `view`, `broadcast` (Android only), `http`, `copy`. Short formats: `view, <label>, <url>[, clear=true]`; `http, <label>, <url>[, method=<m>][, headers.<H>=<v>][, body=<b>][, clear=true]`; `;` separates actions, values with commas must be quoted. **The `http` action is executed by the phone/browser client, not the server**, so the approval endpoint must be reachable from the phone and carry its own auth (put a bearer token in `headers.Authorization`, or a single-use signed URL). Auth to ntfy: Basic, `Authorization: Bearer tk_...`, or `?auth=` query. Action buttons are "Supported on Android, iOS, Web". Non-ASCII headers need RFC 2047. [V: `binwiederhier/ntfy` `docs/publish.md` L1160-1330, L4312-4326]

```bash
curl -H "Authorization: Bearer $NTFY_TOKEN" -H "Title: Q-482: pin or adapt?" -H "Priority: 4" \
  -H "Actions: http, Pin, https://factory.example/answer/482?choice=pin, headers.Authorization=Bearer $ANS_TOKEN, clear=true; \
               view, Open, https://factory.example/q/482" \
  -d "fastapi 0.115 breaks POST /ingest. Recommended: constrain <0.115." https://ntfy.example/factory
```
- **Cloudflare Tunnel/Access with GitHub webhooks**: Access actions are Allow, Block, **Bypass**, Service Auth. Bypass disables enforcement and logging for matched traffic, cannot use identity selectors, and is documented for "OAuth callback URLs, webhook receivers, or health check paths". Service tokens authenticate via `CF-Access-Client-Id`/`CF-Access-Client-Secret` headers (or one configurable header via `read_service_tokens_from_header`). GitHub's webhook UI sends fixed headers (`X-GitHub-Event`, `X-GitHub-Delivery`, `X-Hub-Signature-256` HMAC) and cannot add custom ones, so a Service Auth policy would block GitHub; use a Bypass policy on the exact webhook path plus mandatory HMAC verification. Webhooks must answer 2xx within **10 s** (fpt/ghec); queue and respond. GitHub publishes sender IPs at `GET /meta` for an allow-list. [V: `cloudflare-docs` `policies/index.mdx` L55-77, `common-policies.mdx` L1084, `service-tokens.mdx` L45-79; `github/docs` `best-practices-for-using-webhooks.md`, `webhook-events-and-payloads.md` L45-50]
- **Webhook redelivery**: GitHub does **not** auto-redeliver. `GET /repos/{owner}/{repo}/hooks/{hook_id}/deliveries` (params `per_page`, `cursor`, `status`), then `POST /repos/{owner}/{repo}/hooks/{hook_id}/deliveries/{delivery_id}/attempts`. Org and GitHub App equivalents exist. Redelivery window **3 days** on fpt/ghec (7 on GHES). Redelivered deliveries keep the same `X-GitHub-Delivery` GUID, so dedupe on it. Repo admin needed. [V: OpenAPI; `redelivering-webhooks.md`, `handling-failed-webhook-deliveries.md`, `data/variables/webhooks.yml`, `best-practices-for-using-webhooks.md`]
- **Rootless Podman limits**: `--memory`, `--cpus`, `--cpuset-cpus`, `--pids-limit` (default 2048), `--ulimit`, `--runtime`. On cgroups v2 only delegated controllers can be limited; check `/sys/fs/cgroup/user.slice/user-$(id -u).slice/user@$(id -u).service/cgroup.controllers`; by default only e.g. `memory pids` are delegated; to get `cpu`/`cpuset` add `/etc/systemd/system/user@.service.d/delegate.conf` with `[Service]\nDelegate=memory pids cpu cpuset` and re-login. `--memory` is not supported on cgroups v1 rootless. [V: `containers/podman` `troubleshooting.md` L715-737, `options/memory.md`, `options/pids-limit.md`]
- **gVisor**: default platform is `systrap` (replaced `ptrace` mid-2023), works without KVM; KVM platform needs `/dev/kvm` access (kvm group) and bare metal for best performance, nested virt is slow. Rootless works via Podman/Docker's user namespace ("Method 2"); needs `newuidmap` setuid helpers for multi-UID maps, and `ignore_chown_errors = "true"` in `storage.conf` for flattened single-UID setups. `runsc --rootless` supports `runsc do` only. [V: `google/gvisor` `g3doc/user_guide/platforms.md`, `rootless.md`]
- **Firecracker**: x86_64 or aarch64 Linux, KVM module with read/write on `/dev/kvm` (`setfacl -m u:$USER:rw /dev/kvm` or `kvm` group), `tools/devtool checkenv` to validate; docs say they use `.metal` EC2 because KVM is only on metal there, so a home server must expose `/dev/kvm` (bare metal or nested virt). Getting-started resources are "not intended for production"; production hardening is in `prod-host-setup.md` and `jailer.md` (not read in depth). [V: `firecracker-microvm/firecracker` `docs/getting-started.md`]

## Corrections to wave-one docs

1. Doc 05 header: "Several primary doc sites ... were unreachable ... Renovate/Dependabot details come from search snippets and secondary posts". Now resolved from source repos; see above.
2. Doc 05 s2: "Known gap: lockFileMaintenance and transitive resolution can bypass the age gate (Renovate discussion #38115; the 44.x security preset nulls minimumReleaseAge for lockFileMaintenance)". Partly confirmed (the preset does null it for `lockFileMaintenance`, `replacement`, `pin`; docs list those as unsupported) but "44.x" and discussion #38115 are not verifiable here, and the gap is narrower than stated: Renovate passes `--before` to npm and `POETRY_SOLVER_MIN_RELEASE_AGE` to Poetry; other managers (uv, cargo, pip-compile, go) get nothing, per the docs' own FAQ.
3. Doc 05 s1 table: "Renovate flaw pre-44.3.1 skipped minimumReleaseAge for digest updates [V: releasealert.dev/CVE-2026-88884, secondary]". Not confirmed by Renovate source; the docs say the digest guarantee depends on datasource timestamps (Docker Hub `tag_last_pushed` OK; GHCR/Quay/ECR have none, so digest updates are held indefinitely under `timestamp-required`; `github-tags` ages against original commit date and cannot detect force-pushed tags). Downgrade that row to [U] and use the doc's wording.
4. Doc 05 s2: "Mend's best-practices preset defaults to a 3-day npm minimum age". Confirmed in source: `best-practices` extends `security:minimumReleaseAgeNpm` (3 days). Also note `minimumReleaseAge` default is null for plain `config:recommended`.
5. Doc 05 s2: "Dependabot `cooldown` ... The claim that GitHub made a 3-day cooldown the default on 2026-07-14 is single-source [U]". The 3-day default is [V] (GitHub docs source); only the date remains [U]. Also doc says "applies to version updates, not security updates": confirmed, and the default also excludes security updates.
6. Doc 05 s2: "pip 26.0 `--uploaded-prior-to`". Incomplete: durations (`P3D`) arrived in 26.1; 26.0 accepts only absolute datetimes. Relative cooldown in pip therefore needs >=26.1.
7. Doc 05 s1: "`npm deprecate` is the npm analogue of a soft yank [U]" - confirmed per version, reversible with an empty message. "Yank semantics similar to PyPI [U]" for crates - confirmed in substance (no new deps; lockfiles keep working).
8. Doc 05 open questions: "PyPI 14-day upload change (single source ...)" - now [V] via PyPI's own blog source, but doc 05 never states what it means. It is "no new files on releases >14 days old", not a cooldown or an upload window for fresh releases.
9. Doc 05 s3: "check the `withdrawn` field, since some are false positives". Incomplete: POST queries already exclude withdrawn records; the field only matters when fetching by ID (GET) or from the bulk export.
10. Doc 05 s1: Go detection via `@v/list`, `@latest`, `.info` timestamps - `.info` `Time` is the commit time, so it cannot serve as a trustworthy "age" signal for cooldown.
11. Doc 05 s1 table implies cooldown needs the npm `time` map; crates sparse index lines now carry `pubtime`, so crates.io cooldown needs no extra API call.

## Implications and concrete changes

### To `docs/research/05-dependency-updates-and-constraints.md`

- Replace the cooldown-tooling paragraph with a verified table: tool | option | unit | per-file or per-release | default | strict by default.
  - uv `exclude-newer` / `exclude-newer-package` | duration or timestamp | per file | none.
  - pip `--uploaded-prior-to` | `PnD` (>=26.1) | per file | none.
  - npm `min-release-age` (+`-exclude`) | days (>=11.10) | per version | null.
  - pnpm `minimumReleaseAge` | minutes | per version | 1440, non-strict unless set explicitly.
  - Cargo `registry.global-min-publish-age` | duration (Rust 1.100) | `pubtime` | 0.
  - Renovate `minimumReleaseAge` | duration string | per version | null.
  - Dependabot `cooldown.default-days` | days | per version | 3 (version updates only).
  - Go | none (proposal on hold).
- Add to s2 "What both lack" #4/#5: security updates **bypass** cooldown in both Renovate and Dependabot, matching the factory's `cooldown.security: 0d`. Keep the factory-side rule that security fixes bypass the cooldown but not canary.
- Add a Go caveat: use the factory's own first-seen timestamp (store when the watcher first saw `@v/list` contain the version), because `.info` `Time` is commit time.
- Fix OSV text (correction 9). Add `Homebrew` as an OSV ecosystem (schema >=1.9.0) so brew formula checks can use OSV.
- Add PyPI s1 row: prefer the Simple API JSON (`Accept: application/vnd.pypi.simple.v1+json`, ETag, `upload-time`, `yanked`) over `/pypi/<pkg>/json`; add the 14-day-upload note. Add crates `pubtime`.
- Constraint projection (s6): record verified native syntaxes: Renovate `allowedVersions` (cannot coexist with `matchUpdateTypes` in one rule), Dependabot `ignore.versions` plus `dependency-name`, `uv`/pip constraints. Note `ignore` also applies to Dependabot security updates, so a generated ignore can suppress a security fix; the factory's advisory scan must therefore run independently, and generated ignores should carry an expiry comment.
- Policy file (s10): add `cooldown_units: normalised` and a per-tool compile table (above); add `native_defence: {npm: min-release-age, pnpm: minimumReleaseAge+Strict, uv: exclude-newer, cargo: global-min-publish-age}` and require `minimumReleaseAgeStrict: true` when pnpm is used.
- Mark verified items [V] with the URLs in Sources and move the remaining [U] items below into Open questions.

### To `docs/BUILD_PLAN.md`

- Line 29 (Detection): keep "Renovate for detection and PR mechanics only", and add the generated-config contract: factory compiles `constraints.yaml` into `allowedVersions` (Renovate) or `ignore` (Dependabot) and native cooldowns; Renovate runs self-hosted (`ghcr.io/renovatebot/renovate`, `RENOVATE_TOKEN`, platform `github|forgejo|gitea`).
- Phase 1/2: for CI on GitHub use JIT runners via `generate-jitconfig`, with a small launcher (no Kubernetes) unless ARC is wanted, in which case budget a k3s/kind install. For Forgejo, require Forgejo >= v15 plus a recent runner for ephemeral mode, and confirm with `forgejo-runner register --help` (Open Q1).
- Open question 3 (host capacity): add checks `ls -l /dev/kvm`, `lsmod | grep kvm`, and the cgroup `cgroup.controllers` delegation file; make the `Delegate=memory pids cpu cpuset` drop-in part of the host setup script so `--cpus` and `--cpuset-cpus` work rootless. If no KVM, gVisor `systrap` is the isolation tier; if KVM exists, Firecracker is possible but its docs mark the getting-started setup as non-production.
- Open question 4 and Phase 4 (escalation): design ntfy actions as `http` actions that call a factory answer endpoint with a bearer token header (the phone makes the call, so the endpoint must be internet or VPN reachable); limit to 3 buttons; always include a `view` link to the web answer page. For GitHub webhooks via Cloudflare Tunnel: Access Bypass on `/hooks/github` only, verify `X-Hub-Signature-256`, respond within 10 s and enqueue, dedupe on `X-GitHub-Delivery`, and run a periodic reconciler that lists deliveries and redelivers failures (3-day window).
- Poll budget: ETag conditional requests on `releases/latest` with an authenticated token; 5,000/h primary, stay well below 900 points/min per endpoint.
- Update Verification item 3 and 4 (lines 71-72): mark as done, leaving only the items under "Remaining unverified".

## Remaining unverified items

- Dependabot default-cooldown rollout date (2026-07-14) and whether it applies to existing configs without `cooldown` (docs say yes: "even when `cooldown` is not configured").
- Renovate discussion #38115, the "44.x" security-preset claim, and CVE-2026-88884 (digest age bypass). Only Renovate's own docs were verified.
- Renovate JSON schema file contents (URL known, file not fetched); `renovate-schema.json` was not in the repo clone.
- Yarn `npmMinimalAgeGate`, Bun cooldown; npm packument unpublish-72h rule (cited by Renovate docs, not npm's).
- GitHub `releases.atom`/`tags.atom` ETag/conditional behaviour and any separate rate limit for web feeds.
- Forgejo/Gitea client-side flags (`act_runner register --ephemeral`, config keys) and Forgejo's primary docs; the v15 requirement is from search summaries.
- Homebrew: JSON API field names (e.g. `versions.stable`), any per-version publish timestamp, and live behaviour (host unreachable).
- OSV live behaviour (hosts unreachable): actual batch-size cap on `querybatch`, current response shapes; only docs/source read.
- Go third-party age-gating proxies (secondary only).
- Cloudflare: Tunnel `config.yml` ingress syntax and `cloudflared` run-parameter details (pages exist in the clone, not read); whether Bot Fight or WAF rules block GitHub webhook User-Agent on a free plan.
- gVisor with Podman: exact `podman run --runtime runsc` and `containers.conf` runtime stanza and the ARM/x86 support matrix (only generic rootless guidance read). Firecracker `jailer`, seccomp, and host production setup.
- Cargo `min-publish-age` requires Rust 1.100: confirm the installed toolchain version on the host.

## Sources

Docs-site source repos (cloned 2026-10-07, shallow/sparse) and live endpoints:
- Renovate: https://github.com/renovatebot/renovate (`lib/config/options/index.ts`, `docs/usage/configuration-options.md`, `docs/usage/key-concepts/minimum-release-age.md`, `docs/usage/upgrade-best-practices.md`, `lib/config/presets/internal/{config,security}.preset.ts`, `docs/usage/getting-started/running.md`)
- Dependabot: https://github.com/github/docs (`content/code-security/reference/supply-chain-security/dependabot-options-reference.md`, `data/features/dependabot-cooldown-default-days.yml`, `data/reusables/dependabot/default-cooldown-period.md`)
- GitHub REST/webhooks/Actions: https://github.com/github/docs (`content/rest/using-the-rest-api/*`, `data/reusables/rest-api/*`, `content/webhooks/**`, `content/actions/**`), OpenAPI https://raw.githubusercontent.com/github/rest-api-description/main/descriptions/api.github.com/api.github.com.json
- PEPs: https://github.com/python/peps (pep-0592, 0691, 0700, 0714); PyPI 14-day: https://github.com/pypi/warehouse (`docs/blog/posts/2026-07-22-releases-now-reject-new-files-after-14-days.md`, PR #19727); live https://pypi.org/simple/requests/, https://pypi.org/pypi/requests/json
- uv: https://github.com/astral-sh/uv (`docs/concepts/resolution.md`, `indexes.md`, `docs/guides/integration/{dependabot,renovate}.md`); pip: https://github.com/pypa/pip (`docs/html/user_guide.rst`, `src/pip/_internal/cli/cmdoptions.py`)
- npm: https://github.com/npm/cli (`workspaces/config/lib/definitions/definitions.js`, `CHANGELOG.md`, `docs/lib/content/commands/npm-deprecate.md`); live https://registry.npmjs.org/left-pad; pnpm: https://github.com/pnpm/pnpm.io (`docs/settings/dependency-resolution.md`)
- crates/Cargo: https://github.com/rust-lang/cargo (`doc/book/src/reference/{registry-index,config,resolver,publishing}.md`); live https://index.crates.io/se/rd/serde, https://index.crates.io/config.json
- Go: https://github.com/golang/website (`_content/ref/mod.md`); live https://proxy.golang.org/github.com/pkg/errors/@v/list; proposal https://github.com/golang/go/issues/76485
- Homebrew: https://github.com/Homebrew/formulae.brew.sh (`docs/api.md`), https://github.com/Homebrew/brew (`docs/Brew-Livecheck.md`, `Library/Homebrew/dev-cmd/livecheck.rb`)
- OSV: https://github.com/google/osv.dev (`docs/api/*`, `docs/faq.md`), https://github.com/ossf/osv-schema (`docs/schema.md`, `CHANGELOG.md`, `ecosystems.json`)
- ntfy: https://github.com/binwiederhier/ntfy (`docs/publish.md`)
- Cloudflare: https://github.com/cloudflare/cloudflare-docs (`cloudflare-one/access-controls/policies/*.mdx`, `service-credentials/service-tokens.mdx`)
- Gitea: https://github.com/go-gitea/gitea (`routers/api/actions/runner/runner.go`, `models/actions/{runner,task}.go`); Forgejo (secondary, WebSearch summary of https://forgejo.org/docs/latest/admin/actions/registration/ and docs PR forgejo/docs#1575)
- Podman: https://github.com/containers/podman (`troubleshooting.md`, `docs/source/markdown/options/*`); gVisor: https://github.com/google/gvisor (`g3doc/user_guide/{platforms,rootless}.md`); Firecracker: https://github.com/firecracker-microvm/firecracker (`docs/getting-started.md`)
