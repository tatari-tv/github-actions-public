# github-actions-public

Reusable GitHub Actions workflows for Tatari's **public** repos.

## Why this repo exists

Our shared workflows live in [`tatari-tv/github-actions`](https://github.com/tatari-tv/github-actions), which is **internal**. GitHub does not let a public repo call a reusable workflow (or action) from a private or internal repo: the run fails with "this run likely failed because of a workflow file issue". So every public Tatari repo that pointed at the internal `github-actions` had broken CI.

This repo is the public home for the workflows a public repo actually needs. It is a straight port: `rust-ci.yml` and `rust-cli-release.yml` here are byte-identical to their twins in the internal `github-actions` and in `scottidler/github-actions`. Only the workflows that are safe in the open live here. Anything wired to ECR, internal registries, or CodeArtifact stays internal.

## What's here

| workflow | job | use it when |
|---|---|---|
| `rust-ci.yml` | one job: `otto ci` (fmt, clippy, whitespace, test) | any Rust repo, lib or CLI |
| `rust-cli-release.yml` | tag-driven cross-compile -> tarball + sha256 -> GitHub Release | Rust repos that ship a binary |
| `mutants.yml` | diff-scoped `cargo mutants` -> annotations, summary, sticky PR comment | any Rust repo with an otto `mutants` task; alert-only |

None of these workflows builds or pushes container images and none deploys. Those stay with the caller.

## Consume it

The caller owns the trigger. Reference a pinned major tag, never `@main`.

### CI (`.github/workflows/ci.yml`)

```yaml
name: ci
on:
  push:
    branches: ['**']
  pull_request: {}

jobs:
  ci:
    uses: tatari-tv/github-actions-public/.github/workflows/rust-ci.yml@v1
    # with:
    #   rust-version: "1.96.0"   # default
    #   private-deps: true       # only if you pull a PRIVATE git dep
    # secrets:
    #   private-deps-token: ${{ secrets.ACTIONS_CLONE_SUBMODULE_TOKEN }}
```

A public git dependency needs no token: leave `private-deps` off. Set it only when a build pulls a private repo.

### Release (`.github/workflows/release.yml`)

```yaml
name: release
on:
  push:
    tags: ['v*']
  workflow_dispatch: {}

jobs:
  release:
    uses: tatari-tv/github-actions-public/.github/workflows/rust-cli-release.yml@v1
    # with:
    #   bin-names: "clone reposlug stale-branches"   # default: repo name
    #   archive-prefix: "git-tools"                  # default: repo name
    #   targets: "linux-amd64 linux-arm64 macos-x86_64 macos-arm64"  # default: all four
    #   extra-files: "completions/foo.bash"          # optional
```

Libraries do not use `rust-cli-release.yml`: there is no binary to tarball.

### Mutation testing (`.github/workflows/mutants.yml`)

`mutants.yml` runs `cargo mutants` against the diff of a change: a PR's diff, or a push's `before..after` range. A mutant is one small edit (flip `==`, delete a `!`, replace a body with `Default::default()`). A **missed** mutant means every test still passed with that edit, so the code is untested or the test covering it is dead. Every missed mutant (and every timeout) in the change is reported.

It is alert-only. It runs in parallel with CI and never blocks a merge. **Never add it to required status checks or a ruleset.** Red means "look at this", not "you cannot merge".

```yaml
name: mutants
on:
  pull_request: {}
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write

jobs:
  mutants:
    uses: tatari-tv/github-actions-public/.github/workflows/mutants.yml@vX.Y.Z
    # with:
    #   private-deps: true       # only if you pull a PRIVATE git dep
    # secrets:
    #   private-deps-token: ${{ secrets.ACTIONS_CLONE_SUBMODULE_TOKEN }}
```

Pin an exact `vX.Y.Z` here, not the major tag. cargo-mutants is pinned inside the workflow (27.1.0, sha256 verified), and a different version generates different mutants, so a fix to the workflow reaches your repo only when you bump the pin. Never use `pull_request_target`: it runs fork code with a write token.

The caller grants `pull-requests: write`. The workflow narrows that: only the report job holds it, every other job gets `contents: read`.

#### Inputs

| input | type | default | purpose |
|---|---|---|---|
| `rust-version` | string | `1.96.0` | toolchain |
| `target-wall` | number | `600` | seconds one shard should spend testing mutants; sizes the shard count |
| `max-shards` | number | `0` | ceiling on shard runners per run; `0` = runner-class default (public 10, private 6) |
| `shard-timeout-minutes` | number | `0` | `timeout-minutes` for each shard; `0` = runner-class default (public 45, private 60) |
| `private-deps` | boolean | `false` | fetch private git deps; needs `private-deps-token`, same contract as `rust-ci.yml` |
| `whitespace-version` | string | `v0.1.6` | `tatari-tv/whitespace` release tag to install; mirrors `rust-ci.yml` |
| `eza-version` | string | `""` | `eza` release tag to install, for repos whose tests shell out to it; empty skips it |

| secret | required | purpose |
|---|---|---|
| `private-deps-token` | no | token for private git deps, used when `private-deps: true` |

Pass the same `with:` and `secrets:` to this caller that you pass to your `rust-ci.yml` caller. The shard job prepares the same environment, so what builds under `otto ci` builds here.

The shard count is sized per run: `N = ceil(count * cost / target-wall)`, clamped to `[1, min(max-shards, count)]`. `count` is the number of mutants in the diff, `cost` is a measured per-mutant constant for the runner class (public 7s, private 11s; private repos get smaller runners and bill org minutes per shard). A diff with no mutable `.rs` lines runs zero shards and reports clean.

#### The otto `mutants` task

The workflow prepares the environment and **otto runs the checks**, same as `rust-ci.yml`. Your repo needs an otto `mutants` task, so `otto mutants --diff x.diff` locally is the run CI does. The contract:

- four optional params: `--diff FILE`, `--shard k/N`, `--jobs J`, `--in-place` (boolean). No params runs the repo's local audit unchanged.
- `--in-place` with `--jobs` is a loud error (cargo-mutants rejects the pair). CI passes `--in-place` and no `--jobs`.
- with `--diff`, the diff is the scope: repo-specific `-f` file lists do not apply. Excludes in `.cargo/mutants.toml` still do.
- with `--diff`, a non-empty diff with no `+++ b/` header is rejected loudly. A diff written with `diff.mnemonicPrefix` (`i/ w/` prefixes) lists zero mutants and exits 0, which reads as a false green. The workflow writes its own diff with explicit `a/ b/` prefixes; the guard covers hand-made diffs.
- always `--workspace`, with the feature flags your `otto test` passes.
- `--sharding round-robin`, so no shard is empty.
- explicit `--timeout 300` and `--annotations auto`.
- writes cargo-mutants' raw exit code to an `rc` file next to `mutants.out/`, then exits non-zero itself on failure. The report reads `rc`, not otto's exit code (otto exits 1 on any task failure).
- self-installs cargo-mutants only when `cargo mutants --version` differs from `${MUTANTS_VERSION:-27.1.0}`. The workflow sets `MUTANTS_VERSION` and installs the pinned binary first, so CI never compiles it.

#### What you get back

- **Pull request, same repo:** inline annotations, a step summary, and one sticky comment, rewritten in place on every push (marker `<!-- mutants-on-prs -->`). A clean run after a red one flips the comment to clean.
- **Fork PR:** annotations and step summary only. A fork token is read-only, so no comment. With `private-deps: true` a fork gets no secret, so the run reports `incomplete: private deps unavailable on fork`.
- **Push to main:** there is no PR, so no comment. You get annotations, the summary, and a **failed run**; GitHub emails the pusher on a failed run. A push whose `before` SHA is zero (a new branch) or cannot be fetched (a force push dropped it) has no base to diff, and reports `incomplete: no base for push`. It never falls back to the last commit, which would test only part of a multi-commit push.

The summary and comment list the first 50 survivors, then point at the `mutants-shard-*` artifacts for the rest.

"Clean" is reported only when every planned mutant has an outcome from exactly one shard. A killed shard, a missing artifact, a failed plan, or a baseline that fails its own tests is **red**, never clean:

| finding | report |
|---|---|
| any missed mutant | red, each listed with its diff |
| any timeout | red, each listed (a hang is a finding) |
| baseline build or tests failed | red, `mutation baseline failed`, with the log excerpt |
| diff derivation error (cargo-mutants rc 5 / 6) | red, `incomplete: diff derivation bug` |
| outcome mismatch, missing artifact, plan failed, fork + `private-deps` | red, `incomplete` |
| all caught or unviable | green |

## Versioning

Reference a stable major tag (`@v1`) for `rust-ci.yml` and `rust-cli-release.yml`. `mutants.yml` is the exception: pin an exact `@vX.Y.Z` (see its section). The point is a public repo pins one line and never re-reads a whole CI definition it would otherwise have to keep in sync by hand. When these workflows change in the internal `github-actions`, the change lands here too and the major tag moves forward.
