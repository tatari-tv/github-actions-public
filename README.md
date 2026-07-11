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

Neither workflow builds or pushes container images and neither deploys. Those stay with the caller.

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

## Versioning

Reference a stable major tag (`@v1`). The point is a public repo pins one line and never re-reads a whole CI definition it would otherwise have to keep in sync by hand. When these workflows change in the internal `github-actions`, the change lands here too and the major tag moves forward.
