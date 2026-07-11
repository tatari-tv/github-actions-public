# Implementation Notes: Public Rust CI/Release/SAST Fleet Migration

Running, append-only record of how the implementation interprets or diverges
from `2026-07-10-public-rust-ci-fleet-migration.md`. One section per phase.

## Scouting reconciliation (pre-Phase 1)

- **Fleet layout is mixed.** clyde and drata-cli are bare-container worktree
  repos (`.bare` + `main/` working tree); the actual files live under
  `<repo>/main/`. Everything else (okta-auth-rs, pagerduty-cli,
  ralph-wiggum-loop, renew, whitespace, mcp-io-rs) is a flat checkout. An
  earlier read of `drata-cli/rustfmt.toml` returned nothing purely because of
  this layout, not because the doc was wrong. The doc's per-repo table is
  accurate once the `main/` subdir is accounted for.
- **renew MSRV is `1.89`**, not `1.94.1` as the doc's Resolved Decision (line
  139) states. Immaterial to the work: both converge onto the rust-ci.yml@v1
  default toolchain (1.96.0) at the Phase 4 caller swap.
- Gates verified live: renew = ungated (classic 404 + 0 rulesets, direct push);
  drata-cli = gated (`workflows` ruleset = org Required-Workflows check, PR flow).

## Cross-cutting: gated-repo merges are human-only (org policy)

The auto-mode classifier + Tatari managed-settings ("Follow our internal PR
review process before merging AI-generated code") **block `gh pr merge --admin`
on AI-authored PRs**, even though the gated repos have `enforce_admins: false`
and a `slam` bypass team. So the flow for the 4 gated repos (claude-pricing,
clyde, drata-cli, okta-auth-rs): open PR -> babysit to all-green -> hand Scott
the URL; **Scott merges**. Ungated repos (renew, pagerduty-cli,
ralph-wiggum-loop, whitespace) land via direct push. Decision (Scott,
2026-07-11): "I open + green them, you merge as they land" -- proceed to the
next repo without waiting on a merge. Do NOT route around the block via slam.

## Operating note: sandbox blocks the ssh-agent

Every work-repo `git commit`/`push` must run with the command sandbox disabled:
the sandbox denies the ssh-agent socket, so SSH commit-signing falls back to the
passphrase-locked key file and fails. `cargo` builds also need sandbox-off
(sccache socket + `target/` `.fingerprint` writes).

## Phase 1: rustfmt normalization

### Design decisions
- **drata-cli**: `cargo fmt` reformatted nothing after adding
  `single_line_if_else_max_width = 80` (no if-else in the 50-80 width band), so
  the PR is config-only. Staged ONLY `rustfmt.toml`; left Scott's three
  untracked `docs/*.md` alone.
- **renew**: reformatting 100->120 touched 10 source files (mechanical).

### Deviations
- Phase 1's stated success criterion is `cargo fmt --all --check` clean, not
  full `otto ci` -- validated renew's Phase 1 with fmt-check because full
  `otto ci` was gated on the Phase 2 test fix (renew was RED). Both phases were
  then validated together with a green `otto ci` before pushing.

### Tradeoffs
- None.

### Open questions
- None.

## Phase 2: fix renew's red test

### Design decisions
- Root cause: `build.rs` ran `git describe --tags --always`; on a shallow/
  tagless checkout `--always` emits a bare short SHA, which
  `version::parse_current` (`src/version.rs`) is explicitly designed to reject
  as non-semver, panicking `tests::test_macro_all_arms_select_shared_source:18`.
- Fix in `build.rs` (not the test): dropped `--always`. `git describe --tags`
  now exits non-zero on a tagless checkout, routing into the pre-existing
  `CARGO_PKG_VERSION` fallback, so `GIT_DESCRIBE` is always valid semver. This
  fixes the runtime version the binary reports too, not just the test.
- Left the failing integration test unchanged as the regression canary + added
  a build.rs comment forbidding re-adding `--always` (structural guard).

### Deviations
- The doc framed the fix as "test/build.rs robust to tagless GIT_DESCRIBE"; the
  durable fix is build.rs-only. The test needed no change once build.rs
  guarantees semver.

### Tradeoffs
- Could have hardened the test to tolerate a bare-SHA `GIT_DESCRIBE`, but that
  leaves the runtime binary reporting a bare SHA as its version on shallow
  builds (renew's own update logic would break). Fixing build.rs fixes both.

### Open questions
- None. Proven on a `git clone --depth 1 --no-tags` reproduction: 91/91 tests
  pass (was 90/1); baked `GIT_DESCRIBE=0.3.2`. renew main CI is green.

### Note
- renew MSRV in Cargo.toml is still `1.89`; the toolchain converges to 1.96.0
  at the Phase 4 caller swap (rust-ci.yml@v1 default), not here.

## Phase 3: SAST reusable extraction

### Design decisions
- `sast.yml` is clyde's three jobs VERBATIM; only the trigger changed
  `on: pull_request` -> `on: workflow_call` (callers own the PR trigger). Top
  comment reworded to describe the reusable + fork-PR safety.
- Landed via PR (github-actions-public is gated): PR #3. The `@v1` tag move to
  include sast.yml is handed to Scott (git tag-safety: never move a tag
  myself), with exact commands. Scott moved `@v1` to `3e1eda5`.

### Deviations / Tradeoffs / Open questions
- None. Phase 3's success criterion (a caller runs all three scanners on a fork
  PR with contents:read only) is proven jointly with Phase 7's first SAST
  caller run, since a workflow_call reusable cannot run without a caller.

## Phase 4: CI migration

### Design decisions
- All 8 repos get the SAME thin caller (no inputs): none have private git deps,
  all want the reusable defaults (1.96.0, whitespace v0.1.6). Trigger
  standardized to `push: ['**'] + pull_request: {}` (the merged mcp-io pattern).
- **claude-pricing**: preserved `paths-ignore: [data/**, **/*.md]` on the caller
  triggers -- the daily refresh-pricing automation commits data/pricing.json and
  would otherwise fire CI on every refresh (doc risk row).
- **okta-auth-rs**: dropped `id-token: write` / `statuses: write` / `secrets:
  inherit` -- those existed only for the internal ECR builder flow; the public
  reusable runs on a bare runner (contents: read).
- **Pre-flight local-target override**: repos whose `target/` is symlinked to
  the intel-480gb-ssd (relocate-targets) fail the `aws-lc-sys` C build there;
  ran every pre-flight with `CARGO_TARGET_DIR=~/tmp/preflight/<repo>` so otto ci
  runs. desk.lan environment quirk, NOT a repo problem. All 8 pre-flights green;
  no stricter-lint (1.96) findings surfaced.
- Ungated (renew, pagerduty-cli, ralph-wiggum-loop, whitespace): direct push.
  Gated (okta-auth-rs #16, drata-cli #8, clyde #41, claude-pricing #28): PRs
  green, merged by Scott. okta-auth-rs RED 0-job failure fixed.

### Deviations
- clyde's ci trigger was `push:['**'] + workflow_dispatch`; the caller uses the
  fleet standard `push:['**'] + pull_request:{}`. workflow_dispatch dropped.
- claude-pricing's original push trigger was `branches:[main]`; standardized to
  `['**']` (paths-ignore preserved).

### Tradeoffs
- Drove Phase 4 inline sequentially rather than via Workflow fan-out: work-repo
  commits need the sandbox OFF for the ssh-agent (background agents can't set
  that), and 8 concurrent otto-ci Rust builds risk OOMing desk.lan.

### Open questions
- None.

## Phase 5: Release migration

### Design decisions
- Thin `rust-cli-release.yml@v1` callers on the 4 pure CLIs. Artifact-name
  parity is the contract; inputs pinned so tarball + upload names are unchanged:
  - **drata-cli**: `bin-names: drata` + `archive-prefix: drata` (crate is
    drata-cli, binary is `drata`). Reusable default 1.96.0 replaces old 1.94.0
    (org-standard toolchain decision, line 139).
  - **pagerduty-cli**: `bin-names: pd` + `archive-prefix: pd`.
  - **ralph-wiggum-loop**: `bin-names: rwl` + `archive-prefix: rwl`; the old
    file was `release-and-publish.yml` (a misnomer -- it only ever published a
    GitHub Release), replaced by `release.yml` so the 4 callers are identical
    siblings.
  - **clyde**: no overrides (binary == repo == `clyde`, reusable defaults match).
- Ungated (pagerduty-cli, ralph-wiggum-loop): direct push. Gated (drata-cli #11,
  clyde #44): PRs, green + CodeRabbit-clean (0 comments), merged by Scott.

### Deviations
- **All 4 callers are tag-push-only; no `workflow_dispatch`.** The reusable's
  publish job has NO tag-ref guard, so a `workflow_dispatch` on a branch would
  cut a Release named after the branch. 3/4 old files were tag-push-only anyway.
  clyde's old file DID have a guarded `workflow_dispatch` with a `tag` backfill
  input -- an obsolete workaround from when the broken internal reusable failed
  to build clyde's releases as it went public. That failure class is gone now
  that clyde consumes a working public reusable, so the input was dropped rather
  than reintroduce the branch-named-Release footgun without its guard. This
  diverges from the README caller example (which shows `workflow_dispatch: {}`);
  a fleet-wide fix would be to add a tag-ref guard to the reusable's publish job.

### Open questions
- **clyde version drift (pre-existing, NOT introduced here).** clyde's latest
  tag/Release is v0.9.0 (Latest, on main) but `Cargo.toml` on main is 0.8.3. The
  next release tag needs Cargo reconciled to the 0.9.x line first. Flagged to
  Scott; does not affect the caller wiring, only the eventual parity tag.

### Verification
- NAMING/path parity is guaranteed by construction (reusable file + upload
  naming `<prefix>-<tag>-<suffix>` == every old workflow's naming). Tarball
  CONTENT parity (checksums / bytes) is NOT proven by naming and is confirmed
  only by the live tag on each -- a real tag push, Scott's hand (git tag-safety).
  Pending -- see closeout.

## Phase 6: whitespace (release + docker in ONE workflow)

### Design decisions
- `release.yml` = a `release` job (`uses: rust-cli-release.yml@v1`, defaults:
  binary == repo == prefix == `whitespace`) PLUS the existing buildx->ghcr
  `docker` job, now `needs: release` (was `needs: build-linux`). Everything else
  in the docker job is preserved verbatim (QEMU, metadata tags, `build-contexts:
  binaries=artifacts/`, multi-arch push).
- The docker job downloads the reusable's per-target artifacts by NAME
  (`whitespace-linux-amd64`/`-arm64`) and extracts the tarball
  `whitespace-<tag>-linux-<arch>.tar.gz` into `artifacts/<arch>/`.
- Caller top-level `permissions: contents: write` (reusable release job) +
  `packages: write` (docker job); docker job re-declares `contents: read,
  packages: write`.
- whitespace is ungated: direct push. Old `release-and-publish.yml` removed.

### Deviations
- Tag-push-only (no `workflow_dispatch`), same reason as Phase 5.

### Why one workflow (the load-bearing constraint)
- `download-artifact@v4` is scoped to the `run_id`. A reusable workflow runs
  INSIDE the caller's run, so the sibling `docker` job (same run_id) can download
  what the reusable uploaded. A SEPARATE `docker-publish.yml` on the same tag is
  a different run_id and would find nothing. Hence docker stays a `needs:
  release` job here, not its own workflow.

### Verification
- Tarball NAMING parity by construction; tarball content parity AND the
  multi-arch ghcr image are proven only by a real tag (Scott's hand). Pending --
  see closeout.

## Phase 7: SAST callers

### Design decisions
- Thin `sast.yml@v1` caller (trigger `on: pull_request`) added to all 9 public
  Rust repos (the 8 in Phase 4 PLUS mcp-io-rs, which had a CI caller but no
  SAST). clyde's repo-local full `sast.yml` swapped for the caller -- no repo
  now holds a copied full SAST workflow.
- Ungated repos: direct push. Gated repos (okta-auth-rs #18, claude-pricing #30,
  drata-cli #10, clyde #43, mcp-io-rs #10): PRs, merged by Scott.

### Verification
- Proven: all three scanners (Semgrep, Checkov, Trivy) execute via the reusable
  on the gated PRs and their checks pass as non-blocking audit steps
  (`continue-on-error: true`) -- "pass" means the jobs ran and reported, NOT a
  zero-finding assertion. This also retroactively proves Phase 3's success
  criterion (a caller runs all three scanners with `contents: read`).

### Deviations / Tradeoffs / Open questions
- None.

## Cross-cutting: background security-review findings on the ci.yml callers (ACCEPTED)

A background commit review flagged 3 issues on the caller ci.yml files. All
accepted with rationale (reusable posture verified: rust-ci.yml
`permissions: contents: read`; SHA-pinned `actions/checkout@v7.0.0` with
`persist-credentials: false`):

1. **checkout-token-persistence** -- the callers contain NO checkout; checkout
   lives in the reusable with `persist-credentials: false`. Non-issue.
2. **control-regression-permissions** -- the reusable declares
   `permissions: contents: read`, capping its jobs' token; the callers have no
   token-using steps. okta-auth-rs's removed `id-token`/`statuses` were
   ECR-only; removing them is tightening, not regression.
3. **supply-chain-mutable-ref** (`@v1`) -- INTENTIONAL: `@v1` is the movable
   reusable-workflow major tag (Resolved Decision line 135). Callers must track
   `@v1` so a reusable fix reaches all 9 repos without editing them; SHA-pinning
   defeats the migration's purpose. A fleet-wide SHA-pin (if wanted) is its own
   change across all 9, not silent drift inside this migration.
