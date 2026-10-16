# Design Document: Mutation Testing on Every PR

**Author:** Scott Idler
**Date:** 2026-10-14
**Status:** Implemented
**Review Passes Completed:** 5/5; review-panel round 1 folded in

## Summary

Add a reusable `mutants.yml` workflow to `github-actions-public` that runs `cargo mutants` against the diff of every change (every PR, and every push to main), in parallel with `rust-ci.yml`, and alerts on every surviving mutant that intersects the change (a test passed without checking that code). It never blocks a merge. Sharding is sized per run from the mutant count. Every Rust repo Scott controls gets a caller and an otto `mutants` task, and `scaffold` emits both for new repos.

## Problem Statement

### Background

- `cargo-mutants` makes one small change at a time (flip `==` -> `!=`, replace a body with `Default::default()`, delete a `!`) and runs the tests. **caught**: a test failed. **missed**: every test still passed, so that behavior is untested or the test covering it is dead.
- Two repos run it today, both by hand, neither in CI:
  - `tatari-tv/clyde`: `otto mutants` (`.otto.yml:154-257`), zero-survivor policy, `-f` scoped to 15 routing/report files.
  - `otto-rs/otto`: `otto mutants` (`.otto.yml:198-212`) plus `bin/mutants-gate`, a baseline ratchet.
- Measured clyde runs on desk (64 cores, sccache warm):

| run | mutants | wall | missed |
|---|---|---|---|
| 2026-08-01 | 121 | ~3.5 min | 0 |
| 2026-10-10 | 654 | 8 min | 1 |
| 2026-10-11 | 782 | 10m44s | 0 |

- Pitch: https://marquee.internal.tatari.dev/p/~scott-idler/mutation-testing-on-every-pr/

### Problem

- The 2026-10-10 miss (`common/src/repo.rs:196:68: delete ! in Resolver::prewarm`) was introduced by clyde PR #97 (`b530468`), merged green, and found 12 days later by a hand run. Fixed in #102 (`1b30141`).
- `cargo mutants --list --in-diff <#97 diff> --workspace` lists 38 mutants, and the prewarm mutant is one of them. A diff-scoped PR job would have flagged it at review time.
- A hand-run 8-11 minute job does not get run. Scott: "8min and 10m is forever."

### Goals

- Every PR on an in-scope repo runs mutants on its diff, in parallel with CI, never blocking merge. (Scott: "a non-blocking parallel code path to run mutants on every PR")
- Every push to main runs mutants on the pushed range too, so repos that land commits without PRs (most of `scottidler`) are covered. (Scott, 2026-10-15: "lets do it on push")
- A survivor intersecting the change alerts: inline annotation, step summary, one sticky PR comment on a PR; a red workflow run (GitHub's failed-run email) on a push. (Scott: "if it finds dead tests it alerts")
- Parallelism is decided per run from the mutant count, not hardcoded. (Scott: "figured out dynamically how much parallelism is right")
- One reusable workflow in `github-actions-public`; a caller plus an otto `mutants` task in every Rust repo Scott controls; `scaffold` emits both. (Scott: "add this to every .otto.yml ... and GHA as well as the scaffold")

### Non-Goals

- **Excluded: auto-fix.** No agent writes tests or pushes commits. An agent told to make survivors go away takes the cheap path (`mutants:skip`, weaken the test, change the code) and turns the check into a false green. (Scott: "sans the autoheal")
- **Excluded: merge blocking.** The check is never added to required status checks or rulesets.
- **Excluded: whole-workspace mutation per PR.** That stays the job of a repo's no-arg `otto mutants`.
- **Excluded: `otto-rs` org repos.** Outside "tatari-tv and scottidler". otto's `mutants-gate` is mined for scar tissue only.
- **Parked: re-pointing scaffold's existing `ci.yml`/`release.yml` callers** from `__ORG__/github-actions@v1` to `tatari-tv/github-actions-public@v2.x`. Revisit when: a separate scaffold-refresh doc is opened. This doc adds only the `mutants.yml` caller.

### Scope

"Every Rust repo Scott controls", counted as repos, not worktrees (GitHub API, 2026-10-14):

| org | Rust repos | archived | forks | in scope |
|---|---|---|---|---|
| `scottidler` | 64 | 11 | 2 | **51** (50 public, 1 private: `sitr`) |
| `tatari-tv` | 21 non-archived local | n/a | 0 | **21** (10 public, 11 private) |

- private and public repos are both in scope (Scott, 2026-10-14)
- archived and forked repos are out
- the authoritative list is generated in Phase 6 from the GitHub API, not hand-kept

## Proposed Solution

### Overview

```
pull_request | push:main                         caller (per repo)
  └─ uses: tatari-tv/github-actions-public/.github/workflows/mutants.yml@vX.Y.Z
       ├─ plan    diff -> cargo mutants --list --json --in-diff -> mutant list -> N
       │          artifact: pr.diff + plan.json (sha, N, expected mutant names)
       │          count == 0 -> no shards
       ├─ shard   matrix k = 0..N-1, on plan's sha
       │          install pinned cargo-mutants; otto mutants --diff --shard k/N
       │          artifact: mutants.out + rc
       └─ report  always runs; reconcile by identity -> summary + annotations
                  + sticky comment -> conclusion
```

- House rule (`rust-ci.yml:6-12`): the workflow prepares the environment and **otto runs the checks**. The shard step calls `otto mutants` with params, so `otto mutants --diff x.diff` locally is the same run CI does.

### Architecture

**1. Reusable workflow: `.github/workflows/mutants.yml`** (`workflow_call`)

- **workflow-level**
  - `permissions: contents: read`; only the report job adds `pull-requests: write`
  - `concurrency`: on `pull_request`, group `mutants-pr-${{ github.event.pull_request.number }}`, `cancel-in-progress: true`. A new push to the PR re-tests the whole PR diff, so cancelling the stale run loses nothing.
  - on `push`, group `mutants-push-${{ github.sha }}`, `cancel-in-progress: false`. Each push tests only its own `before..after` range, so cancelling push A because push B landed would drop A's change untested.
  - pins `MUTANTS_VERSION=27.1.0` and its tarball sha256 in ONE place (env at the top of the file); no version input. A newer cargo-mutants generates different mutants and silently changes what "zero" means, so the pin moves only by a release of this repo.
- **plan job**
  - checkout `fetch-depth: 2`, `persist-credentials: false` (copy `rust-ci.yml:64-66`)
  - diff on `pull_request`: `git diff HEAD^1 HEAD` on the merge commit. `origin/main` does not exist on a default PR checkout, and cargo-mutants exits 5 (`FilterDiffMismatch`) if the diff's new text does not match the tree. Proven in Phase 0.
  - diff on `push`: explicit `git -c http.extraheader="AUTHORIZATION: basic <base64 x-access-token:GITHUB_TOKEN>" fetch --depth=1 origin ${{ github.event.before }}` (the header rides this one command only; `persist-credentials` stays `false`. Phase 0: a private repo's bare fetch fails rc 128 "could not read Username"), then `git diff ${{ github.event.before }} ${{ github.sha }}`. If `before` is the zero SHA (new branch) or the fetch fails (force push dropped it), the plan job writes "incomplete: no base for push" and fails. It never falls back to `HEAD^`, which would silently test only the last commit of a multi-commit push. Proven in Phase 0.
  - install the pinned prebuilt cargo-mutants, sha256 verified inline (the release ships no `.sha256`; copy the curl pattern at `rust-ci.yml:89-107`)
  - `cargo mutants --list --json --in-diff pr.diff --workspace <task features>` -> mutant list (about 0.2s; `--list` runs `cargo metadata --no-deps`, so no private deps are fetched, `workspace.rs:102-103`)
    - zero mutants prints NOTHING on stdout (not `[]`) and exits 0. The script tests for the empty string explicitly and sets count 0; `printf '' | jq length` prints nothing, so piping straight to `jq` is wrong. Any other unparseable output fails the job loudly.
  - sizing (below) -> `N`, `J`
  - upload artifact `mutants-plan`: `pr.diff`, `plan.json` = `{sha, N, J, mutants: [names]}`
  - emit `matrix` via `$GITHUB_OUTPUT` (copy `rust-cli-release.yml:71-121`); count 0 -> empty matrix, shard job skipped
- **shard job** (`strategy.matrix.shard`, `fail-fast: false`, `timeout-minutes: ${{ inputs.shard-timeout-minutes }}`)
  - checkout `plan.json`'s sha; download `mutants-plan`
  - same env prep as `rust-ci.yml` (toolchain, rust-cache, otto, `private-deps`, and every runtime-tool input `rust-ci.yml` has, e.g. `eza-version`, `whitespace-version`), so a repo's caller passes the same `with:` to both
  - install the pinned prebuilt cargo-mutants before otto runs, so the task's self-install never fires (clyde's task otherwise `cargo install`s from source, `.otto.yml:166-175`)
  - rust-cache: one `shared-key`; only shard 0 of a `push` run saves (`save-if`), so PR runs restore main's warm cache instead of saving per-PR caches nobody else reads (Phase 0: a PR-saved cache is scoped to that PR's merge ref)
  - `df -h` before and after the otto step, printed to the log (private runners have 13-14G free)
  - `otto mutants --diff pr.diff --shard ${k}/${N} --in-place` (Phase 0 picked in-place for both runner classes; cargo-mutants 27.1.0 rejects `--in-place` with `--jobs`, so no `--jobs` is passed); otto's own exit code is NOT used for classification (otto exits 1 on any task failure, `otto src/main.rs:234-241`)
  - upload `mutants.out/` plus `rc` (cargo-mutants' raw exit code, written by the task) as `mutants-shard-${k}`, `if: always()`
- **report job** (`if: ${{ !cancelled() }}`, `permissions: pull-requests: write`; `always()` would let a cancelled run overwrite its successor's comment with "incomplete")
  - runs even when plan failed: then the conclusion is "incomplete: plan failed"
  - download plan + all shard artifacts
  - reconcile by identity (scar tissue from `otto-rs/otto/bin/mutants-gate:70-150`):
    - read `DIR/mutants.out/*`, never `DIR/*`; the latter reads empty and reports clean
    - union of mutant names across shard `outcomes.json` must equal `plan.json.mutants` exactly, no duplicates, none missing; else **incomplete**
    - a shard artifact missing -> **incomplete**
  - classify from `outcomes.json` + `rc`, enumerating every outcome (cargo-mutants' rc reports timeout ahead of missed when both occur, so the code alone hides one):

| source | condition | report |
|---|---|---|
| outcomes | any `missed` | **red**: list each with its diff |
| outcomes | any `timeout` | **red**: list each (a hang is a finding) |
| rc | 4, baseline failed | **red**: "mutation baseline failed", with the shard's baseline log excerpt |
| rc | 5 / 6, diff mismatch / invalid | **red**: "incomplete: diff derivation bug" |
| reconcile | identity mismatch, missing artifact, plan failed | **red**: "incomplete" |
| fork PR + `private-deps` | no secret available | **red**: "incomplete: private deps unavailable on fork" |
| all caught / unviable | | green |

  - `$GITHUB_STEP_SUMMARY` and the comment are bounded: counts plus the first 50 survivors, then "N more in artifact `mutants-shard-*`". The step summary caps at 1 MiB, and a 1242-mutant PR can exceed it.
  - sticky comment: upsert one comment by marker `<!-- mutants-on-prs -->` (copy `tatari-tv/gitops-infra/.github/workflows/argocd-impact.yaml:100-125`). It carries the head SHA and run URL. A clean run after a red one rewrites it to clean. Skipped on fork PRs (read-only token); annotations and summary still land.
  - on `push` there is no PR, so no comment: annotations, summary, and a failed report job. GitHub emails the pusher on a failed run; Phase 2 proves that email arrives.
  - red is never required, so it never blocks merge

**2. Otto `mutants` task contract** (every in-scope repo)

- params: `--diff FILE`, `--shard k/N`, `--jobs J`, `--in-place` (boolean); all optional. `--in-place` and `--jobs` together is a loud task error (cargo-mutants 27.1.0: `'--in-place' cannot be used with '--jobs'`)
- no params -> the repo's local audit run, unchanged (clyde keeps its `-f` list)
- `--diff` given -> the task rejects a non-empty diff with no `+++ b/` header, loudly (Phase 2: a `diff.mnemonicPrefix` diff with `i/ w/` prefixes lists 0 mutants and exits 0, a silent false green; Scott's dotfiles set it)
- `--diff` given -> repo-specific `-f` scoping is NOT applied; the diff is the scope. Excludes in `.cargo/mutants.toml` still apply, since cargo-mutants reads that file itself.
- self-install only when `cargo mutants --version` differs from `${MUTANTS_VERSION:-27.1.0}`; the reusable sets `MUTANTS_VERSION`, so CI never compiles it
- always `--workspace` (harmless on single crates: identical counts on 6 bump commits; required under `default-members`: clyde #97 lists 38 with it, 0 without)
- feature flags: exactly what that repo's `otto test` passes. clyde: none (`cargo test --workspace --no-fail-fast`, `.otto.yml:146`).
- `--sharding round-robin` (the 27.1.0 default `slice` gives `ceil(total/N)` chunks, so 5 mutants over N=4 yields 2,2,1,0, `src/shard.rs`)
- `--annotations auto`, explicit
- `--timeout 300` explicit; the auto timeout comes from an idle baseline and manufactures TIMEOUTs under parallel load (clyde `.otto.yml:215-221`)
- `RAYON_NUM_THREADS = max(1, nproc / jobs)`, with jobs = 1 under `--in-place` (clyde `.otto.yml:223-239`)
- `TMPDIR` on disk, outside any `.git` ancestor (clyde `.otto.yml:196-213`)
- writes cargo-mutants' raw exit code to `mutants.out/../rc` before exiting non-zero itself, so local runs still fail loudly

**3. Scaffold** (`~/repos/scottidler/scaffold/src/templates.rs`)

- `generate_github_workflows` (`:626-639`) also emits `.github/workflows/mutants.yml` (the caller)
- `generate_otto_yml` (`:722+`) also emits the `mutants` task per the contract
- the `scaffold-rust-repo` skill only runs the binary (`SKILL.md:51-59`), so only its "Produces" list changes

### Sizing (dynamic parallelism)

- inputs, measured or configured:
  - `count`: from the plan job's `--list`
  - `cost`: per-mutant seconds on the target runner; one constant per runner class, selected by `github.event.repository.private`. Phase 0, in-place: public 7s (4 vCPU / 16 GB / 86G free), private 11s (2 vCPU / 8 GB / 13-14G free)
  - `target-wall`: input, seconds a shard should run
  - `max-shards`: input, ceiling on runners per run
- `N = clamp(ceil(count * cost / target-wall), 1, min(max-shards, count))`; `N <= count` plus round-robin means no shard is empty
- build strategy is `--in-place` (one job) for both runner classes. Phase 0, 157 mutants: public 769s in-place vs 929-956s copy; private 1322s vs 1517-1912s; disk 3.5-4.3G in-place vs 9.4-10.4G copy-target
- defaults from Phase 0: `target-wall` 600s both; `max-shards` public 10, private 6; `shard-timeout-minutes` public 45, private 60
- private repos bill org Actions minutes per shard; `max-shards` is the spend ceiling, with its own private default from Phase 0
- observed in-diff counts: clyde #102 0, #93 2, #101 14, #92 15, #97 38, #96 157, #103 1242; bump commits 0-44
- known variance: per-mutant cost differs by dependency depth; `shard-timeout-minutes` bounds the worst case

### API Design

**Reusable inputs** (`workflow_call`)

| input | type | default | purpose |
|---|---|---|---|
| `rust-version` | string | same as `rust-ci.yml` | toolchain |
| `target-wall` | number | from Phase 0 | sizing |
| `max-shards` | number | `0` = runner-class default (public 10, private 6) | sizing ceiling |
| `shard-timeout-minutes` | number | `0` = runner-class default (public 45, private 60) | per-shard `timeout-minutes` |
| `private-deps` | boolean | `false` | same contract as `rust-ci.yml:23-54` |
| `whitespace-version`, `eza-version` | string | same as `rust-ci.yml` | runtime tools; mirrors every env input `rust-ci.yml` has |

secrets: `private-deps-token` (same as `rust-ci.yml`).

**Caller** (what scaffold emits and the fleet gets)

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
```

- the caller grants the ceiling; the reusable narrows it per job (only report writes)
- never `pull_request_target` ("the pwn-request hazard", `tatari-tv/conductor/.github/workflows/skill-attestation.yaml:30-33`)
- callers pin an immutable `vX.Y.Z` like `rust-ci.yml` callers (`scottidler/rkvr` pins `@v2.1.0`), so a fix to the reusable reaches the fleet only by a fleet bump; Phase 6 tooling is reused for that

### Implementation Plan

Ship order: `github-actions-public` -> `scaffold` -> fleet callers. Each phase is one commit, `otto ci` green where the repo has it.

#### Phase 0: Prove the environment (zero code)
**Model:** opus
- local, scratch clone of clyde at `b530468`: `cargo mutants --in-diff 97.diff --workspace -t 300` (clyde's own feature set: none)
- throwaway public repo carrying clyde at #97's parent, PR applying #97: a hand-written workflow measuring
  - `nproc`, memory, free disk on `ubuntu-latest`
  - wall time and peak disk for `--in-place --jobs 1`, `--copy-target true --jobs 2`, default copy `--jobs 2`
  - `git diff HEAD^1 HEAD` on the merge ref at `fetch-depth: 2`: exit code of `cargo mutants --list --in-diff`
  - a 2-commit push to the throwaway repo's main: `git fetch --depth=1 origin <before>` + `git diff <before> <sha>` covers both commits, and `cargo mutants --list --in-diff` exits 0
  - comment PATCH/POST from a reusable workflow, job-level `pull-requests: write`, caller-granted
  - rust-cache behavior across 2 matrix shards (`shared-key`, `save-if`)
- the same workflow in a throwaway PRIVATE `tatari-tv` repo: `nproc`, memory, disk, wall time per build strategy (GitHub docs say 2 CPU / 8 GB private vs 4 / 16 public; measure, do not assume)
- marquee on a stock `ubuntu-latest` runner with `private-deps`: does `cargo test` pass outside its ECR builder container (`marquee ci.yml:89-110`)? Recorded either way; if it fails, the reusable gains a `container` input matching marquee's builder, added in Phase 2
- record every number in the Implementation Notes; they set `cost`, `J`, build strategy, `target-wall`, `max-shards`, `shard-timeout-minutes`, per runner class
- **Success criteria:**
  - local `mutants.out/missed.txt` contains `common/src/repo.rs:196:68: delete ! in Resolver::prewarm`
  - diff derivation run exits 0 or 2, never 5
  - comment upsert from the reusable returns 200/201

#### Phase 1: Otto task contract, landed in clyde
**Model:** sonnet
- clyde's `mutants` task per the contract: params, round-robin, `rc` file, version-check install, features matching `otto test`; no-arg path unchanged
- fixture: a scratch checkout of `b530468` with this phase's `.otto.yml` copied in, and `97.diff = git diff b530468^1 b530468` (the tree matches the diff's new text, so no exit 5)
- **Success criteria:**
  - on the fixture, `otto mutants --diff 97.diff --shard 0/1 --in-place` leaves `rc` = `2` and `mutants.out/missed.txt` listing `common/src/repo.rs:196:68: delete ! in Resolver::prewarm`
  - `otto mutants --help` lists the four params
  - `otto mutants --in-place --jobs 2` exits non-zero with a message naming both flags

#### Phase 2: Reusable `mutants.yml`
**Model:** opus
- plan, shard, report jobs per Architecture; inputs per API Design; constants from Phase 0
- **Success criteria**, on the Phase 0 throwaway repo:
  - a PR touching no mutable `.rs` lines runs zero shard jobs and the report is green
  - the #97 PR: the comment lists exactly the two #97 survivors, `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` and `sessions/src/db/activity.rs:55:9: replace Db::indexed_rows ...` (Phase 0 local run: rc 2, 2 missed of 38); a second push edits that comment (comment count stays 1); restoring the test flips the same comment to clean
  - a shard forced to fail mid-run, and a fork PR with `private-deps`, each make the report red with "incomplete", never "clean"
  - a push to main adding an untested `fn` makes the report job fail with that `fn` in the summary, and GitHub's failed-run email reaches Scott
  - pushing a new branch with the push trigger temporarily widened to it (zero `before`) makes the report red with "incomplete: no base for push"

#### Phase 3: README and release
**Model:** sonnet
- `README.md` section for `mutants.yml`: caller, inputs, permissions, fork behavior
- Scott cuts the tag (operator step, `...-fleet-migration-implementation-notes.md:99-101`)
- **Success criteria:**
  - the Phase 0 throwaway repo's caller pinned to the new tag completes a run (`gh run list --workflow mutants.yml -L 1` shows `completed`)

#### Phase 4: Clyde canary
**Model:** sonnet
- rebase `mutants-otto-task` onto current clyde main; add the `+++ b/` diff-header guard to the task (contract, section 2)
- add the caller to `tatari-tv/clyde`
- canary PR (closed unmerged): revert `1b30141`'s test AND touch the `Resolver::prewarm` condition line. Reverting the test alone changes no `src` line, so `--in-diff` would never mutate prewarm.
- **Success criteria:**
  - the canary's sticky comment lists a missed `delete !` mutant in `Resolver::prewarm` (`common/src/repo.rs`)
  - the check context the run produces (from `gh api repos/tatari-tv/clyde/commits/<sha>/check-runs`) appears in neither classic `required_status_checks` nor any ruleset's required contexts

#### Phase 5: Scaffold
**Model:** sonnet
- `templates.rs`: emit the caller and the otto task; tests in `src/templates/tests.rs`; update the skill's "Produces" list; bump scaffold
- **Success criteria:**
  - `scaffold` into a temp dir produces `.github/workflows/mutants.yml` and an `.otto.yml` with a `mutants:` task
  - in the generated repo, a fixture diff adding an untested `fn` leaves `rc` = `2` with that `fn` in `missed.txt`, and a fixture diff adding a tested `fn` leaves `rc` = `0`

#### Phase 6: Fleet rollout
**Model:** sonnet
- repo list from the GitHub API per Scope
- one commit per repo, sequential (fleet-migration precedent: ssh-agent needs sandbox off; parallel Rust builds OOM desk)
- ungated -> direct push; gated -> PR, Scott merges (AI admin-merge blocked by managed settings)
- caller's `with:` copies the repo's `rust-ci.yml` caller `with:`
- pre-flight per repo: `cargo mutants --list --workspace` count > 0
- **Success criteria:**
  - every in-scope repo has a completed `mutants.yml` run on its trigger (`gh run list -R <repo> --workflow mutants.yml -L 1 --json conclusion` non-empty)

## Acceptance Criteria

- [x] The Phase 4 canary PR on `tatari-tv/clyde` gets one sticky comment naming a missed `delete !` mutant in `Resolver::prewarm`.
  - Verified 2026-10-15: clyde #110 (closed unmerged) sticky comment `## mutants: 1 surviving mutant(s) in this change` lists `missed common/src/repo.rs:196:70: delete ! in Resolver::prewarm`; run https://github.com/tatari-tv/clyde/actions/runs/38838642734 red, as designed.
  - Precondition observed 2026-10-14 on a scratch clone at `b530468`, `cargo mutants --list --in-diff 97.diff --workspace`: exit 0, 38 lines, line 2 = `common/src/repo.rs:196:68: delete ! in Resolver::prewarm`. Same command without `--workspace`: 0 lines.
- [x] A PR with no mutable `.rs` lines runs zero shard jobs.
  - Verified 2026-10-15: scottidler/mutants-probe #2 run 38824642414 logs `mutants: 0 -> 0 shard(s)`, shard job skipped, verdict `clean`; same on clyde #109 (`.otto.yml`/YAML only) and its push to main (run 38842269145).
  - Precondition observed 2026-10-14 at `1b30141` (test-only, 29 insertions): `cargo mutants --list --in-diff 102.diff --workspace` exits 0, stderr `INFO No mutants to filter`; with `--json`, stdout is empty, not `[]`.
- [x] No in-scope repo lists the mutants check context as required, in classic protection or rulesets.
  - Verified 2026-10-15 across all 44 in-scope repos (43 fleet + clyde): 37 have no classic protection (404) and no required-status rules; the 7 protected (clyde, atlassian-cli, mcp-io-rs, okta-auth-rs, rust-axum-svc, slack-cli, valet) list no context matching `mutants` in `required_status_checks` or any ruleset.
  - Observed on clyde main 2026-10-14: classic protection `.required_status_checks` = `null`; rulesets = `["deletion","non_fast_forward","workflows"]`, required contexts `[]`. Re-run across the fleet in Phase 6.
- [x] A fresh `scaffold` repo contains `.github/workflows/mutants.yml` and a `mutants:` otto task, and its fixture survivor is reported.
  - Verified 2026-10-15 with installed scaffold v0.2.0: generated repo has `.github/workflows/{ci,mutants,release}.yml` (caller `@v2.2.0`), `.otto.yml` `mutants:` task, `.gitignore` with `/rc` and `mutants.out*`. Fixture: untested `double` -> `rc` 2, `missed.txt` lists `replace * with + in double`; with a test -> `rc` 0; a `w/`-prefixed diff is refused.
  - Observed 2026-10-14: `grep -c mutants scaffold/src/templates.rs` = 0.
- [x] Every in-scope repo has a completed `mutants.yml` run.
  - Verified 2026-10-15: all 43 fleet repos `completed/success` on their main commit (37 direct pushes, 6 gated merges), plus clyde run 38854360037 on the #113 merge. In scope narrowed from the Scope table: 26 repos skipped (21 already fail `otto ci` on main, 4 lack an otto CI setup, second-brain's baseline fails on workspace-level features); listed in the implementation notes.
- [ ] A push to main with an untested change fails the run and emails the pusher.
  - Run half verified 2026-10-15: scottidler/mutants-probe run 38825348002 fails with the untested fn in the summary and annotations. Email half UNVERIFIED: the pusher's address (`scott.a.idler@gmail.com`) is not readable from this session; Scott to confirm.

## Resolved Decisions

- 2026-10-14, Scott: reusable home is `tatari-tv/github-actions-public`.
- 2026-10-14, Scott: no auto-fix; alert only.
- 2026-10-14, Scott: **private and public repos are in scope.** Overrides the panel's "later phase" rec. Consequences folded in: per-runner-class constants (Sizing), private-runner and marquee measurements (Phase 0).
- 2026-10-14, author + panel round 1 (staff accept, architect reject; architect's "make it required" contradicts Scott's no-blocking goal): **red-but-not-required** on a survivor. Green would hide a finding; "not required" already makes it non-blocking.
- 2026-10-14, author + panel round 1 (both accept; architect's baseline intersection rejected, it contradicts clyde's "THE THRESHOLD IS ZERO"): **zero policy in diff mode**. Every survivor intersecting the change is reported, including a pre-existing gap on a touched line. Opt-outs live in `.cargo/mutants.toml` or `#[mutants::skip]` with a reason, reviewable in the diff.
- 2026-10-14, author + panel round 1 (both accept): **fork PRs get annotations + summary, no comment.** A `workflow_run` two-stage commenter is parked; revisit when fork PRs become common (1 in the sampled history).
- 2026-10-14, author + panel round 1 (staff accept after contract repair; architect's objection misread the design): **the shard step calls `otto mutants`**, with the contract above (raw `rc`, version source, features, sharding mode).
- 2026-10-15, Scott: **trigger on `pull_request` and `push` to main.** Sampled `scottidler` repos land 0% of commits through PRs (bump 0/19, manifest 0/20, gx 0/27, second-brain 0/300), so PR-only never fires there. Folded in: explicit `before` fetch with "incomplete" on a zero or missing SHA, per-SHA push concurrency with no cancellation, a failed-run email proof in Phase 2.
- 2026-10-15, Scott: **everything on Rust 1.99.** Reusable defaults moved to `1.99.1` in `v2.3.0` (1.99.1 fixes a 1.99.0 x86_64 FFI bool-return miscompilation); fleet callers pin `@v2.3.0` and `rust-version: "1.99.1"`, except `platform-standard-ci` callers, which pin `"1.99"` because base-images publishes minor tags only (`base/rust:1.99-builder-base`).
- 2026-10-15, Scott (merged slack-cli #75 with the thread open): **non-member crates are out of diff scope.** `--workspace` does not reach slack-cli's `valet-client`; a PR touching only it reports clean. Accepted as a known limit.
- 2026-10-15, Scott: **`mutants.yml` lives in `github-actions-public` only**, called by public and private repos alike. Private-to-public calls already work (`atlassian-cli` calls `sast.yml@v2.1.0`), it touches no ECR/internal registry/CodeArtifact (`README.md:9`), and a second internal copy would drift like `rust-ci.yml` did (internal `github-actions#663`). Exception: if Phase 0 shows marquee needs its ECR builder container, marquee gets its own caller path.

## Alternatives Considered

### Call `cargo mutants` directly in the reusable
- **Pros:** one place for all flags; no per-repo task contract
- **Cons:** breaks the house rule (otto runs the checks); local and CI runs diverge; repo-specific flags (features, excludes) need workflow inputs
- **Why not chosen:** green-local == green-CI is the reason `rust-ci.yml` exists

### Fixed shard count
- **Cons:** most PRs have 0-44 mutants, so a fixed N pays N cold builds for nothing; a 1242-mutant PR still runs long
- **Why not chosen:** Scott asked for dynamic sizing; counting is about 0.2s

### `pull_request_target` so fork PRs get comments
- **Why not chosen:** runs fork code with a write token. Annotations cover forks for free.

### Run in `rust-ci.yml` as a step
- **Why not chosen:** serializes with CI and inherits its required-check status. Separate workflow = parallel and non-blocking by construction.

## Addendum: rejected review findings (round 1)

- **"plan job fails on private git deps"** (architect): refuted. `--list` calls `cargo metadata --no-deps` (`workspace.rs:102-103`). Shard jobs need private deps and get them.
- **"centralize sharding in the reusable"** (architect): sizing already lives in the plan job; the task only forwards `--shard k/N`.
- **"fleet kill switch via org variable"** (architect): `scottidler` is a user account, no org variables. Rollback is a per-repo revert; the immutable-tag consequence is stated under Caller.
- **"dependency depth breaks count * cost sizing"** (architect): variance is real; Phase 0 measures cost and `shard-timeout-minutes` bounds it. Noted under Sizing.

## Technical Considerations

### Dependencies
- cargo-mutants 27.1.0 prebuilt `x86_64-unknown-linux-gnu` tarball (no arm64, no `.sha256`)
- otto (pinned setup-otto SHA, `rust-ci.yml:81-87`), dtolnay toolchain, Swatinem/rust-cache

### Performance
- `copy_target` defaults false: every job copy rebuilds dependencies cold, and rust-cache only helps with `--copy-target true` or `--in-place`. Phase 0 picks.
- clyde's `target/debug` is 26 GB locally; disk per copy is the binding constraint on runners.

### Security
- reusable narrows to `contents: read` everywhere except the report job's `pull-requests: write`
- `persist-credentials: false` on checkout
- no `pull_request_target`

### Testing Strategy
- golden fixture: clyde #97 / the prewarm mutant, at every layer (local, otto, reusable, canary)
- negatives: no-`.rs` PR, forced-incomplete shard, fork PR with private deps, clean-after-red comment flip

### Rollout Plan
- Phase 4 canary on clyde before any other repo
- Phase 6 sequential; rollback is a per-repo revert of the caller

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| False green from a killed, partial, or duplicated shard | Med | High | report reconciles mutant identity against `plan.json`; mismatch is red |
| Runner disk exhaustion | Med | Med | Phase 0 measures; build strategy and `J` chosen from it |
| Private-repo Actions minutes multiply with shards | Med | Low | private `max-shards` default from Phase 0 |
| marquee cannot build outside its ECR builder container | Med | Low | Phase 0 measures; `container` input if needed |
| cargo-mutants upgrade changes the mutant set | Low | Med | one pinned version + sha256 in the reusable |
| A large survivor list exceeds summary/annotation caps | Low | Low | summary and comment bounded to 50; full list in artifacts |

## Open Questions

None. The scottidler trigger question is resolved (push:main, 2026-10-15, under Resolved Decisions).

## References

- Pitch: https://marquee.internal.tatari.dev/p/~scott-idler/mutation-testing-on-every-pr/
- Precedent: `docs/design/2026-07-10-public-rust-ci-fleet-migration.md`
- clyde task: `tatari-tv/clyde/.otto.yml:154-257`
- otto gate: `otto-rs/otto/bin/mutants-gate`
- cargo-mutants 27.1.0 source: `src/shard.rs`, `src/in_diff.rs`, `src/exit_code.rs`, `src/annotation.rs`, `src/workspace.rs`
- Panel round 1 synthesis: `~/.cache/review-panel/runs/5DqCmxv6/synthesis.md`
