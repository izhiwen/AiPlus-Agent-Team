# Goal G-AT-TOKEN-COST-STANDALONE-1 — Retrospective (COMPLETE)

**Status**: **DONE** — `aiplus-token-cost` shipped dual-mode (standalone + bundled) on 2026-05-19.
**Original goal**: `docs/proposals/goal-G-AT-TOKEN-COST-STANDALONE-1.md`
**Closing date**: 2026-05-19 (opened same day)
**Duration**: ~1 work day (vs. ≤5 day budget). Under budget by ~80%.

---

## What shipped

| Phase | Sub-deliverable | Final commit / artifact | Repo |
|---|---|---|---|
| Spec | Goal doc + CEO briefing | `a65d014` | aiplus-agent-team |
| A | New standalone repo `izhiwen/AiPlus-Token-Cost` v0.1.0 release | repo + tag v0.1.0 + 2 binary artifacts | github.com/izhiwen/AiPlus-Token-Cost |
| B | aiplus-public bundling (assets, dual-binary release, install scripts, README × 2, manifest schema) | `9948e27` | aiplus-public |
| C | Cargo+install.sh bump 0.6.5 → 0.6.6 + CHANGELOG `## 0.6.6` + subtree sync doc | `3486394` | aiplus-public |
| C | Retrospective (this doc) + push | post-`3486394` | aiplus-agent-team |

**Total**: ~3197 lines standalone repo + ~731 lines aiplus-public bundling + ~150 lines spec/retrospective ≈ 4100 lines.

## D-goal status

- D1 (new GitHub repo created): ✓ — `izhiwen/AiPlus-Token-Cost` public, v0.1.0 release published with 2 binary artifacts + checksums
- D2 (standalone binary): ✓ — Apple Silicon Mac + Intel Windows artifacts in release; verified Apple Silicon binary runs `--help` cleanly
- D3 (aiplus-public bundling): ✓ — `aiplus install` deploys both binaries; `aiplus agent token-cost` still works
- D4 (README updates): ✓ — "6 → 7 bundled modules" in both English and Chinese READMEs
- D5 (`assets/aiplus-token-cost/` materialization): ✓ — full module set with manifest, README × 2, LICENSE, SECURITY, MODULES, RELEASE_CHECKLIST, runtime adapter notes
- D6 (subtree sync mechanism): ✓ — `docs/operations/token-cost-subtree-sync.md` published with two paths (subtree pull + manual sync) and Cargo.toml differences documented

## What went well

1. **gh CLI delegation worked smoothly.** Owner picked "create repo via web UI" but then said "请你帮我做这些事情". Advisor verified `gh auth status` first (logged in as `izhiwen` with `repo`+`workflow` scopes), then ran `gh repo create` per explicit Owner delegation. Cleaner than waiting for manual repo creation; faster end-to-end.
2. **CI release pipeline worked first-try.** The new AiPlus-Token-Cost `.github/workflows/release.yml` (modeled on aiplus-public's recently-narrowed 2-platform matrix) ran cleanly on first push: build (Apple Silicon Mac) + build (Intel Windows) + publish-release all green. ~5 min total.
3. **Cargo build / 9 tests / `--help` smoke test** all passed before any commit. The standalone repo's `main.rs` initially had wrong field names from the lib (`label`/`tokens` instead of `window_label`/`total_tokens`); caught on first build, fixed in one Edit, build clean. Tight loop.
4. **Module manifest schema extension** (`binaryAssets` field) was added by CEO during Phase B without my briefing explicitly specifying its exact shape. CEO inferred the right structure from `assets/aiplus-agent-memory/aiplus-module.json` as reference + my hint in the briefing. Clean schema evolution.
5. **CEO chose Approach A for dual-binary release** (download standalone binary at release time instead of building it locally in workspace). This is the more cross-repo-decoupled approach — standalone repo is fully source-of-truth, aiplus-public consumes its release artifacts at release time. Matches the long-term subtree pattern.
6. **Zero forbidden file touched.** CEO Phase B respected the scope fence rigorously: CONTRACT.md untouched, `crates/aiplus-token-cost/` untouched (subtree mirror handling is Phase C / Advisor), calibration fixture existing entries untouched. The one in-scope test edit (`parity.rs`) was a benign update to the bundled-module-list assertion.
7. **install.sh dual-binary backward compat.** CEO's install.sh handles both new dual-binary tar.gz AND older single-binary archives (silently skips second-binary step if absent). Mitigates users mid-upgrade from older tar.gz.

## What went wrong

1. **PowerShell live demo deferred.** CEO's local machine has no pwsh, so install.ps1's dual-binary support is code-path-verified but not live-tested. **Cost**: small risk that the Windows installer has a runtime bug not caught by static review. **Fix path**: next aiplus-public CI release in the wild will exercise it; or Advisor can run install.ps1 in a VM if needed.
2. **Subtree mechanism is "documented but not exercised."** D6 produced the operational doc, but no actual subtree pull has been performed yet (the v0.1.0 mirror was a manual copy at initial repo creation). First real test of the doc is when AiPlus-Token-Cost ships v0.1.1+. **Mitigation**: doc includes two paths (subtree pull + manual sync) and explicit Cargo.toml difference list; the first sync will validate which works.

## Architectural lessons codified

1. **Dual-mode (standalone + bundled) works for binary-producing modules.** Different pattern from the 6 substrate modules (data/template only). Key elements: independent repo with own release pipeline, manifest extension for `binaryAssets`, aiplus-public release workflow downloads from upstream module repo. Reusable for any future binary-shipping substrate module.
2. **Owner delegation of GitHub repo creation is OK when `gh auth status` is verified.** The Owner-gate rule "no external resource creation without consent" was honored by Advisor checking gh CLI auth state, then executing per Owner's explicit "请你帮我做这些事情" delegation. Saves Owner manual web-UI step.
3. **Module schema extensions land cleanly via `aiplus-module.json` evolution.** The `binaryAssets` field was added in Phase B without breaking the existing 6 substrate modules. JSON schema is forward-compatible by design.

## Open follow-ups

- First real subtree sync from AiPlus-Token-Cost to aiplus-public (when v0.1.1 ships). Validates `docs/operations/token-cost-subtree-sync.md`.
- PowerShell install.ps1 live verification on actual Windows (currently code-path-only).
- Future binary-shipping modules can copy this pattern.
- Decide whether to publish `aiplus-token-cost` to crates.io (currently `publish = false`).
- `aiplus install --module-only token-cost` (future ergonomic) — install just the standalone binary without full aiplus suite.

## Next goal

No new goal pre-committed. AiPlus is at v0.6.6, with 7 bundled substrate modules + standalone install paths for both `aiplus` and `aiplus-token-cost`.

Advisor recommends a real dogfood window (now even more strongly than before, since the toolchain just added a major dimension — dual-mode bundling — without any real Owner usage data). If Owner wants more goals, they can be drafted, but ~1 week of using the tooling on real work would inform priority better than continued speculation.

---

— Advisor, 2026-05-19, G-AT-TOKEN-COST-STANDALONE-1 closed.
