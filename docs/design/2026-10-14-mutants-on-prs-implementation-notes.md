# Implementation Notes: Mutation Testing on Every PR

Design doc: `docs/design/2026-10-14-mutants-on-prs.md`

## Phase 0: Prove the environment

Measured 2026-10-15. cargo-mutants 27.1.0 everywhere (prebuilt `x86_64-unknown-linux-gnu` tarball, sha256 `dfe6dc37d0342c891d2829b5a695aa57c2d0edecef7e7d0399a30cc6e206411e`, verified on desk and inside every workflow). Toolchain 1.96.0 via `dtolnay/rust-toolchain`.

Throwaway repos (measurement workflows live only there; kept for Phases 2-3):

- public: https://github.com/scottidler/mutants-probe, PR https://github.com/scottidler/mutants-probe/pull/1
- private: https://github.com/tatari-tv/mutants-probe-private, PR https://github.com/tatari-tv/mutants-probe-private/pull/1

Both carry clyde history up to `2a4c67f` (#96, the parent of `b530468`), then one commit replacing clyde's 5 workflows with `probe.yml` (env, diff, strategy matrix, cache matrix), `caller.yml` and `comment.yml` (the reusable). PR #1 cherry-picks `b530468` (#97). clyde's git deps (`tatari-tv/mcp-io-rs`, `tatari-tv/renew`) are public, so the public repo builds clyde with no token and no substitute was needed.

### Success criteria

| criterion | result | evidence |
|---|---|---|
| local `mutants.out/missed.txt` contains `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` | PASS | missed.txt line 1, below |
| diff derivation exits 0 or 2, never 5 | PASS | PR merge ref: `LIST-RC=0`, 38 mutants (public and private); push range: `LIST-RC=0`, 9 mutants (public and private) |
| comment upsert from the reusable returns 200/201 | PASS | first run: `POST HTTP=201`, then `PATCH HTTP=200`; every later run `PATCH HTTP=200`; marker comment count stays 1 (public and private) |

### 1. Local (desk, 64 cores, sccache warm)

Scratch clone of clyde at `b530468`, `97.diff = git diff b530468^1 b530468` (604 lines), `TMPDIR=/tmp/claude-1000` (no `.git` ancestor).

```
$ cargo mutants --list --in-diff 97.diff --workspace | head -3
common/src/repo.rs:196:9: replace Resolver::prewarm with ()
common/src/repo.rs:196:68: delete ! in Resolver::prewarm
session/src/parse.rs:71:5: replace parse_sessions -> Vec<ParsedSession> with vec![]

$ cargo mutants --in-diff 97.diff --workspace -t 300
Found 38 mutants to test
ok       Unmutated baseline in 11s build + 3s test
MISSED   common/src/repo.rs:196:68: delete ! in Resolver::prewarm in 1s build + 1s test
MISSED   sessions/src/db/activity.rs:55:9: replace Db::indexed_rows -> Result<HashMap<String, IndexedRow>> with Ok(HashMap::new()) in 1s build + 2s test
38 mutants tested in 83s: 2 missed, 8 caught, 28 unviable
rc=2
Elapsed (wall clock) time: 1:23.24
Maximum resident set size (kbytes): 689068

$ cat mutants.out/missed.txt
common/src/repo.rs:196:68: delete ! in Resolver::prewarm
sessions/src/db/activity.rs:55:9: replace Db::indexed_rows -> Result<HashMap<String, IndexedRow>> with Ok(HashMap::new())
```

Per-mutant means from `outcomes.json` (build + test): CaughtMutant n=8 4.3s, MissedMutant n=2 3.6s, Unviable n=28 0.9s. Baseline: Build 11.17s, Test 3.96s.

### 2. Runner shape

| | public (`scottidler/mutants-probe`) | private (`tatari-tv/mutants-probe-private`) |
|---|---|---|
| `nproc` | 4 | 2 |
| CPU | AMD EPYC 9V45 / 9V74 | AMD EPYC 9V45 / Intel Xeon 6973P-C |
| memory (`free -m` total) | 15989 MB | 7938 MB |
| swap | 3071 MB | 3071 MB |
| `/` (`df -h`) | 145G size, 59G used, 86G avail | 72G size, 59G used, 13-14G avail |
| other disks | none (`/mnt` absent) | none |
| `/tmp`, `$RUNNER_TEMP` | on `/` (ext4), `TMPDIR` unset | same |

GitHub's documented 4 CPU / 16 GB public vs 2 CPU / 8 GB private holds. The number the docs do not state: the private runner has 13-14 GB of free disk, versus 86 GB on public.

`dtolnay/rust-toolchain` exports `CARGO_INCREMENTAL=0` for every later step, so every per-mutant rebuild in these numbers is non-incremental.

### 3. Build strategies

Each strategy job: checkout at `fetch-depth: 0`, toolchain, cargo-mutants, then a `cargo test --workspace --no-run` prebuild (stands in for a warm rust-cache), then `cargo mutants --in-diff <diff> --workspace -t 300 <flags> --output $RUNNER_TEMP` with `RAYON_NUM_THREADS = nproc / jobs`. Disk and memory were sampled every 2s with `df -B1M --output=used /` and `free -m`; "delta" is peak minus the pre-prebuild baseline, so it includes the prebuild's `target/`.

`--in-place --jobs 1` (the design's spelling) is rejected by cargo-mutants 27.1.0 before any work:

```
error: the argument '--in-place' cannot be used with '--jobs <JOBS>'
Command exited with non-zero status 1
MUTANTS wall=0.01s
rc=1
```

The in-place rows below are `--in-place` with no `--jobs` (it is single-job by construction).

**Public, #97 (38 mutants: 8 caught, 2 missed, 28 unviable)**, two runs (run 1 had the in-place flag error):

| strategy | run | mutants wall | baseline build + test | prebuild | mean s/mutant (all) | sum s | disk delta | mem peak | rc |
|---|---|---|---|---|---|---|---|---|---|
| `--in-place` | 2 | 303.83s | 23.86s + 16.53s | 81.98s | 6.92 | 262.9 | 3539 MB | 2692 MB | 2 |
| `--copy-target true --jobs 2` | 1 | 281.12s | 32.34s + 16.22s | 74.20s | 11.59 | 440.3 | 9415 MB | 2496 MB | 2 |
| `--copy-target true --jobs 2` | 2 | 291.22s | 33.54s + 16.84s | 75.60s | 12.00 | 456.1 | 9442 MB | 2651 MB | 2 |
| default copy `--jobs 2` | 1 | 300.08s | 39.86s + 15.37s | 78.15s | 12.80 | 486.4 | 4824 MB | 2685 MB | 2 |
| default copy `--jobs 2` | 2 | 267.43s | 35.57s + 13.01s | 70.11s | 11.44 | 434.7 | 4827 MB | 2487 MB | 2 |

**Public, #96 (157 mutants: 48 caught, 6 missed, 103 unviable)**, `workflow_dispatch` with `range="2a4c67f^1 2a4c67f"`:

| strategy | mutants wall | baseline build + test | prebuild | mean s/mutant (all) | sum s | disk delta | mem peak | rc |
|---|---|---|---|---|---|---|---|---|
| `--in-place` | 769.03s | 21.03s + 24.77s | 54.06s | 4.60 | 722.7 | 4266 MB | 2486 MB | 2 |
| `--copy-target true --jobs 2` | 955.96s | 53.37s + 32.59s | 77.21s | 10.98 | 1724.2 | 10361 MB | 2785 MB | 2 |
| default copy `--jobs 2` | 928.55s | 57.13s + 30.52s | 70.44s | 10.69 | 1677.7 | 6988 MB | 2585 MB | 2 |

**Private, #97 (38 mutants)**:

| strategy | run | mutants wall | baseline build + test | prebuild | mean s/mutant (all) | sum s | disk delta | mem peak | rc |
|---|---|---|---|---|---|---|---|---|---|
| `--in-place` | 2 | 473.47s | 36.90s + 26.66s | 153.59s | 10.77 | 409.3 | 3540 MB | 1854 MB | 2 |
| `--copy-target true --jobs 2` | 1 | 526.20s | 61.35s + 25.30s | 134.92s | 21.01 | 798.5 | 9445 MB | 2183 MB | 2 |
| `--copy-target true --jobs 2` | 2 | 418.05s | 45.87s + 17.66s | 94.33s | 16.65 | 632.5 | 9451 MB | 2147 MB | 2 |
| default copy `--jobs 2` | 1 | 612.80s | 78.11s + 27.87s | 152.11s | 26.53 | 1008.0 | 4857 MB | 2138 MB | 2 |
| default copy `--jobs 2` | 2 | 542.82s | 69.47s + 25.84s | 143.11s | 23.35 | 887.4 | 4865 MB | 2224 MB | 2 |

**Private, #96 (157 mutants)**:

| strategy | mutants wall | baseline build + test | prebuild | mean s/mutant (all) | sum s | disk delta | mem peak | rc |
|---|---|---|---|---|---|---|---|---|
| `--in-place` | 1321.83s | 37.76s + 41.40s | 101.00s | 7.91 | 1242.1 | 4268 MB | 1964 MB | 2 |
| `--copy-target true --jobs 2` | 1516.65s | 87.74s + 49.72s | 121.78s | 17.05 | 2677.4 | 10410 MB | 2072 MB | 2 |
| default copy `--jobs 2` | 1911.70s | 122.08s + 55.41s | 147.56s | 22.06 | 3462.9 | 7019 MB | 2094 MB | 2 |

Every strategy, both classes, both diffs, found the same survivors. #97: `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` and `sessions/src/db/activity.rs:55:9: replace Db::indexed_rows -> Result<HashMap<String, IndexedRow>> with Ok(HashMap::new())`. #96: six in `clyde/src/doctor.rs` `reprobe` (lines 396-397: two `delete match arm`, four `replace +=`). No timeouts anywhere.

Reading the tables:

- At 38 mutants the three strategies land inside run-to-run noise (public 267-304s). At 157 mutants `--in-place` wins on both classes: public 769s vs 929-956s, private 1322s vs 1517-1912s.
- `--jobs 2` halves the cores each cargo build gets, and the per-mutant time roughly doubles (public #96: 4.60s -> 10.69-10.98s), so two jobs buy nothing on 2-4 vCPUs.
- Disk: `--copy-target true` peaks at 9.4-10.4 GB over baseline for clyde. On the private runner that leaves 3-4 GB of the 13-14 GB free; a repo with a larger `target/` hits ENOSPC. `--in-place` peaks at 3.5-4.3 GB.
- Memory never passed 2.8 GB used on either class.

### 4. Diff derivation

**PR (`pull_request`, checkout `fetch-depth: 2`, `git diff HEAD^1 HEAD`)**. HEAD is GitHub's merge commit, HEAD^1 the base tip:

```
0eee707 Merge 6a540a8c0f8b3f66bcd61eba2c6ef81090ff8466 into 4418ffd871b939bd9d1765067ddffc176fae8c25
4418ffd probe: --in-place rejects --jobs; drop it
6a540a8 perf(sessions): parallel session parse (#97)
diff-rc=0
+++ b/Cargo.lock ... +++ b/sessions/src/index/tests.rs   (10 files, same as #97)
LIST-RC=0
count=38
LIST-JSON-RC=0 bytes=88729 jq-length=38
```

Same on the private repo (`LIST-RC=0`, `count=38`), on all four PR runs.

**Push, 2 commits (public)**. Commit A added `pub fn probe_push_a` to `common/src/lib.rs`, commit B added `pub fn probe_push_b` to `session/src/lib.rs`, pushed together:

```
event=push sha=a6bb707117b834d52baf7a5e32eb9f20ac22fabb
before=3628d8b51eb714ed0b42fa22bc2d1834b02dfc97
--- fetch before, anonymous (persist-credentials: false)
 * branch            3628d8b51eb714ed0b42fa22bc2d1834b02dfc97 -> FETCH_HEAD
anon-fetch-rc=0
diff-rc=0
--- commits in before..sha
a6bb707 probe: push commit B (session fn)
6c50c32 probe: push commit A (common fn + derive-diff errexit fix)
--- files in diff
+++ b/.github/workflows/probe.yml
+++ b/common/src/lib.rs
+++ b/session/src/lib.rs
LIST-RC=0
count=9
common/src/lib.rs:64:5: replace probe_push_a -> bool with true
common/src/lib.rs:64:5: replace probe_push_a -> bool with false
common/src/lib.rs:64:7: replace > with == in probe_push_a
common/src/lib.rs:64:7: replace > with < in probe_push_a
common/src/lib.rs:64:7: replace > with >= in probe_push_a
session/src/lib.rs:43:5: replace probe_push_b -> u32 with 0
session/src/lib.rs:43:5: replace probe_push_b -> u32 with 1
session/src/lib.rs:43:7: replace * with + in probe_push_b
session/src/lib.rs:43:7: replace * with / in probe_push_b
LIST-JSON-RC=0 bytes=9659 jq-length=9
```

Both commits' mutants are listed, so `before..sha` covers the whole push.

**Push, 3 commits (private), same probe**. The anonymous fetch of `before` fails on a private repo, because `persist-credentials: false` leaves no credential; the job token passed as an extraheader for that one fetch works:

```
before=e7618dade09e155abc2eca19e445b660990cb3a7
--- fetch before, anonymous (persist-credentials: false)
fatal: could not read Username for 'https://github.com': No such device or address
anon-fetch-rc=128
--- fetch before, with job token as extraheader
auth-fetch-rc=0
diff-rc=0
LIST-RC=0
count=9
```

`git log before..sha` showed only 2 of the 3 commits (the checkout is shallow at depth 2), but `git diff before sha` is tree-to-tree and the list still carries commit A's `probe_push_a` mutants, so the diff covers all 3.

**Push to a new branch (zero `before`)**. The first push of `main` to the empty repo:

```
before=0000000000000000000000000000000000000000
fatal: remote error: upload-pack: not our ref 0000000000000000000000000000000000000000
##[error]Process completed with exit code 128.
```

**Push touching no `.rs`**. `workflow_dispatch` on main with an empty range diffed a `.yml`-only commit: `LIST-RC=0`, `count=0`.

### 5. Sticky comment from a reusable workflow

`caller.yml` mirrors the design's caller: workflow-level `contents: read` + `pull-requests: write`, one job `uses: scottidler/mutants-probe/.github/workflows/comment.yml@main`. The reusable sets workflow-level `contents: read`; job `upsert` adds `pull-requests: write`; job `readonly` (negative control) keeps `contents: read` only. The upsert finds the comment by `<!-- mutants-on-prs -->` prefix, then PATCHes it or POSTs a new one, twice per run.

| run | repo | readonly POST | upsert 1 | upsert 2 | marker comments |
|---|---|---|---|---|---|
| 1 | public | 403 `Resource not accessible by integration` | POST 201 | PATCH 200 | 1 |
| 2 | public | 403 | PATCH 200 | PATCH 200 | 1 |
| 1 | private (calls the public reusable cross-owner) | 403 `Resource not accessible by integration` | POST 201 | PATCH 200 | 1 |
| 2 | private | 403 | PATCH 200 | PATCH 200 | 1 |

The caller grants the ceiling, the reusable's job-level `pull-requests: write` is what lets the write through, and a job without it gets 403. A private `tatari-tv` repo calling a public reusable in another owner's repo works.

### 6. rust-cache across 2 matrix shards

`Swatinem/rust-cache@c19371144df3bb44fab255c43d04cbc2ab54d1c4` (v2.9.1), `shared-key: mutants-probe`, `save-if: ${{ matrix.shard == 0 }}`, then `cargo test --workspace --no-run`.

| run | shard | cache-hit | build wall (public) | build wall (private) | post step |
|---|---|---|---|---|---|
| 1 | 0 | false (`No cache found.`) | 79.18s | 149.04s | saved, `Sent 302190791 of 302190791` |
| 1 | 1 | false | 74.14s | 148.76s | no save (save-if false) |
| 2 | 0 | true, `full match: true` | 27.85s | 53.99s | `Cache up-to-date.` |
| 2 | 1 | true, `full match: true` | 22.99s | 54.52s | no save |

- Both shards compute the same key, `v0-rust-mutants-probe-Linux-x64-e0351c3c-5b1fb11c`, so one save serves every shard.
- Exactly one shard uploads per run, and a hit skips the upload.
- The saved cache is scoped to its ref: `gh api .../actions/caches` shows `ref = refs/pull/1/merge`, 302190791 bytes. A cache saved by a PR run is readable by later runs of that PR only; caches saved on `main` are readable by every PR.

### 7. marquee on a stock runner

Run `tatari-tv/mutants-probe-private` `marquee.yml`, https://github.com/tatari-tv/mutants-probe-private/actions/runs/38817995061. Plain `ubuntu-latest` (2 vCPU / 7938 MB / 72G disk, 13G free), no container, marquee `62b75b5` (main), private deps via the org secret `ACTIONS_CLONE_SUBMODULE_TOKEN` as an `insteadOf` rewrite plus `CARGO_NET_GIT_FETCH_WITH_CLI=true` (the same contract as `rust-ci.yml` `private-deps`), then `cargo test --all-features` (marquee's `otto test` task):

```
Updating git repository `https://github.com/tatari-tv/okta-auth-rs.git`
Compiling okta-auth v0.7.0 (https://github.com/tatari-tv/okta-auth-rs.git?tag=v0.7.0#d830c400)
... 27 x "test result: ok", 0 x "test result: FAILED"
TEST wall=327.55s maxrss=2856260KB
CARGO-TEST-RC=0
/dev/root        72G   65G  6.7G  91% /
```

Verdict: **yes**. marquee's tests pass outside its ECR builder container. No `container` input is needed, and marquee needs no caller path of its own. The build left 6.7 GB free on the private runner.

### 8. Derived constants per runner class

`cost` is the mean wall-seconds per mutant at `J = 1` in-place, rounded up from the worst observed diff. Fixed per-shard overhead is setup (checkout, toolchain ~8-10s, cargo-mutants install ~1s) plus the baseline: with a rust-cache hit, the workspace build was 23-28s public and 54s private, and the baseline test pass was 16-25s public and 27-41s private.

| constant | public | private | measurement behind it |
|---|---|---|---|
| `cost` | 7 s/mutant | 11 s/mutant | in-place mean, all outcomes: public #97 6.92, #96 4.60; private #97 10.77, #96 7.91 |
| `J` | 1 | 1 | `--jobs 2` doubles per-mutant time on 4 and 2 vCPUs (public #96 4.60 -> 10.69-10.98 s) and was slower at 157 mutants on both classes |
| build strategy | `--in-place` | `--in-place` | fastest at 157 mutants (public 769s vs 929-956s; private 1322s vs 1517-1912s), smallest disk peak (3.5-4.3 GB vs 4.8-10.4 GB), and uses the rust-cache `target/` directly. Private free disk is 13-14 GB, so `--copy-target` (9.4-10.4 GB for clyde) is unsafe there |
| `target-wall` | 600 s | 600 s | #97 -> N=1 on both (266s public, 418s private); #96 -> N=2 public (1099s), N=3 private (1727s); per-shard overhead (setup + baseline, 1-2 min) stays under a third of the shard |
| `max-shards` | 10 | 6 | public: half of a Free account's 20 concurrent jobs, so a big PR leaves CI room; #103 (1242 mutants) -> 10 shards x 125 mutants x 7s = 875s each. private: spend is roughly `count x cost + N x overhead`, so N only adds overhead minutes; #103 -> 6 x 207 x 11s = 2277s each |
| `shard-timeout-minutes` | 45 | 60 | worst capped shard: public #103 ~15 min + overhead, private #103 ~38 min + overhead (cold private prebuild up to 153.59s); leaves room for a few `-t 300` timeouts before the job is killed |

### Design decisions

- Measured on clyde as it stands, no substitute repo (`probe.yml`): clyde's git deps (`mcp-io-rs`, `renew`) are public, so the public repo builds it with no token.
- Ran #96 (157 mutants) as a second diff via `workflow_dispatch` (`probe.yml` `strategy` job): #97's 38 mutants (28 unviable) put all three strategies within noise; 157 separates them.
- A `cargo test --workspace --no-run` prebuild stands in for a warm rust-cache in the strategy jobs (`probe.yml` `strategy`): it gives `--copy-target` and `--in-place` their warm-`target/` advantage on equal terms; rust-cache itself was measured separately in the `cache` job.
- The sticky-comment probe calls the reusable by full path, `scottidler/mutants-probe/.github/workflows/comment.yml@main`, not `./` (`caller.yml`): the production call is cross-repo, and the private repo exercises private-to-public cross-owner.
- A `readonly` job with no `pull-requests: write` is the negative control (`comment.yml`): a 201 alone does not prove the job-level grant is what lets the write through; its 403 does.
- Constants set to `J=1`, `--in-place` for both classes (section 8), from the measurements, not the docs' CPU counts.

### Deviations

- `--in-place --jobs 1` (Phase 0 bullet and Sizing) is measured as `--in-place` with no `--jobs`: cargo-mutants 27.1.0 rejects the pair (`the argument '--in-place' cannot be used with '--jobs <JOBS>'`, rc 1). Same strategy, the only accepted spelling.
- The local run used `TMPDIR=/tmp/claude-1000` (tmpfs, 48G free, no `.git` ancestor) instead of clyde's `/var/tmp/clyde-mutants`: the scratchpad was the requested location, and the `.git`-ancestor rule from clyde's `.otto.yml` still held.
- Pushes to both throwaway repos went out from the `github-actions-public` working copy (`git fetch <scratch clone> +main:refs/probe/...`, then `git push <url> refs/probe/...:refs/heads/...`): this subagent's Bash cwd does not persist between calls, and the git-network hook only allows a bare `git push` from cwd. The first such fetch omitted `--no-tags` and copied clyde's `v0.*` tags into this repo's local tag list (see Open questions). Nothing was pushed to `tatari-tv/github-actions-public`.
- The throwaway repos' `main` carries commits beyond the one workflow commit (the `probe.yml` errexit fix and the in-place flag fix rode along with the 2-commit push test; private also has `marquee.yml`): the push test needed a fixed workflow at the pushed sha.

### Tradeoffs

- `--in-place` vs `--copy-target true --jobs 2`: in-place is fastest at 157 mutants, peaks at 3.5-4.3 GB, and copies nothing. Copy-target won one private #97 run (418s vs 473s), lost the other (526s), lost at 157 mutants on both classes, and needs 9.4-10.4 GB, which the private runner barely has.
- `max-shards` 10 public vs 20: 20 matches a Free account's concurrency and would starve every other `scottidler` repo's CI during a large PR; 10 keeps a 1242-mutant PR near 15 min per shard.
- `cost` rounded up from the slowest diff (#97) vs averaging #96 and #97: overestimating cost costs one extra shard; underestimating runs a shard past `target-wall`.
- Prebuild in strategy jobs vs rust-cache in strategy jobs: prebuild gives every matrix leg an identical warm `target/` with no cache-key race between legs. The price is that strategy wall times exclude cache-restore time (measured in the `cache` job instead).

### Open questions

- Phase 2's criterion "the comment lists exactly the prewarm survivor" cannot pass as written: #97 has two survivors on every run (local, public, private), `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` and `sessions/src/db/activity.rs:55:9: replace Db::indexed_rows -> Result<HashMap<String, IndexedRow>> with Ok(HashMap::new())`. The 2026-10-10 hand run missed the second because its `-f` list excludes `sessions/src/db/activity.rs`. Reword to "lists the prewarm survivor" or "lists exactly these two"?
- The otto contract (Architecture 2) always forwards `--jobs J`. With `--in-place` chosen, cargo-mutants rejects `--jobs`. Should the task omit `--jobs` when it is 1, or should the contract gain a strategy param?
- Push diff on a private repo: with `persist-credentials: false` the plan job's `git fetch --depth=1 origin <before>` fails (rc 128). The plan job needs the job token on that one fetch (extraheader, as measured) or `persist-credentials: true`. Which one goes into Phase 2?
- rust-cache: a cache saved by a PR run is readable only by that PR (`ref = refs/pull/1/merge`). For PRs to start warm, the shard-0 save has to happen on `push: main` runs. Confirm Phase 2 saves only there (`save-if: github.event_name == 'push' && matrix.shard == 0`)?
- `dtolnay/rust-toolchain` sets `CARGO_INCREMENTAL=0`, so every per-mutant rebuild here is non-incremental. Setting `CARGO_INCREMENTAL=1` in the shard job may cut `cost`. Not measured. Measure in Phase 2?
- The private runner has 13-14 GB free. A private repo with a `target/` much larger than clyde's (3.5-4.3 GB in-place peak) may run out of disk even in-place. Phase 2 could free space (remove preinstalled toolchains) or record free disk per shard. Which?
- This repo's local tag list now holds 59 of clyde's tags, `v0.1.0` through `v0.25.7`, fetched by mistake (no collision with `v1`, `v1.0.1`, `v2.0.0`, `v2.1.0`; local only; `push.followTags` is false; none pushed). The tag rule says never delete a tag without Scott's explicit OK. Delete them locally (`git tag -d`), or leave them?

## Phase 0 follow-up: orchestrator rulings on the Phase 0 open questions

### Design decisions
- Q1 (Phase 2 "exactly the prewarm survivor"): doc defect. #97's diff carries a second survivor, `sessions/src/db/activity.rs:55:9` (Phase 0 local run, rc 2, 2 missed of 38). Criterion amended to list both.
- Q2 (`--in-place` vs `--jobs`): the task contract gains a boolean `--in-place` param; the reusable passes `--in-place` and no `--jobs`; passing both is a loud task error. Phase 1 criteria amended to match.
- Q3 (private push fetch rc 128): a per-command `http.extraheader` carrying `GITHUB_TOKEN` for the `before` fetch only; `persist-credentials: false` stays.
- Q4 (cache scope): only shard 0 of a `push` run saves; PR runs restore main's cache.
- Q6 (private disk 13-14G): `df -h` before/after the otto step in every shard log. No space-freeing step until a repo runs out.
- Sizing defaults from Phase 0 written into the doc's Sizing section.

### Deviations
- None.

### Tradeoffs
- Q3 extraheader vs `persist-credentials: true`: the extraheader keeps the token out of `.git/config` for the shard's test run, which executes PR code.

### Open questions
- Q5 (`CARGO_INCREMENTAL=0` from dtolnay/rust-toolchain): parked; revisit only if per-mutant cost becomes the problem.
- Q7 (59 clyde tags copied into the local tag list of `github-actions-public`, local only, remote verified clean): left for Scott to delete by hand per the tag rule.
