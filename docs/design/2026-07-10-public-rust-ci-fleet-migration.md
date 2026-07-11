# Design Document: Public Rust CI/Release/SAST Fleet Migration

**Author:** Scott Idler
**Date:** 2026-07-10
**Status:** In Review
**Review Passes Completed:** 5/5

## Summary

Converge every public `tatari-tv` Rust repo onto the shared reusable workflows in `tatari-tv/github-actions-public`: `rust-ci.yml@v1`, `rust-cli-release.yml@v1`, and a new `sast.yml@v1` extracted from clyde. Kill three problems at once: public repos can't reach the internal `github-actions` reusable workflows, per-repo inline CI has drifted, and SAST lives on exactly one repo.

## Problem Statement

### Background
- `tatari-tv/github-actions` is **internal**. GitHub blocks a public repo from calling a reusable workflow / action in a private or internal repo, so a public caller fails at startup (0-job run named `.github/workflows/ci.yml`).
- We built `tatari-tv/github-actions-public` (public twin) with `rust-ci.yml@v1` + `rust-cli-release.yml@v1` and proved it end-to-end on `mcp-io-rs` (gated repo, PR #6 merged green, org "Required Workflows Security" check passed alongside the public reusable).
- Three CI shapes exist across the 8 remaining repos: 6 run identical inline cargo CI, clyde already runs the target `otto ci` pattern (it's the in-house precedent the reusables were harvested from), and okta-auth-rs still calls the internal `platform-standard-ci.yml@v5`.
- SAST (Semgrep + Checkov + Trivy, non-blocking audit) exists only on clyde as a repo-local `sast.yml`, part of the `fleet-sast` rollout that is not yet org-registered (ruleset #126206).

### Problem
The public Rust fleet has no shared CI contract. Each repo re-declares its own pipeline, so:
- **okta-auth-rs is RED** — public repo calling the internal `@v5` reusable → 0-job startup failure (run `29138049643`, 2026-07-11). Last green was 2026-07-06; the green is stale.
- **renew is RED** — a real test bug, not CI shape (see below).
- CI definitions drift repo-to-repo (the class `rust-ci.yml` exists to kill).
- rustfmt silently disagrees local (Scott's global `~/.rustfmt.toml`, 120) vs CI (default 100) unless a repo commits `rustfmt.toml`.
- SAST covers 1 of 9 repos.

### Goals
- Every public Rust repo runs CI via `rust-ci.yml@v1`.
- Every public Rust **CLI** releases via `rust-cli-release.yml@v1`, with **byte-identical artifact names** to today.
- Every public Rust repo runs SAST via a new shared `sast.yml@v1` (thin caller, not a copied file).
- Every public Rust repo commits `rustfmt.toml` (org standard: 120 + single_line_if_else 80).
- okta-auth-rs and renew go green.
- No repo loses a capability it has today (coverage, pages, refresh jobs, ghcr image).

### Non-Goals
- Internal/private repos (keep internal `github-actions`).
- Making SAST a hard merge gate (stays audit/non-blocking).
- Registering the SAST reusable as the org-required workflow (that is the `fleet-sast` project's org-ruleset change, tracked separately).
- Adding release to the libraries (claude-pricing, okta-auth-rs, renew ship no binary).
- Non-Rust public repos.

## Proposed Solution

### Overview
Three reusable workflows in `github-actions-public`; each repo reduced to thin callers:
- `rust-ci.yml@v1` — `otto ci` (exists).
- `rust-cli-release.yml@v1` — tag-driven binary release (exists).
- `sast.yml@v1` — Semgrep + Checkov + Trivy audit (NEW; extract clyde's verbatim).

Plus: every repo commits `rustfmt.toml`.

### Architecture
- **Extract, don't copy.** clyde's `sast.yml` becomes a reusable `github-actions-public/.github/workflows/sast.yml` (`on: workflow_call`); each repo gets a ~6-line caller. Nine copies of a 200-line file is the drift anti-pattern this effort kills.
- **Caller owns the trigger.** ci on push/PR, release on tag, sast on PR.
- **bin-name overrides are mandatory** where bin ≠ crate name (see below), or the release renames artifacts and breaks consumers.
- **Repo-specific workflows are preserved**, never migrated away: claude-pricing `pages.yml`+`refresh-pricing.yml`; clyde `pages.yml`+`refresh-pricing.yml`; whitespace docker→ghcr publish.

### Per-repo facts (verified against live main)

| repo | bin(s) | kind | CI today | release today | rustfmt.toml | status | gated |
|---|---|---|---|---|---|---|---|
| claude-pricing | *(lib)* | lib | inline | none | 120+80 OK | green | gated |
| clyde | `clyde` | 1 bin | **already otto ci** | vendored (done) | 120+80 OK | green | gated |
| drata-cli | **`drata`** | 1 bin | inline | release.yml | **120 only, missing 80** | green | gated |
| okta-auth-rs | *(lib)* | lib | **internal @v5** | none | 120+80 OK | **RED (0-job)** | gated |
| pagerduty-cli | **`pd`** | 1 bin | inline | release.yml | 120+80 OK | green | ungated |
| ralph-wiggum-loop | **`rwl`** | 1 bin | inline | release-and-publish.yml | 120+80 OK | green | ungated |
| renew | *(lib)* | lib | inline | none | **100, WRONG** | **RED (test)** | ungated |
| whitespace | `whitespace` | 1 bin | inline | release-and-publish.yml **+ docker→ghcr** | 120+80 OK | green | ungated |

**bin ≠ crate name** for drata-cli (`drata`), pagerduty-cli (`pd`), ralph-wiggum-loop (`rwl`). `rust-cli-release@v1` defaults `archive-prefix` and `bin-names` to the **repo name**, so each of these MUST pass both explicitly, or ship `drata-cli-*` / `pagerduty-cli-*` / `ralph-wiggum-loop-*` tarballs. whitespace matches (no override).

### Release artifact contract (verified from `rust-cli-release.yml`)
- Default targets = `linux-amd64 linux-arm64 macos-x86_64 macos-arm64` (exactly what these repos ship).
- Archive = `${archive-prefix}-${tag}-${suffix}.tar.gz` + `.sha256`, via `softprops/action-gh-release`.
- No repo uses completions/extra files → `extra-files` input not needed.

### Implementation Plan

#### Phase 0: Spike (DONE)
**Model:** n/a
- mcp-io-rs (gated) migrated to `rust-ci.yml@v1`, PR #6 merged green, org Required-Workflows check passed alongside the public reusable.
- **Success criteria:** a gated public repo runs a public thin-caller AND the org security check green. ✅

#### Phase 1: rustfmt normalization (pre-req)
**Model:** sonnet
- renew: `max_width 100 → 120` + add `single_line_if_else_max_width = 80`. drata-cli: add the missing `single_line_if_else_max_width = 80`. Run `cargo fmt` to reformat.
- **Success criteria:** `cargo fmt --all --check` clean under org config on both; no other repo touched.

#### Phase 2: Fix renew's red test
**Model:** opus
- Repair `tests::test_macro_all_arms_select_shared_source` (`src/tests.rs`): `build.rs` `git describe` yields a bare short SHA (no tag) when CI does a shallow checkout without tags, and the test `unwrap()`s it as semver and panics. renew's HEAD does carry `v0.3.2` locally, so the real failure is CI-side tag visibility, not a genuinely tagless repo. Durable fix: make the test/`build.rs` robust to a tagless/shallow `GIT_DESCRIBE` (don't rely on CI fetching tags for `rust-ci`, which checks out shallow). Independent of migration; `otto ci` runs tests so renew stays red until fixed.
- **Success criteria:** `cargo test` green on a shallow checkout with no tags fetched.

#### Phase 3: SAST reusable extraction
**Model:** sonnet
- Add `github-actions-public/.github/workflows/sast.yml` (`on: workflow_call`), lifted from clyde verbatim (Semgrep/Checkov/Trivy, audit, public actions, SHA-pinned). Move the `@v1` major tag to include it.
- **Success criteria:** a `uses: .../sast.yml@v1` caller runs all three scanners, uploads SARIF, never blocks.

#### Phase 4: CI migration
**Model:** sonnet
- Replace inline `ci.yml` with a `rust-ci.yml@v1` caller in the 6 inline repos, swap okta-auth-rs off internal `@v5` onto `@v1`, AND swap **clyde** off its inline `otto ci` onto the `@v1` caller (clyde runs the right steps inline today but is not on the shared caller). That is 8 repos here; mcp-io (the 9th) is already on the caller. One PR per repo; gated repos via PR keeping the org check green.
- Every repo already has an `.otto.yml` with a `ci` task (verified), so this is a caller swap, not an otto-onboarding. **Pre-flight per repo:** run `otto ci` locally BEFORE the PR. The old inline CI ran raw `cargo`; `otto ci` is stricter (whitespace `-r`, `--all-features` clippy on the org-standard 1.96.0 toolchain, and a `_var` binding deny in SOME repos' lint (drata-cli/renew, not ralph-wiggum-loop/whitespace)), so a green-under-cargo repo can surface new findings on first `otto ci` (exactly what mcp-io hit). Fix those in the same PR.
- **Success criteria:** each `ci` run green and resolves `github-actions-public`; okta-auth-rs no longer 0-job-fails.

#### Phase 5: Release migration
**Model:** sonnet
- Thin `rust-cli-release@v1` callers for drata-cli (`bin-names: drata`, `archive-prefix: drata`), pagerduty-cli (`pd`/`pd`), ralph-wiggum-loop (`rwl`/`rwl`), and **clyde** (bin `clyde` == repo name, no override; swaps its vendored release for the caller). whitespace's release is Phase 6 (docker). Missing the bin override is a HARD failure, not a rename: `rust-cli-release` does `cp target/.../release/<archive-prefix-default=repo-name>`, which errors when the binary is `drata`/`pd`/`rwl`, killing the release job.
- **Success criteria:** a REAL test tag on each produces byte-for-byte identically-named tarballs+sha256 (`pd-vX-*`, `drata-vX-*`, `rwl-vX-*`, `clyde-vX-*`).

#### Phase 6: whitespace (release + docker in ONE workflow)
**Model:** opus
- whitespace's docker job `needs: build-linux` and consumes same-run artifacts via `download-artifact@v4` (`build-contexts: binaries=artifacts/`). `download-artifact@v4` is scoped to `run_id`, so a SEPARATE `docker-publish.yml` on the same tag (different run) CANNOT see those artifacts and would fail immediately.
- Correct shape: keep docker in whitespace's `release.yml` **caller** as a `docker:` job with `needs: release` (the reusable runs in the caller's run_id, so its uploaded per-target tarballs are downloadable by the sibling docker job). release half = `uses: rust-cli-release.yml@v1`; docker half = the existing buildx→ghcr job, now depending on the reusable and pulling the same artifact names the reusable produces (`whitespace-<tag>-linux-amd64/arm64`).
- **Success criteria:** a test tag yields all 4 tarballs AND a pushed multi-arch `ghcr.io/tatari-tv/whitespace` image, from a single workflow run.

#### Phase 7: SAST callers
**Model:** sonnet
- Add a thin `sast.yml@v1` caller to all 9 public Rust repos (the 8 here PLUS mcp-io, which has a CI caller but no SAST yet); swap clyde's repo-local `sast.yml` for the caller.
- **Success criteria:** all 9 run SAST via the reusable; no repo holds a copied full SAST file.

## Acceptance Criteria
- [ ] All 9 public Rust repos run `ci` via `rust-ci.yml@v1`, green on default branch.
- [ ] The 5 CLIs (drata-cli, pagerduty-cli, ralph-wiggum-loop, clyde, whitespace) release via `rust-cli-release.yml@v1`; a post-migration test tag produces the SAME artifact names as before (`drata-*`, `pd-*`, `rwl-*`, `clyde-*`, `whitespace-*`).
- [ ] whitespace still publishes its multi-arch `ghcr.io/tatari-tv/whitespace` image post-migration.
- [ ] Every public Rust repo runs `sast` via `sast.yml@v1`; no repo contains a copied full SAST workflow.
- [ ] Every public Rust repo commits `rustfmt.toml` (120 + 80).
- [ ] okta-auth-rs and renew CI are green.
- [ ] Preserved workflows still run: claude-pricing/clyde pages + refresh-pricing.

## Resolved Decisions
- 2026-07-10 (Scott): SAST spreads to all public Rust repos, as a reusable workflow + thin callers (not copy-paste).
- 2026-07-10 (Scott): `@v1` may move as a reusable-workflow major tag (one-time waiver of never-move-tags).
- 2026-07-11 (verified): libraries (claude-pricing, okta-auth-rs, renew) get CI + SAST only, no release.
- 2026-07-11 (verified): drata-cli/pagerduty-cli/ralph-wiggum-loop MUST pass explicit `bin-names`+`archive-prefix`; whitespace needs none.
- 2026-07-11 (Scott + verified): SAST has ONE definition, `github-actions-public/sast.yml@v1`, triggered by a repo-local thin caller in each public repo. There is NO double-run: the org-required SAST (`fleet-sast`, ruleset #126206) **explicitly excludes public repos** (fleet-sast.md: "Public repos are excluded from that ruleset"; goal = "all private repos"). So repo-local callers are the permanent SAST mechanism for public repos, not an interim to be superseded.
- 2026-07-11 (org standard): converge every repo on Rust 1.96.0 (rust-ci/release default). drata-cli (1.94.0) and renew (1.94.1) get bumped. 1.96.0 clippy is stricter (the known 1.96 CI-drift class), so expect new findings on the Phase 4 pre-flight and fix them in the same PR. Override per-repo `rust-version` only if a repo has a concrete blocker.
- 2026-07-11 (scope): this migration does NOT harmonize lint policy across repos. The `_var` binding deny exists in some `.otto.yml` files (drata-cli, renew, mcp-io) but not others (ralph-wiggum-loop, whitespace, which contain `_name` bindings). Aligning them is out of scope; each repo keeps its current `otto ci` contract.

## Review Panel Dispositions (2026-07-11)

Architect (Gemini) + Staff Engineer (Codex), Design Review mode. All findings accepted and folded in:
1. (both, HIGH) Phase 6 docker split broken — `download-artifact@v4` can't cross `run_id`. → Rewrote Phase 6: docker stays a `needs: release` job in the same caller workflow.
2. (both, HIGH) Missing bin override is a hard `cp` failure, not a rename. → Strengthened Phase 5 + risk row; real test tag mandatory.
3. (Staff, HIGH) "All 9" contradicted the phases (clyde not on the caller; mcp-io had no SAST). → Added clyde to Phases 4/5, mcp-io to Phase 7; counts reconciled.
4. (both) SAST double-run open question. → Closed: fleet-sast excludes public repos (verified), so no double-run; repo-local callers are permanent.
5. (Staff, MED) Silent toolchain bump 1.94→1.96. → New Resolved Decision: converge on 1.96.0, expect clippy findings.
6. (Staff, MED) `_var` deny not universal. → Reworded Phase 4; lint-policy harmonization declared out of scope.
7. (Architect, MED) No rollback protocol. → Added a Rollback clause to Rollout Plan.
8. (Staff, LOW) renew root cause is CI tag-visibility, not a tagless repo. → Tightened Phase 2 to fix the shallow-checkout case.

No pushbacks; no findings escalated. Open Questions empty.

## Alternatives Considered

### Alternative 1: Copy clyde's sast.yml into each repo
- **Cons:** nine copies drift; a scanner bump is a nine-repo change. **Why not:** the duplication class this migration kills.

### Alternative 2: Only fix the two reds, leave green repos inline
- **Cons:** leaves drift + no SAST on 6 repos. **Why not:** Scott asked for the full spread.

## Technical Considerations

### Dependencies
- `github-actions-public` reusables; `tatari-tv/whitespace` public releases (rust-ci downloads it).
- internal `github-actions#663` (whitespace twin sync) should land so all three `rust-ci.yml` twins stay byte-identical; not a functional blocker (public repo already has the fix).

### Security
- SAST audit-mode, public actions, SHA-pinned. Secret-scanning stays with the org "Tatari Org Security" (gitleaks), not duplicated.

### Testing Strategy
- Per repo: migration PR CI green; for CLIs, a real test tag to prove artifact-name parity (done means live).

### Rollout Plan
- Phases in order; reds fixed early (renew test in Phase 2, okta-auth in Phase 4). Land in-flight PRs before the next.
- **Phases 4, 5, 7 are multi-repo:** the unit of work is one PR per repo (not one commit for the phase). An execution driver should iterate repos within the phase, each PR babysat to green independently. Ungated repos (pagerduty-cli, ralph-wiggum-loop, renew, whitespace) can push direct; gated repos (claude-pricing, clyde, drata-cli, okta-auth-rs) go via PR and must keep the org Required-Workflows check green.
- **Rollback:** if a migration PR flips a repo's default branch red with no trivial forward fix, revert the caller swap immediately (ungated: direct-push revert; gated: revert PR) to restore the prior inline CI, and pause the fleet at that phase until the cause is diagnosed. Each phase is independently revertible because the migration is per-repo and additive to the shared workflows.

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| drata/pd/rwl missing bin override → release job dies (`cp` of a nonexistent `<repo-name>` binary), not just a rename | High | High | Explicit `bin-names`+`archive-prefix`; prove with a REAL test tag before announcing |
| whitespace ghcr publish lost in migration | Med | High | Phase 6 splits docker into its own workflow first; verify image pushes |
| Migrating a green repo flips it red on fmt | Med | Med | rustfmt sweep (Phase 1) before any ci swap |
| repo-local sast + future org-required sast double-run | Med | Low | audit-mode is cheap; revisit removal when org-required registers |
| renew stays red (test bug orthogonal to CI) | High | Med | Phase 2 fixes the test before its CI swap counts as green |
| `otto ci` stricter than old inline cargo → green repo fails on first migration | Med | Med | run `otto ci` locally per repo as Phase 4 pre-flight; fix findings in the same PR |
| claude-pricing daily data commits trigger the new ci caller (noise/cost) | Med | Low | the caller fires on `push`; keep refresh-pricing commits from triggering ci (path filter or `[skip ci]`), or accept audit-cheap noise |
| whitespace CI bootstraps on its own release | Low | Low | benign: lints against last released whitespace; documented |

## Open Questions
- None. (The SAST double-run question was closed: fleet-sast excludes public repos, so repo-local callers are permanent and never double-run. See Resolved Decisions.)

## References
- `project-github-actions-public` (memory)
- internal `github-actions#663` (whitespace twin sync)
- `tatari-tv/mcp-io-rs#6` (proven pattern)
- `fleet-sast` design doc (`tatari-tv/github-required-workflows`, docs/design/2026-07-03-fleet-sast.md)
