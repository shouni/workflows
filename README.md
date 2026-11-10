# ⚙️ Workflows

[![Status](https://img.shields.io/badge/Status-Active-brightgreen)](#)
[![GitHub tag (latest by date)](https://img.shields.io/github/v/tag/shouni/workflows)](https://github.com/shouni/workflows/tags)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

English | [日本語](README.ja.md)

The repository that holds the CI shared across the fleet. It contains no Go code.

| Contents | Role |
|---|---|
| `.github/workflows/go-ci.yml` | Reusable workflow (`workflow_call`). **Called by 24 repositories** |
| `.github/workflows/lint.yml` | Runs actionlint over the above. **This repository's own CI** |
| README (this document) | Where each repository's `ci.yml` and `dependabot.yml` point instead of writing down their reasons |

---

## 🚀 Why share it

**The goal is distributing fixes, not reducing duplication.** A workaround made necessary by
the runner or an external action (see "Design decisions" below) is written in one place and
reaches all 24 repositories. Kept per repository, the ones that have it and the ones that do
not quietly drift apart.

Per-repository differences are absorbed by inputs (see "Inputs and jobs" below).

---

## 📦 Usage

The caller's `.github/workflows/ci.yml` comes down to this:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  ci:
    uses: shouni/workflows/.github/workflows/go-ci.yml@v1
```

`on` / `concurrency` / `permissions` stay with the caller. Only the called side can declare
`workflow_call`, so the trigger definitions cannot be shared.

---

## ⚙️ Inputs and jobs

| Input | Type | Default | Meaning |
|---|---|---|---|
| `timeout-minutes` | number | `15` | Time limit for each job |
| `golangci-lint-version` | string | `v2.14.0` | golangci-lint version |
| `coverage` | boolean | `false` | Measure coverage and write it to the step summary |
| `upload-coverage` | boolean | `false` | Upload `coverage.out` as an artifact (kept 7 days). Requires `coverage: true` |
| `apt-packages` | string | `''` | apt packages to install before the tests (space-separated) |
| `fuzz-targets` | string | `''` | Space-separated `package#FunctionName`. Fails when no function matches |
| `fuzz-time` | string | `20s` | Fuzz time per target |

There are four jobs. When `fuzz-targets` is empty, `fuzz` is excluded by its condition and
**shows as skipped without running** (it does not disappear from the list).

| Job | What it does |
|---|---|
| `Build & Test` | apt → build → vet → gofmt → test (+ coverage) |
| `Lint` | golangci-lint. **The configuration is read from each repository's `.golangci.yml`** |
| `govulncheck` | Installed by hand and run |
| `Fuzz (smoke)` | Explores each of `fuzz-targets` in turn |

**When fuzzing fails, the reproducing seeds are uploaded as an artifact named `fuzz-corpus`**
(`**/testdata/fuzz/`, kept 7 days). A crashing input found in CI survives nowhere else, so
take it from there and commit it if it should become a regression test. Nothing is produced
on success.

---

## 🗂 Per-repository settings

9 repositories need `with:`; the remaining 15 only call it. **The table below lists only the
8 public ones.**

| Repository | Settings |
|---|---|
| `audio` | `fuzz-targets: "./wav#FuzzExtractAudioData ./wav#FuzzCombineToMatchesCombineWavData ./wav#FuzzCombineToDoesNotPanic"` |
| `gcp-kit` | `fuzz-targets: "./auth/session#FuzzBuildLoginRedirectURL ./auth/session#FuzzIsSafeRelativePath ./auth/oidc#FuzzExtractBearerToken"` |
| `genai-kit` | `coverage: true` / `fuzz-targets: ./gemini#FuzzCleanJSONResponse` / `fuzz-time: 60s` |
| `go-notify` | `fuzz-targets: "./slack#FuzzFormatMarkdown ./slack#FuzzSplitSectionText"` / `fuzz-time: 60s` |
| `go-review-kit` | `fuzz-targets: "./review#FuzzSanitizeJSON ./review#FuzzParseReport"` / `fuzz-time: 60s` |
| `go-utils` | `fuzz-targets: "./jobid#FuzzSanitize ./jobid#FuzzCreatedAt"` / `fuzz-time: 60s` |
| `go-web-reader` | `fuzz-targets: "./extract#FuzzText ./reader#FuzzResolveMediaType"` / `fuzz-time: 60s` |
| `netarmor` | `coverage: true` / `upload-coverage: true` / `fuzz-targets: "./securenet#FuzzValidateURL ./securenet#FuzzIsSecureServiceURL ./securenet#FuzzZonedLiteral"` / `fuzz-time: 60s` |

Each repository's `ci.yml` explains in a comment why it fuzzes those targets.

Six repositories use `fuzz-time: 60s`; `gcp-kit` and `audio` use the default `20s`. **No reason
for this difference was recorded**, so the values are kept as they are with no explanation
attached. If you decide to align them, settle the reason first.

---

## 📐 Design decisions

### Why a separate public repository

Some consumers are private. **A workflow in a public repository can be called from a private
one with no extra setup**, but not the other way round. Kept in a private repository, every
public caller would need its own access grant.

### govulncheck does not run through the action

`golang/govulncheck-action@v1` forces `GOTOOLCHAIN=local`, so when the runner lacks the
version `go.mod` requires, it fails while loading packages, before scanning anything.
`setup-go` resolves from the same `go.mod` and honours `check-latest`, so govulncheck runs on
the same toolchain as the other jobs.

**This can be reverted once the action stops overriding `GOTOOLCHAIN`.**

### `check-latest: true` on every job

Do not let a job use whatever version is baked into the runner. Standard library
vulnerabilities are fixed in patch releases, so preferring the cache leaves govulncheck
scanning an old toolchain and failing over and over (the `go` line in `go.mod` is only a
lower bound).

### External input goes through `env:`

`apt-packages` and `fuzz-targets` are not embedded in `run:` as `${{ }}`; they are put in
`env:` and referenced as shell variables. The callers are under our own control, but no form
where template expansion happens before shell parsing is left in place.

### Fuzzing treats "explored nothing" as a failure

`go test -run '^$' -fuzz '^Name$'` **returns exit 0** even when `Name` does not exist (it
prints `testing: warning: no fuzz tests to fuzz` and PASSes). Looking only at the exit code,
a renamed function, a typo or a moved package turns straight into a "green fuzz job".

**For some input paths fuzzing is the only line of defence** (URL validation in `netarmor`,
Bearer tokens in `gcp-kit`, job ID validation in `go-utils`), so no form that stops silently
is left in place. The step fails when that warning appears in the output.

As a real example, `gcp-kit` was split from `./auth` into `./auth/session` and `./auth/oidc`,
and `fuzz-targets` has already had to follow.

### apt is wrapped in retries

Mirror trouble shows up not as "failing" but as "not responding". In one observed run
`apt-get update` did not respond for 14 minutes and this step alone used up the whole job's
time limit (the `pull_request` run of the same commit finished in four and a half minutes;
the cause was the mirror, not the code). **Without a `timeout` on each command, a hang never
reaches the retry.**

**A retry count that does not fit the step's time limit is a lie.** Three attempts are sized
to fit the step's 10 minutes (the arithmetic is in the comment in `go-ci.yml`). Previously the
commands had 120/180 seconds and the step 6 minutes, so **it was cut off partway through the
second attempt and never reached three.**

### Lint your own workflows with actionlint

`go-ci.yml` is referenced by 24 repositories, yet **this repository alone had no checks at
all**. The 22 public consumers are protected by required status checks while the one handing
out the workflow was not (the 2 private ones cannot set them on the free plan).

actionlint checks the contents of `run:` too: it finds shellcheck on the runner and analyses
the commands as a shell script. **This is the pitfall when running `actionlint` locally: where
shellcheck is missing, the `run:` checks are skipped entirely.** It looks green and then fails
in CI. When this was introduced, the apt step in go-ci.yml actually tripped SC2086.

The version is pinned rather than `@latest`. govulncheck is better the newer it is, but a
linter turning red the moment a new version ships is a problem (the same reason
`golangci-lint-version` is pinned). Bumping it is manual (`go install` versions are outside
Dependabot's scope).

**A pinned linter can be broken from the Go side.** Go moves to the latest patch release with
`check-latest: true`, but golangci-lint stays pinned. In October 2026, Go 1.27.2 moved the
standard library's export data to version 5, and v2.13.1 (which supports up to version 4)
produced a `could not load export data` typecheck error in Lint for every repository; the
default went to v2.14.0. An error of this shape is a version mismatch, not the code, so bump
golangci-lint.

### External actions are referenced by commit SHA

**Tags can be moved, and this file is called by every repository.** One rewrite reaches all
of them at once. The form is the full SHA plus `# vX.Y.Z` (Dependabot bumps both).

`actions.lock` is not needed: none of the four actions used calls other actions, so the SHA
alone pins them just as far.

### `.golangci.yml` is not shared

**Only the version pin is shared; the configuration itself stays in each repository.** 15 of
the 24 match, but the differences in the other 9 (7 variants) include deliberate ones. For
example, `go-notify` adds `exhaustive` with `check: [switch, map]`, and the reason is written
in the configuration: the code that actually distinguishes `notify.Level` values is
`levelColors` in `slack`, which is a map, so a level added and missed there would pass through
as an empty-string lookup, and without the check the notification would go out uncoloured.

**Sharing it would remove the place where this knowledge lives.** The 15 that match are
simply not checked for matching; there is no separate reason they must be the same.

To try a new golangci-lint in a single repository, override `golangci-lint-version`.

---

## 🏷 Tags and rollout

### Reference the moving tag `@v1`

Callers point at `@v1`. The tag is moved on every release.

**Pinning an immutable tag (`@v1.0.0`) and letting Dependabot bump it does not work.** This is
the result of pinning a consumer to `@v1.0.0`, cutting a new version and checking across
several runs of the daily job and manual scans.

| | |
|---|---|
| Does the dependency graph detect the reference? | **Yes** (records `go-ci.yml` with its version) |
| Does Dependabot open a PR bumping the version? | **No** ("No PRs affected" every time, and no branch is created) |

**Dependency graph detection and version update support are separate systems.** A job-level
`uses:` (`jobs.<id>.uses`), unlike a step-level `uses:` (`steps[].uses`), is not covered by
updates. Do not take its presence in the dependency graph as evidence that Dependabot tracks
it.

Rewriting every consumer's ref in bulk with `sed` was also considered, but it rewrites every
repository at the same time on each update, **which looks like a staged rollout without being
one**, so it was not adopted.

### Rolling out a change

A moving tag reaches every consumer the moment it is moved. **Move it onto something broken
and everything fails at once.**

There are two guards. **What a machine can catch, `lint.yml`'s actionlint catches at PR time**
(expression references, `uses:` syntax, the contents of `run:`). But actionlint cannot catch
"syntactically correct, but doing the wrong thing". **The only guard on behaviour is the
procedure, so do not skip step 3.**

1. Change `go-ci.yml` on develop, then merge to main
2. **Cut an immutable tag `v1.x.y`** (kept as history and as a fallback if something breaks)
3. Temporarily point **a single consumer** at `@v1.x.y` and confirm its CI goes green
4. If green, move `v1` to that commit
5. The rest pick up the new content from their next CI run
6. Point the canary back at `@v1`

**Move `v1` only when `go-ci.yml` changes.** Consumers see only `go-ci.yml`, so moving the tag
for a README or `dependabot.yml` fix would stop the tag history from showing "when the
contents changed".

Put the other way round, **merging into this repository delivers nothing to the consumers.**
This works differently from an ordinary repository, where a merge takes effect immediately.
It is by design, not a delay.

---

## 🤖 Dependabot conventions

Every repository in the fleet has a `.github/dependabot.yml` of the same shape. **Each file
holds only the settings; the reasons are here.**

### All dependencies in one group

`patterns: ["*"]` fits everything into one PR. Separate PRs pile up in repositories with many
direct dependencies. More importantly, **dependencies that must stay consistent with each
other get split into separate PRs** (`gcp-kit`'s `cloud.google.com/go*` /
`google.golang.org/*` / `golang.org/x/oauth2` are the example; they used to have a dedicated
group). With grouping, an existing PR is updated rather than recreated.

### Checked daily

So that a tag pushed on an internal library (`github.com/shouni/*`) reaches its dependents as
a PR the next day. This keeps versions from being left diverged.

### Routine updates target develop, but security PRs go to main

`target-branch: develop` sends dependency updates down the same path as every other change
(develop → main), to keep main in a verified state.

**This setting applies only to routine updates.** Security fix PRs always target the default
branch (main) regardless of `target-branch`. This is GitHub's design, which separates urgent
fixes from the progress on develop, and cannot be changed by configuration. **If main and
develop diverge, merged security PRs never come down to develop**, so bring them back in line
regularly.

### Internal library updates are done by hand first

In practice the `github.com/shouni/*` versions are usually bumped by hand all at once
(`deps: bump shouni libraries` and the like), so Dependabot's PR tends to confirm that "it is
already bumped". **The setting stays anyway, to catch what the bulk update missed.** Done by
hand across dozens of repositories, something always slips.

### Ecosystems follow the repository's contents

| Kind of repository | What it declares |
|---|---|
| Go library | `gomod` + `github-actions` |
| Go app (has a `Dockerfile`) | `gomod` + **`docker`** + `github-actions` |
| Terraform | `terraform` + `github-actions` |
| **This repository** (no Go code) | `github-actions` only |

Declaring `gomod` with no Go code only runs a zero-dependency scan every day. **This
repository's `dependabot.yml` is this exception, not the standard form**, so look at a Go
repository's file if you want a model.

Only repositories with a `Dockerfile` need `docker`, and the test is not "is the build in CI".
**Images are built on Cloud Build and never show up in GitHub Actions, but that is no reason
not to track them.** A base image pinned to a version, such as `mwader/static-ffmpeg:7.1`,
stays on that version forever if nobody bumps it. A floating tag such as `golang:1.27-alpine`
picks up patches on rebuild, but never moves past `1.27` on its own.

`FROM scratch` is outside Dependabot's scope, so writing it or not changes nothing.

---

## ⚠️ Limitations

**A reusable workflow cannot have arbitrary steps inserted.** A repository with a requirement
the inputs cannot express goes back to its own `ci.yml`. Whether to add an input or let that
repository drop out is decided case by case.

The rule of thumb is "**could this requirement happen to other repositories too?**"
`apt-packages` could, so it became an input; adding inputs for requirements that only one
repository has turns the shared workflow into a collection of branches and erodes the point of
sharing it.

---

## 📄 License

MIT License. See [LICENSE](LICENSE) for details.
