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
