# Cross-provider Auditor Removal — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (single session, ~half-day)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-AUDITOR-REMOVE-1.md`
- Reference for what to delete: v0.6.4 ship at `aiplus-public@47873e6`, commits `36b2976` + `eff0973`
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.auditor-remove/`
- Branch:    `feat/auditor-remove`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.auditor-remove -b feat/auditor-remove`
- All work inside this worktree. Do NOT touch main worktree.
- Push to `feat/auditor-remove` only.

Shared-file ownership: this session OWNS exclusively the files listed in "Deletion targets" below. Do NOT modify anything else.

## Scope — pure deletion + cleanup

Remove the cross-provider auditor feature shipped in v0.6.4 G-AT-SEC-1 D2. **Keep** the other 2 SEC-1 sub-features (tamper-evident log + hardware signing).

## Deletion targets (specific)

### File deletions

- `crates/aiplus-cli/tests/sec_1_auditor_smoke.rs` — delete entire file (3 tests inside).

### File modifications (delete code from)

- `crates/aiplus-cli/src/agent/route.rs`:
  - Remove the auditor dispatch block (the part that runs after primary completes and invokes a different provider's CLI)
  - Remove `auditor_verdict` event emission
  - Remove `--auditor-provider` plumbing in `run_adaptive_route` or wherever the flag is consumed
  - Remove imports + helpers used only by auditor code

- `crates/aiplus-cli/src/agent/commands.rs`:
  - Remove the `--auditor-provider <provider>` CLI flag definition

- `crates/aiplus-cli/src/agent/doctor.rs`:
  - Remove the `auditor_provider_configured` INFO line emission
  - Remove the helper function that computes it (if there is one and only used by this INFO)

### Files NOT to touch (stay completely)

- `crates/aiplus-cli/src/agent/audit.rs` — STAYS (it's the tamper-evident log module, not the cross-provider auditor; despite the name similarity, different feature)
- `crates/aiplus-cli/src/agent/audit/verify_log.rs` — STAYS
- `crates/aiplus-cli/src/identity/setup_signing.rs` — STAYS (hardware signing)
- `crates/aiplus-cli/tests/sec_1_tamper_evident_smoke.rs` — STAYS
- `crates/aiplus-cli/tests/sec_1_setup_signing_smoke.rs` — STAYS
- All other SEC-1 files

## Phase structure

### Phase 1 — Write `docs/proposals/auditor-remove-impl-notes.md` BEFORE coding

Required sections:

1. Precise grep targets and what each will find (use `git log -p 36b2976 -- crates/aiplus-cli/src/agent/route.rs` to find the auditor block boundaries).
2. List every code region to delete (file + line range or function name).
3. Verification plan: `cargo test`, `cargo build`, `grep -r "auditor_provider"` returns nothing.
4. CHANGELOG ## 0.6.5 draft text (Advisor will compose actual CHANGELOG; CEO drafts the auditor-removal paragraph).

### Phase 2 — Execute deletions

Suggested order:
1. `tests/sec_1_auditor_smoke.rs` delete (smallest first, immediately reduces what to keep working)
2. `route.rs` auditor block removal
3. `commands.rs` flag removal
4. `doctor.rs` INFO line removal

After each step: `cargo build --bin aiplus` to confirm still compiles.

### Phase 3 — Evidence

- Test count: aiplus-cli 363 → ~360 PASS; workspace 558 → ~555 PASS
- `grep -r "auditor_provider\|auditor_verdict\|AuditorVerdict" crates/aiplus-cli/` returns nothing
- `cargo build --bin aiplus` clean
- `./target/debug/aiplus agent route --help` does NOT show `--auditor-provider`
- `./target/debug/aiplus doctor` does NOT show `auditor_provider_configured`
- Trailer in `auditor-remove-impl-notes.md`

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** — a non-auditor test fails → STOP + ping Advisor
- **STOP only on `_SCOPE-BREACH_`** — you touched a file in the "STAYS" list above
- **Do NOT STOP on `_IMPL-BUG_`** — your deletion missed a callsite, fix and continue

### Retry-once gate (mandatory before classifying any FAIL)

Standard pattern. (Less critical for deletion work but still required.)

## Conformance / verification

- 0 new clippy warnings (deletion should reduce, not introduce, warnings)
- 0 forbidden files touched
- Test count decreased by exactly 3
- `cargo build --bin aiplus` clean

## Scope fence

- **IN**: deletions enumerated above + Phase 1 notes file
- **OUT**: everything else, especially `audit.rs` / `verify_log.rs` / `identity/setup_signing.rs` / CONTRACT.md / scoring / calibration fixture / install.sh / Cargo.toml version / CHANGELOG actual (only the draft text in notes)
- **GRAY**: small comment cleanup adjacent to deletions is OK

## Deliverables (3)

1. `docs/proposals/auditor-remove-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. Code deletions across `route.rs` / `commands.rs` / `doctor.rs` + test file deletion
3. CHANGELOG draft text in impl-notes (Advisor commits actual CHANGELOG at Phase C)

## Time ceiling

**Hard ceiling: T+1d** (likely ~half-day actual).

## Handoff endpoint

Report:
- `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}`
- Test count delta
- Confirmation that all 4 grep targets return nothing
- Phase 3 evidence summary

Advisor ratifies + ff-merges. (No version bump yet — v0.6.5 bump happens after Wave 2 ships.)

## Owner gates

Standard: no push main, no force ops, no `--amend`, no hook bypass.

BWS: `aiplus secret-broker run --aliases anthropic,openai -- <cmd>` if needed (probably not needed for pure deletion work).

## Dependencies

- v0.6.4 shipped ✓ (`aiplus-public@47873e6`)
- All auditor code added in `36b2976` + `eff0973` — those commits are the source-of-truth for "what to remove"

---

— Advisor, 2026-05-19, Wave 1 of post-4-goal sprint. Day-0 narrow STOP + retry-once per `advisor_briefing_stop_rules`.
