# Goal G-AT-TOKEN-COST-STANDALONE-1 — Extract aiplus-token-cost as dual-mode (standalone + bundled)

**Status**: PROPOSED — Owner-authorized 2026-05-19 with design choices (default-bundled / prebuilt-binary install / source-of-truth = standalone repo).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-TOKEN-COST-STANDALONE-1`

---

## TL;DR

`aiplus-token-cost` (v0.6.5 G-AT-TOKEN-COST-1) currently lives as a workspace member crate inside aiplus-public, exposed only via `aiplus agent token-cost` subcommand. Owner wants it usable two ways: **(1) standalone** — download/install/use without installing full aiplus; **(2) bundled** — auto-installed alongside aiplus via `aiplus install`.

This goal **extracts the crate into a new standalone GitHub repo** (`github.com/izhiwen/AiPlus-Token-Cost`) as source-of-truth, with its own CLI binary + install.sh + release pipeline. The aiplus-public workspace member becomes a **git subtree mirror** of the standalone repo; existing `aiplus agent token-cost` subcommand behavior unchanged.

## Success criterion (single sentence)

> Owner can either (a) `curl -L .../AiPlus-Token-Cost/releases/latest/download/aiplus-token-cost-aarch64-apple-darwin.tar.gz | tar xz && sudo mv aiplus-token-cost /usr/local/bin/ && aiplus-token-cost` for standalone use, OR (b) `aiplus install <runtime>` to get both `aiplus` and `aiplus-token-cost` binaries installed together, with `aiplus agent token-cost` continuing to work as before.

## Goals (D1-D6)

- **D1**: New GitHub repo `izhiwen/AiPlus-Token-Cost` created (Owner action) with initial v0.1.0 commit pushed by Advisor.
- **D2**: Standalone binary — `aiplus-token-cost` CLI executable shipped via the new repo's release pipeline. Apple Silicon Mac + Intel Windows (matching aiplus-public's narrowed platform support).
- **D3**: aiplus-public bundling — `aiplus install` ships and installs both `aiplus` and `aiplus-token-cost` binaries from the same tar.gz; `aiplus agent token-cost` subcommand continues to work.
- **D4**: README updates — aiplus-public main README adds `aiplus-token-cost` as 7th bundled module entry (alongside the 6 substrate modules), with explicit "also installable standalone" mention. README.zh-CN parity.
- **D5**: `assets/aiplus-token-cost/` materialized in aiplus-public with module manifest (`aiplus-module.json`), README, LICENSE, etc., matching the existing 6 module convention.
- **D6**: Git subtree pull mechanism established — Advisor documents the sync procedure for future updates (source = standalone repo, mirror = aiplus-public).

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/token-cost-standalone-impl-briefing.md` (CEO briefing for Phase B) | DRAFT (this commit) |
| 3 | AiPlus-Token-Cost initial v0.1.0 commit (Phase A, Advisor) | PENDING |
| 4 | AiPlus-Token-Cost release.yml + first tagged release (Phase A, after repo exists) | PENDING |
| 5 | `aiplus-public/assets/aiplus-token-cost/` (Phase B, CEO) | PENDING |
| 6 | aiplus-public release.yml updates — dual-binary tar.gz (Phase B, CEO) | PENDING |
| 7 | `aiplus install` updates — install both binaries (Phase B, CEO) | PENDING |
| 8 | README updates — "7 bundled modules" framing (Phase B, CEO) | PENDING |
| 9 | Git subtree sync documentation (Phase C, Advisor) | PENDING |
| 10 | Retrospective (Phase C, Advisor) | PENDING |

## Phases + sequencing

```
Phase A (Advisor + Owner, ~0.5 day):
  - Advisor: prepare AiPlus-Token-Cost initial layout in ~/Projects/AiPlus/AiPlus-Token-Cost/
    (main.rs CLI + install.sh + install.ps1 + release.yml + README × 2 + manifest + LICENSE + initial CHANGELOG)
  - Owner: `gh repo create izhiwen/AiPlus-Token-Cost --public --description "..."` (empty repo)
  - Advisor: git init + push initial commit + tag v0.1.0 → triggers release CI
  - Verify: standalone install works from the new release

Phase B (CEO codex, ~2-3 days):
  - aiplus-public/assets/aiplus-token-cost/ added with manifest + README + LICENSE
  - aiplus-public/.github/workflows/release.yml: bundle aiplus-token-cost binary
    into aiplus tar.gz (same release, dual binaries)
  - `aiplus install` updates to deploy both binaries
  - README + README.zh-CN: 6 modules → 7 modules; token-cost entry with both
    install paths documented
  - Continue to work: `aiplus agent token-cost` subcommand

Phase C (Advisor, ~0.5 day):
  - Set up git subtree pull mechanism: documented in
    aiplus-public/docs/operations/token-cost-subtree-sync.md
  - Verify both install modes end-to-end
  - Retrospective
  - Close goal
```

**Total target**: ~3-5 days.

## Out of scope

- **cargo install path** (Owner chose prebuilt-binary only for standalone install). Future: could add if users request it.
- **YubiKey path** for setup-signing (separate goal, future).
- **Multi-currency support** in pricing (USD only for v1, future).
- **Real-time alerts / dashboards** beyond CLI display (future).
- **Changes to existing `crates/aiplus-token-cost/` workspace path** — must stay where it is so aiplus-cli's `agent token-cost` keeps working.
- **CONTRACT.md edits** (FROZEN, irrelevant to this work).

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| New GitHub repo creation requires Owner credentials | Owner runs `gh repo create` explicitly; Advisor prepares everything locally first |
| Standalone CLI duplicates aiplus-cli's `agent token-cost` parsing | Extract shared CLI logic into aiplus-token-cost crate; aiplus-cli's subcommand becomes a thin wrapper |
| Dual-binary tar.gz breaks existing install.sh | install.sh extracts both binaries; backward-compatible with single-binary tar.gz via tar listing |
| Git subtree drift between source-of-truth and mirror | Documented sync procedure; CI check (Phase C optional) verifies `crates/aiplus-token-cost/` matches the latest AiPlus-Token-Cost tag |
| README "6 → 7 modules" rebrand may surprise users | Explicit migration note in CHANGELOG; existing 6 modules unchanged |
| First release on new repo could expose bugs in release.yml not seen on aiplus-public | Phase A includes "verify v0.1.0 release pipeline runs cleanly" as gate |

## Acceptance criteria

- ✓ D1-D6 met
- ✓ Standalone install works: `curl -L .../release/aiplus-token-cost-*.tar.gz | tar xz && ./aiplus-token-cost` runs without aiplus installed
- ✓ Bundled install works: `aiplus install <runtime>` deploys both binaries to `~/.local/bin/`
- ✓ `aiplus agent token-cost` subcommand continues to work
- ✓ Tests pass: `cargo test --workspace` in aiplus-public + `cargo test` in AiPlus-Token-Cost
- ✓ Both READMEs updated; "7 bundled modules" framing consistent

## Dependencies

- aiplus-public v0.6.5 shipped ✓ (`47873e6` post-Phase-C of Wave 1+2 sprint)
- Owner authority to create new GitHub repo
- Platform support narrowed to 2 (Apple Silicon Mac + Intel Windows) — already done

## Next action

Owner creates `izhiwen/AiPlus-Token-Cost` empty public repo; Advisor pushes initial commit + triggers Phase B.

---

— Advisor, 2026-05-19, dual-mode goal post-platform-narrowing.
