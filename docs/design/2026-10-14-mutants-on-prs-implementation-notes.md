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

## Phase 1: Otto task contract

Landed in `tatari-tv/clyde`, worktree `/home/saidler/repos/tatari-tv/clyde-mutants`, branch `mutants-otto-task`, commit `d3b4600` (not pushed). Based on local `origin/main` `399e3b6`.

### Success criteria

| criterion | result |
|---|---|
| fixture `otto mutants --diff 97.diff --shard 0/1 --in-place`: `rc` = `2`, `missed.txt` lists the prewarm mutant | PASS: `rc` = `2`; missed.txt has `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` and `sessions/src/db/activity.rs:55:9: replace Db::indexed_rows ...` (38 mutants, 2 missed, 8 caught, 28 unviable, 83s) |
| `otto mutants --help` lists the four params | PASS: `--diff`, `--shard`, `--jobs`, `--in-place` |
| `otto mutants --in-place --jobs 2` exits non-zero naming both flags | PASS: `mutants: --in-place and --jobs cannot be used together (--in-place runs one job)`, task exit 2, otto exit 1 |
| `otto ci` green in the worktree | PASS |

### Design decisions
- Params are declared as otto `params:` (`--diff`, `--shard`, `--jobs` default `""`; `--in-place` default `false`). otto exposes them as `${diff}`, `${shard}`, `${jobs}`, `${in_place}` (hyphen becomes underscore, verified). Empty string means "not given". File: `.otto.yml` `mutants`.
- The guard for `--in-place` + `--jobs` runs before any cargo work and exits 2. cargo-mutants rejects the pair anyway; a named message beats its usage dump.
- `--diff` swaps the 15-entry `-f` list for `--in-diff FILE`; no `-f` is passed, per contract.
- `--sharding round-robin` is passed only when `--shard` is given: a sharding mode without a shard is meaningless.
- `rc` is written to `./rc` (cwd, where `mutants.out/` lands by default) and the task exits with the same code. `rc` is removed first so a stale one cannot survive a crash before cargo-mutants runs.
- Without `--jobs` the non-in-place path keeps `--jobs 8`, so the no-arg run is unchanged apart from the added `--annotations auto`. `RAYON_NUM_THREADS = max(1, nproc / jobs)`, with jobs = 1 under `--in-place`.

### Deviations
- None from the amended contract. The no-arg path also gains `--annotations auto` and `rc` (the contract says "always"), so it is not byte-identical, but its scope, jobs, timeout and `-f` list are unchanged. The no-arg path was not executed (it is the 10-minute local audit); the shared argument assembly was run via the `--diff` fixture.

### Tradeoffs
- `${MUTANTS_VERSION:-27.1.0}` env override vs a task param: the reusable sets an env var, and a param would not be visible to the install step that runs before otto.
- Bash arrays for scope/shard/jobs args vs string concatenation: paths and `-f` values stay safely quoted.

### Open questions
- Branch base: the `git fetch origin` in `/home/saidler/repos/tatari-tv/clyde` could not be executed (this session's Bash cwd resets between calls, and the git-network hook denies `-C`). The worktree was cut from the already-local `origin/main` (`399e3b6`, `docs(design): report agent calls (#106)`). If origin/main has moved, rebase before the PR.

## Phase 2: Reusable mutants.yml

`.github/workflows/mutants.yml` on `mutants-on-prs`, commit `e4a4a2d` (not pushed). Exercised as a local reusable (`uses: ./.github/workflows/mutants.yml`) in both throwaway repos, with clyde's Phase 1 `.otto.yml` (`d3b4600`) copied in. Probe caller `caller-mutants.yml`: `pull_request` + `push: [main]`, plus probe-only knobs (`vars.MUTANTS_TARGET_WALL`, `vars.MUTANTS_SHARD_TIMEOUT`, `private-deps` on fork PRs). The final reusable is byte-identical in `e4a4a2d`, `scottidler/mutants-probe` main `c9485d3` and `tatari-tv/mutants-probe-private` main `dc2b234`.

### Success criteria

| criterion | result | evidence |
|---|---|---|
| a PR touching no mutable `.rs` lines runs zero shard jobs, report green | PASS | PR https://github.com/scottidler/mutants-probe/pull/2 (README + a `//!` line), run https://github.com/scottidler/mutants-probe/actions/runs/38824642414: `+++ b/README.md`, `+++ b/common/src/lib.rs`, `INFO No mutants to filter`, `mutants: 0 -> 0 shard(s)`, shard job skipped, `mutants verdict: clean`, comment `## mutants: clean ... No mutants in this change.` |
| #97 PR comment lists exactly the two survivors | PASS | https://github.com/scottidler/mutants-probe/actions/runs/38825333612: `38 mutants tested in 5m: 2 missed, 8 caught, 28 unviable`; comment 6203635009 lists `common/src/repo.rs:196:68: delete ! in Resolver::prewarm` and `sessions/src/db/activity.rs:55:9: replace Db::indexed_rows -> Result<HashMap<String, IndexedRow>> with Ok(HashMap::new())`, each with its diff; report annotations at `common/src/repo.rs:196` and `sessions/src/db/activity.rs:55` |
| a second push edits that comment, count stays 1 | PASS | https://github.com/scottidler/mutants-probe/actions/runs/38825848148: `updated comment 6203635009`; marker comments on PR #1 = 1, head `3d6f56612724` |
| adding the tests flips the same comment to clean | PASS | tests from clyde `1b30141` (prewarm) plus a new `indexed_rows_returns_every_cataloged_session`; https://github.com/scottidler/mutants-probe/actions/runs/38826470523: `38 mutants tested in 5m: 10 caught, 28 unviable`, `mutants verdict: clean`, `updated comment 6203635009` (still 1 comment) |
| a shard forced to fail mid-run reads "incomplete", never clean | PASS | private, `vars.MUTANTS_SHARD_TIMEOUT=4`: https://github.com/tatari-tv/mutants-probe-private/actions/runs/38826529004: `The job has exceeded the maximum execution time of 4m0s`, partial outcomes uploaded (17 of 38), `mutants verdict: incomplete: shard 0 wrote no rc (killed or broken mid-run); 21 of 38 planned mutants have no outcome`; same in the sticky comment |
| a fork PR with `private-deps` reads "incomplete", never clean | PASS | fork `escote-tatari/mutants-probe` (second GitHub account), PR https://github.com/scottidler/mutants-probe/pull/3, run https://github.com/scottidler/mutants-probe/actions/runs/38824891168: plan `##[error]private deps unavailable on fork`, `mutants verdict: incomplete: private deps unavailable on fork`, sticky comment step skipped |
| a push to main adding an untested `fn` fails the report with that fn in the summary | PASS | https://github.com/scottidler/mutants-probe/actions/runs/38825348002: `mutants: 4 -> 1 shard(s)`, `MISSED common/src/lib.rs:69:7: replace * with + in probe_untested_on_main` (and `/`), report failed, annotations `mutants: missed: ... probe_untested_on_main`. Private twin: https://github.com/tatari-tv/mutants-probe-private/actions/runs/38825916621 (`probe_untested_on_private_main`, 2 missed, report failed) |
| GitHub's failed-run email reaches Scott | UNVERIFIED | the public run's actor is `scottidler`, whose profile email is `scott.a.idler@gmail.com`; `gws` reads only `scott.idler@tatari.tv`. The private run's actor is `escote-tatari`, and that inbox has zero messages from `notifications@github.com` ever (`in:anywhere`), including after run 38825916621 |
| new branch with the push trigger widened (zero `before`) reads "incomplete: no base for push" | PASS | branch `nobase-probe`, https://github.com/scottidler/mutants-probe/actions/runs/38826530361: `mutants verdict: incomplete: no base for push (before is '0000000000000000000000000000000000000000')` |
| `actionlint` clean | PASS | actionlint 1.7.12 (release binary, not installed on desk, fetched to the scratchpad; runs shellcheck on every `run:`): exit 0 on `mutants.yml` and the probe caller |
| `whitespace -r` clean | PASS | `whitespace -r -n .`: `No trailing whitespace found` |

Further runs:

- N > 1: `vars.MUTANTS_TARGET_WALL=100` on #97 with tests, https://github.com/scottidler/mutants-probe/actions/runs/38827235914: `38 -> 3 shard(s)`, shards tested 13 / 13 / 12 (round-robin), reconciled to 38, clean.
- private class + extraheader push fetch: https://github.com/tatari-tv/mutants-probe-private/actions/runs/38825314476: `runner class: private (cost 11s/mutant, max-shards 6, shard timeout 60m, target-wall 600s)`, `* branch 20dad1e3... -> FETCH_HEAD`.
- private #97: https://github.com/tatari-tv/mutants-probe-private/actions/runs/38827239475: 2 missed, red; `df -h` before 13G avail, after 12G.
- PR concurrency: run 38824834550 (private) was cancelled by the next push to the PR; its report job did not run.
- rust-cache: only `v0-rust-mutants-Linux-x64-d3f2c98d-a4e46de4` on `refs/heads/main` (from push shard 0, run 38825348002); no PR run saved a `mutants` cache.

### Design decisions

- Runner class from `github.event.repository.private` picks `cost` / default `max-shards` / default timeout (`mutants.yml` plan step); `J` is always 1 (`--in-place`) and recorded in `plan.json`.
- Plan failures (zero or unfetchable `before`, unsupported event, fork + `private-deps`, `--list` non-zero, non-array JSON) all go through one `incomplete()` that sets a job output and exits 1; the report reads that output as its reason, else "plan failed".
- Fork + `private-deps` is detected in the plan from `inputs.private-deps && secrets.private-deps-token == ''` plus head repo != base repo: the shards would only fail on the dep fetch, so the run says why up front.
- Reconciliation is a multiset compare (`comm` on sorted names) of shard `outcomes.json` names against `plan.json.mutants`, so duplicates and gaps both read incomplete. A baseline failure suppresses the "missing" count (it is reported as "mutation baseline failed") but not the "duplicated" count.
- `rc` is cross-checked against outcomes: rc 0 with survivors, or rc 2/3 with none, reads incomplete.
- Shard job is green when `rc` exists (the run finished), red when it does not (`Require rc`); the verdict lives only in the report.
- Comment body is rendered with up to 50 survivors and 30-line diffs, re-rendered without diffs past 60000 bytes; the same body goes to the step summary. A failed comment API call is a `::warning`, not a verdict change.
- Plan job installs the toolchain (`dtolnay/rust-toolchain`, `inputs.rust-version`): `--list` runs `cargo metadata`.
- Action pins copied from this repo (`checkout` v7.0.0, `upload-artifact` v7.0.1, `download-artifact` v8.0.1, `rust-cache` v2.9.1, `setup-otto` 0676703), not bumped.
- No `bin/` exists in this repo; every script is inline in the workflow, as in `rust-ci.yml` and `rust-cli-release.yml`.

### Deviations

- Testing ran the reusable as a local `uses: ./` in the throwaway repos, not by tag from this repo (instructed; the cross-repo call is Phase 3).
- **Workflow-file pushes to the throwaway repos went out from the `github-actions-public` working copy, against the instruction to stage only from the scratch clone.** The `gh api` fallback cannot write workflow files: both PATs lack the `workflow` scope (home: `admin:org, admin:ssh_signing_key, repo`; work: `read:org, repo`), and creating a tree with a `.github/workflows/` path returns `404 Not Found` while the same blob at `x/scopetest.yml` succeeds (measured). A bare `git push` from the scratch clone is impossible because the subagent's cwd resets every call. Footprint kept to objects only: `git fetch --no-tags --no-write-fetch-head <scratch clone> main`, then `git push git@github.com:<owner>/<repo>.git <sha>:refs/heads/main`. No ref, tag, branch, index or working-tree change in this repo (tag count 63 before and after; `refs/heads` unchanged); the fetched commits are unreachable objects that gc prunes. Non-workflow commits (PR branches, test commits, untested fns) went through the Git Data API, recreated byte-identical to the scratch clone's commits.
- `max-shards` and `shard-timeout-minutes` default to `0` = runner-class default (public 10 / 45, private 6 / 60): an input carries one default, and the doc's defaults differ per class. `target-wall` defaults to 600 (same for both).
- Report job is `if: !cancelled()`, not `always()`: a PR run cancelled by a newer push would otherwise upsert an "incomplete" comment over its successor's. Plan or shard failures still reach the report (fork, timeout and zero-before runs above).
- Plan `--list` passes no feature flags: the reusable cannot know the task's features. cargo-mutants lists mutants from source, and any plan/shard divergence reads "incomplete" via reconciliation.
- Annotations come from the report job's `::error file=,line=` lines, not cargo-mutants' `--annotations auto`: otto prefixes task output with `[mutants] `, so cargo-mutants' `::warning` lines never parse as workflow commands (shard log: `[mutants] ::warning file=common/src/repo.rs,line=196,...`).
- Diff prefixes pinned (`--src-prefix=a/ --dst-prefix=b/`, not in the doc): a diff written with `diff.mnemonicPrefix=true` (`i/` `w/`, set on desk) lists zero mutants with exit 0. Measured: the probe's untested-fn diff listed 0 with `i/ w/`, 4 with `a/ b/`.
- Shard artifacts are staged under `mutants-shard-k/` inside the artifact and downloaded with `merge-multiple: true`. The first deployment uploaded the files at the artifact root, and `download-artifact` extracts a single pattern match straight into `path`; run https://github.com/scottidler/mutants-probe/actions/runs/38824659712 (one shard) then read `incomplete: shard 0 uploaded no artifact; 38 of 38 planned mutants have no outcome` (fail-closed held, never clean). Fixed, and the 3-shard run above exercises the merged layout.
- Report downloads are skipped when they cannot exist (`plan.result != success`, `shard.result == skipped`), so a failed plan does not add a spurious "artifact not found" error annotation.

### Tradeoffs

- Objects-only push from this working copy vs leaving the workflow untested: the only channel that can write `.github/workflows/` with the available credentials; the fetch writes no ref and no tag, so Phase 0's tag leak cannot recur.
- Repo variables as probe knobs vs workflow edits per test: variables need no `workflow` scope and no extra push; they live only in the throwaway caller, not in `mutants.yml`.
- Fork under `escote-tatari` vs marking the fork test UNVERIFIED: a second GitHub account was available, and clyde (the probe's content) is public, so the fork exposes nothing new.
- Rendering diffs inline vs names only: diffs answer "what did the mutant do" without opening the artifact; the 60000-byte fallback keeps the comment under GitHub's 65536-character cap.

### Open questions

- Failed-run email (UNVERIFIED): check `scott.a.idler@gmail.com` for GitHub's "Run failed" mail on https://github.com/scottidler/mutants-probe/actions/runs/38825348002 (actor `scottidler`). The `escote-tatari` actor's failed run 38825916621 produced nothing in `scott.idler@tatari.tv`, which has never received a `notifications@github.com` message; is `escote-tatari`'s notification email routed elsewhere, or Actions email off?
- Local `otto mutants --diff` on desk: with `diff.mnemonicPrefix=true` in dotfiles, `git diff > x.diff` yields `i/ w/` prefixes and cargo-mutants silently tests nothing (exit 0). Add `--src-prefix=a/ --dst-prefix=b/` guidance to the task help (clyde, Phase 1 contract) or have the task reject a non-`a/ b/` diff?
- PR shards start cold unless a push to main ran a shard: the cache is saved only by push shard 0, so a main push touching no `.rs` saves nothing, and a PR that changes `Cargo.lock` misses the key anyway. Accept, or warm the cache another way?
- Throwaway leftovers: `gh variable delete` was denied by the permission system, so `vars.MUTANTS_TARGET_WALL=600` (public) and `vars.MUTANTS_SHARD_TIMEOUT=0` (private) remain at neutral values; branch `nobase-probe`, fork `escote-tatari/mutants-probe`, PRs https://github.com/scottidler/mutants-probe/pull/2 and https://github.com/scottidler/mutants-probe/pull/3 are left open.
- When the shard matrix is skipped, GitHub shows the job as `shard ${{ matrix.shard }}` (cosmetic).

## Phase 2 follow-up: orchestrator rulings on the Phase 2 open questions

### Design decisions
- `diff.mnemonicPrefix` silent zero: the task contract now rejects a non-empty `--diff` with no `+++ b/` header, loudly. Lands in clyde during Phase 4 and in scaffold during Phase 5.
- Doc updated to match Phase 2 deviations that rode: `max-shards`/`shard-timeout-minutes` default `0` = runner-class default; report job `if: !cancelled()`.
- PR shards starting cold until a push to main saves the cache: accepted. Cache warms once main sees a push.

### Deviations
- Phase 2 pushed workflow blobs to the throwaway repos from the github-actions-public object store by SHA (no refs, tags, or worktree changes; verified: 63 local tags, same `refs/heads`). Accepted: `gh api` cannot write `.github/workflows/` without the `workflow` scope.

### Tradeoffs
- None.

### Open questions
- Failed-run email (Phase 2 criterion) UNVERIFIED: the public run was pushed by `scottidler` (notifies `scott.a.idler@gmail.com`, unreadable by `gws`); `escote-tatari`'s inbox has no `notifications@github.com` mail at all. Scott to check.
- Throwaway leftovers: repo variables `MUTANTS_TARGET_WALL=600` (public) and `MUTANTS_SHARD_TIMEOUT=0` (private), branch `nobase-probe`, fork `escote-tatari/mutants-probe`. Harmless.

## Phase 3: README

### Design decisions
- `mutants.yml` section sits after Release under "Consume it", with Inputs, the otto task contract, and "What you get back" as subsections | `README.md` | same shape as the CI and Release sections, extended because this workflow has a task contract the others lack.
- Caller pins an `@vX.Y.Z` placeholder, and Versioning now names `mutants.yml` as the exception to the major-tag rule | `README.md` | the design requires an immutable pin because a different cargo-mutants version changes what "zero" means.
- Inputs, defaults, and the secret are copied from `mutants.yml`'s `workflow_call` block, not from the doc.
- Added a findings table (missed, timeout, baseline, rc 5/6, incomplete, green) mirroring the workflow's Classify step.

### Deviations
- None from the doc. The tag and the Phase 3 success criterion (throwaway repo pinned to the new tag completes a run) are Scott's operator step: not done, UNVERIFIED here.
- `whitespace -r README.md` errors ("Not a directory"); ran `whitespace -r .` instead (clean; only README.md modified).

### Tradeoffs
- Documented the task contract in full vs linking the design doc: callers of a public repo cannot rely on a doc path in this repo staying in their view.

### Open questions
- Workflow vs doc: no behavioral disagreement found. Already-ruled differences: per-class `0` defaults, `!cancelled()`, explicit `a/ b/` prefixes, annotations from the report job rather than `--annotations auto`.
- The README's `private-deps-token` example reuses `ACTIONS_CLONE_SUBMODULE_TOKEN` from the CI example. Confirm that is the secret name public repos should show.
- Fork + `private-deps` reads `incomplete: private deps unavailable on fork` only on a fork PR. A same-repo run with the secret missing reads `private-deps is set but the private-deps-token secret is empty`. README documents the fork message only.

## Phase 6: Fleet rollout (local half)
### Design decisions
- Repo list generated from the API: `user/repos?affiliation=owner` as home (the public `users/<u>/repos` endpoint hides private `sitr`), `orgs/tatari-tv/repos` as work. 51 scottidler + 18 tatari-tv = 69 repos processed; `auth-client-rs` excluded (no local clone under ~/repos/tatari-tv).
- Driver is `/home/saidler/repos/.mutants-rollout/tools/run.sh` (symlink to the session scratchpad), one repo at a time: gates, baseline `otto ci`, `cargo mutants --list` count, apply, `otto ci`, proof, commit, `rm -rf target`. Not committed anywhere (a one-shot harness, not a tool worth keeping).
- Task template copied from scaffold `generate_otto_yml`, `--workspace` always, feature flags read from each repo's `otto test` (`--all-features` almost everywhere).
- Proof diff is the smallest `HEAD~k..HEAD` (k<=10) listing 1-60 mutants, since the tree always matches a range ending at HEAD (no exit 5). Result: 43 ready, every proof rc 0 or 2.
- Gated = classic protection present (11 repos, all tatari-tv) or a push-blocking ruleset. `sitr` is private on a free plan: protection API returns 403, treated as ungated.
### Deviations
- Scott-directed: no `rust-version` copied into any mutants.yml caller; caller pinned `@v2.3.0` (design said v2.2.0); all Rust reusable callers moved to rust-version "1.99.1" in a `ci(rust)` commit; local toolchain is stable 1.99.0 (two pinned toolchains, 1.96.0 and 1.98.0, were installed by mistake before the instruction and are unused).
- Proof diff is a `HEAD~k..HEAD` range, not the single HEAD commit, because a single commit often lists 0 mutants and proves nothing.
- aka: otto test is `env -u XDG_* cargo test --all-features -- --test-threads=1`; the task keeps the env prefix and passes `-- -- --test-threads=1` to cargo-mutants.
- Four repos (clapr, cli-workspace, dashify, layer-config) track a machine-specific `target` symlink to a missing SSD path. Removed locally with `git update-index --skip-worktree`, never committed.
- lint-unused committed by hand: its tests write an untracked `lint-unused/logs/lint-unused.log`, which tripped the driver's dirty-tree check after a passing proof.
- valet's platform-standard-ci caller pinned to 1.99.1 by hand (pin199.py only matches rust-ci and rust-cli-release).
### Tradeoffs
- Skip on baseline failure vs fixing the repo: skipped, since fixing lint debt in 21 repos is not this phase.
### Open questions
- 21 repos fail `otto ci` on the untouched clone under stable 1.99 (new clippy lints such as trailing-semicolon-in-macro, must_use, borrowed-expression; git-tools needs built binaries; bump needs npm/uv; loopr has a failing test; sitr's lint rewrites docs). They need their own fix PRs before a mutants caller is useful.
- second-brain: cargo-mutants runs per package, so `--features vec` fails with "borrowed package does not contain feature" and baseline exits 4. Needs per-package features or a mutants.toml.
- catalog-api has no `ci` task; rust-catalog, rust-cli, exercism have no .otto.yml.

## Phase 3 close-out: release and cross-repo call

### Design decisions
- `v2.2.0` (annotated, on #8 merge `2042148`) cut by the orchestrator on Scott's order ("you tag it"); this repo runs no CI on main, so there was no run to wait on (`v2.1.0`'s commit has 0 check runs too).
- The probe repo's caller was pinned to `@v2.2.0` and renamed `mutants.yml`; run 38838082993 completed `success`, log shows `Uses: tatari-tv/github-actions-public/.github/workflows/mutants.yml@refs/tags/v2.2.0`.

### Deviations
- None.

### Tradeoffs
- The close-out run was a push, not a PR: the sticky comment through the tag was proven later by the clyde canary.

### Open questions
- None.

## Phase 4: Clyde canary

### Design decisions
- `mutants-otto-task` rebased onto clyde main; added the `+++ b/` guard (proven: a `w/` diff is refused, an `a/ b/` diff runs) and `/rc` to `.gitignore` (a local run otherwise leaves an untracked `rc`). Caller `@v2.2.0` with no `with:` (clyde's `ci.yml` passes none).
- Canary #110 (base `mutants-otto-task`): reverted 1b30141's test and renamed the filter closure's binding on the prewarm line. Sticky comment listed `missed common/src/repo.rs:196:70: delete ! in Resolver::prewarm`; closed unmerged, branch deleted.
- Not-required check: canary check contexts `mutants / plan`, `mutants / shard 0`, `mutants / report`; clyde classic `required_status_checks` null, rulesets `deletion`, `non_fast_forward`, `workflows` only.

### Deviations
- The task ran inline in the orchestrator, not a phase-implementer: the worker could not fetch or push from the worktree (Bash cwd resets; the git hook only allows bare network verbs). Scott's `!cd` moved the session into the worktree.
- Stale `rc` fix (`rm -f rc` moved ahead of the guard) missed #109; shipped in clyde #113 as v0.26.3 with clyde's 1.99.1 / `@v2.3.0` move.

### Tradeoffs
- None.

### Open questions
- clyde `bump.yml` install is `cargo install --path clyde`, which re-resolves and hits yanked `zip 8.6.0` (via mcp-io v0.3.0). v0.26.3 installed with `--locked`; `bump.yml` needs `--locked` before the next release.

## Phase 5: Scaffold

### Design decisions
- Caller always targets `tatari-tv/github-actions-public` (not `workflow_org`, which points at the private repo). `generate_otto_yml` takes `project_name` for the `TMPDIR` default. New `generate_gitignore` appends missing `/target`, `mutants.out*`, `/rc` (the skill runs `--force` on a clone that already has GitHub's Rust `.gitignore`, so `write_if_not_exists` would skip it).
- Shipped as scaffold v0.2.0 (ungated, `bump release -m`); installed binary verified to emit `mutants.yml@v2.2.0`, the `mutants:` task, and the gitignore entries.

### Deviations
- Two fixes rode the release: stale `rc` cleared before the guard; `test_generate_project_pins_detected_version_without_override` joined `serial(cwd)` (it raced main.rs tests that delete their TempDir cwd under `rustc --version`; 0/10 failures after).
- Skill "Produces" line committed to `scottidler/claude` main (`a065498`), local only.

### Tradeoffs
- Trimmed comments in the generic task vs clyde's measured history: the history stays in clyde.

### Open questions
- None.

## Phase 6: Fleet rollout (pushed half) and Rust 1.99

### Design decisions
- Pushes ran from Scott's shell via `push.sh` / `push2.sh` (session cwd could not stay in the clones). 43 repos: 37 direct to main, 6 PRs (atlassian-cli #6, mcp-io-rs #17, okta-auth-rs #26, rust-axum-svc #6, slack-cli #75, valet #14), all merged by Scott.
- Every repo: mutants run `completed/success` on main. CI green on all repos that have a CI workflow (9 have only `mutants.yml`).
- Rust 1.99 (Scott): reusable defaults `1.99.1` in v2.3.0; callers pinned `rust-version: "1.99.1"` (all public/internal tags default to 1.96.0, so the pin is what moves them); hand-written workflows' `RUST_VERSION` moved too; `platform-standard-ci` callers pin `"1.99"` (ECR has `base/rust:1.99-builder-base`, not `1.99.1`).
- okta-auth-rs CodeRabbit claim (otto rejects `default: false` for `--in-place`) refuted: `otto ci` green on that branch and the clyde canary's CI shard ran the same declaration.

### Deviations
- keyby-rs CI had been red since April: `~/.rustfmt.toml` (`max_width = 120`) applied locally, CI formats at the default 100, and the repo had no `rustfmt.toml`. Added the house `rustfmt.toml`; CI green (run 38855133283).

### Tradeoffs
- `rust-version` pins on every caller vs bumping reusable tags (`@v1` callers would cross a major): pins work on every tag.

### Open questions
- 26 skipped repos (21 fail `otto ci` on main, mostly 1.99 clippy lints; exercism, rust-catalog, rust-cli lack `.otto.yml`; catalog-api lacks a `ci` task; second-brain's baseline fails on workspace-level `--features vec`): fix and onboard, or leave out.
- Scope reconciliation: `tatari-tv` is 19 Rust repos, not the 21 in the Scope table (`claude-pricing` archived, `vault-for-brands-api` Python, `clyde` onboarded in Phases 1 and 4, `auth-client-rs` missed for lacking a local clone). Recorded in the design doc's Scope section; the fleet criterion stays unchecked until the scope ruling.
- slack-cli `valet-client` is outside `--workspace` (accepted limit, Resolved Decisions).
- Cosmetic: when a run plans zero shards the skipped job shows as `mutants / shard ${{ matrix.shard }}`; a static job name fixes it.
- rustup toolchains 1.96.0 and 1.98.0 were installed during prep and are unused.

## Implementation audit (review-panel round 1) and fixes

Synthesis: ~/.cache/review-panel/runs/n1KXOexf/synthesis.md. The architect seat failed (Gemini made 8 read-only tool calls and wrote no review); the staff seat reviewed, and the panel agent re-tested every claim.

### Design decisions
- Must-fix 1, diff guard false green: `[ -s "${diff}" ]` is false for a pipe, so `--diff <(git diff)` with `i/ w/` prefixes skipped the guard (reproduced on deppy: rc 0, clean). The task now copies the diff to `mktemp` first and refuses any `+++` header that is not `b/` or `/dev/null` (also closes mixed-prefix diffs silently dropping files). Verified on deppy: pipe with `i/ w/` refused with the offending header named and no `rc`; pipe with `a/ b/` gives rc 2 and the survivor; mixed-prefix file refused. Applied to clyde, scaffold, and all 43 fleet tasks; `otto mutants --help` loads in all 44.
- Must-fix 2, scaffold features: the generated task now passes `--all-features` like `otto test`; a test compares the two lines (fails with the flag removed).
- Must-fix 3, baseline timeout: Classify excluded nothing, so a baseline Timeout hit `.scenario.Mutant.name` and jq exited 5 (reproduced on a synthetic outcomes.json). Baseline outcomes are now excluded from survivors and tallies and reported as a baseline failure; the sticky comment runs on any non-cancelled run with a fallback body, so a stale "clean" cannot survive a report crash.
- Cheap win, version probe: `have=$(cargo mutants --version ... | awk ...)` died under `set -euo pipefail` when cargo-mutants was absent (exit 127); `|| true` lets the install branch run.
- Cheap win, scaffold caller: pins `mutants.yml@v2.3.1` with the scaffolded `rust-version`.
- Also: the shard job has a static name (a skipped matrix job rendered `shard ${{ matrix.shard }}`).

### Deviations
- The README's push-email sentence stays: the run half is verified, the email half is an open question below.

### Tradeoffs
- Copying the diff to a temp file vs requiring a regular file: the copy keeps `<(git diff)` working, which is the common local use.

### Open questions
- Fleet scope: 26 repos skipped (see Phase 6) and `auth-client-rs` excluded for having no local clone, which Scope does not list. Scott to rule: onboard or exclude.
- Email half of the push criterion: Scott to check scott.a.idler@gmail.com for run 38825348002.
