# Goal G-AT-AUDITOR-REMOVE-1 — Remove cross-provider auditor

**Status**: PROPOSED — Owner-authorized 2026-05-19. Wave 1 of the post-4-goal sprint.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-AUDITOR-REMOVE-1`

---

## TL;DR

Cross-provider auditor (`--auditor-provider`) was shipped in v0.6.4 as one of three G-AT-SEC-1 sub-features. Per Advisor honest re-evaluation + Owner decision, this feature doesn't carry enough solo-Owner value to justify shipping it. **Remove cleanly before any public dogfood** so the feature surface is honest and minimal. Tamper-evident log + hardware signing (the other 2 SEC-1 features) stay.

## Success criterion (single sentence)

> After this goal: no `--auditor-provider` flag, no `auditor_verdict` event in dispatch-log schema, no auditor code path in route.rs, no `sec_1_auditor_smoke.rs` test file, no `auditor_provider_configured` line in `aiplus doctor`; everything else from v0.6.4 unchanged; all 358 remaining aiplus-cli tests + 553 workspace tests still PASS.

## Goals (D1-D3)

- **D1**: Remove all auditor code paths from `route.rs`, `commands.rs`, `doctor.rs`.
- **D2**: Delete `tests/sec_1_auditor_smoke.rs` (3 tests). Net test count: 363 → 360 aiplus-cli; 558 → 555 workspace (or thereabouts).
- **D3**: CHANGELOG entry under upcoming v0.6.5 explaining the removal honestly.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/auditor-remove-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/auditor-remove-impl-notes.md` (Phase 1 design + Phase 3 evidence) | PENDING — CEO |
| 4 | Code deletions across `route.rs` / `commands.rs` / `doctor.rs` | PENDING — CEO |
| 5 | Test file deletion + CHANGELOG draft text in impl-notes (Advisor commits actual CHANGELOG) | PENDING — CEO |

## Phases

```
Phase 1 (CEO codex, ~30 min):
  - Write impl-notes design: precise grep targets, what gets deleted, what stays.

Phase 2 (CEO codex, ~2-3 hours):
  - Make deletions.
  - Run cargo test to confirm everything else still green.

Phase 3 (CEO codex, ~1 hour):
  - Evidence trailer with test counts before/after.
```

**Total target**: ≤ 1 day (likely ~half-day).

## Out of scope

- `audit.rs` / `verify_log.rs` (tamper-evident log — STAYS)
- `identity/setup_signing.rs` (hardware signing — STAYS)
- CONTRACT.md (FROZEN)
- Adapter code
- Scoring rubric
- Any v0.3 feature

## Acceptance criteria

- ✓ `grep -r "auditor_provider\|auditor_verdict\|AuditorVerdict" crates/aiplus-cli/` returns nothing
- ✓ `crates/aiplus-cli/tests/sec_1_auditor_smoke.rs` does not exist
- ✓ `cargo test --package aiplus-cli` PASS (test count decreases by 3)
- ✓ `cargo test --workspace` PASS (test count decreases by 3)
- ✓ `cargo build --bin aiplus` PASS, no clippy regressions vs main
- ✓ `aiplus doctor` output no longer includes `auditor_provider_configured` line
- ✓ `aiplus agent route --help` no longer shows `--auditor-provider` flag

## Why this goal exists (forensic note for future Advisor)

Cross-provider auditor was in DESIGN.md §22 as a v0.3 "Auditor independence Layer 7" consideration. Advisor packaged it into G-AT-SEC-1 as one of 3 sub-features per the "全都做" program commitment. After ship, Advisor's honest re-evaluation flagged the feature as marginal for solo-Owner use: doubles API cost per task, no monitoring, expected one-shot evaluation. Owner agreed and chose removal over deprecation-and-keep. Tamper-evident log and hardware signing stay because they have proven solo-Owner value.

## Dependencies

- v0.6.4 shipped ✓
- Owner explicit removal authorization ✓ (2026-05-19)

---

— Advisor, 2026-05-19. Wave 1 of post-4-goal sprint.
