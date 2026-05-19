# Goal G-AT-POLISH-1 — Polish bundle (AdapterResult + schemaVersion + doctor --quiet + clippy + briefing Skill)

**Status**: PROPOSED — Owner-authorized 2026-05-19. Wave 2-D (parallel with B and C).
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-POLISH-1`

---

## TL;DR

Ship 6 polish items together as v0.6.5 polish bundle. Each item is small to medium; bundling reduces Advisor Phase-C overhead vs shipping 6 separate goals. Order by blast radius — AdapterResult plumbing (largest) first since other items can rebase on top.

## Goals (D1-D6)

- **D1**: **AdapterResult return-value plumbing** — `route_known_role` returns `Result<AdapterResult>` not `Result<()>`. Threads structured adapter output (final_text, tool_calls, usage_tokens) back to callers. Unblocks future features that act on primary output.
- **D2**: **dispatch-log row `schemaVersion` field** — every JSONL row in `.aiplus/agents/dispatch-log.jsonl` includes `"schemaVersion": "0.4.0"` (or whichever version) to let downstream consumers feature-detect cleanly.
- **D3**: **`aiplus doctor --quiet`** — flag that suppresses INFO lines, shows only WARN+ and FAIL.
- **D4**: **install.sh / Cargo.toml parity pre-commit hook** — when `Cargo.toml` version field changes, hook checks `install.sh` fallback version matches. Refuses commit otherwise.
- **D5**: **Clippy lint debt cleanup** in pre-existing `aiplus-core` + older `aiplus-cli` files so `cargo clippy --workspace --all-targets -- -D warnings` runs clean.
- **D6**: **Briefing template Skill** — extract the recurring briefing structure (worktree isolation / shared-file ownership / STOP rule / retry-once gate / phase structure / scope fence) into a reusable Skill at `~/Projects/AiPlus/aiplus-agent-team/skills/aiplus-ceo-briefing.md`.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/polish-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/polish-1-impl-notes.md` | PENDING — CEO |
| 4 | `crates/aiplus-cli/src/agent/route.rs` — AdapterResult plumbing + schemaVersion field | PENDING — CEO |
| 5 | `crates/aiplus-cli/src/agent/doctor.rs` — `--quiet` flag handling | PENDING — CEO |
| 6 | `.git/hooks/pre-commit` install script (or `scripts/install-hooks.sh`) | PENDING — CEO |
| 7 | Clippy fixes across `aiplus-core/` and older `aiplus-cli/` files | PENDING — CEO |
| 8 | `aiplus-agent-team/skills/aiplus-ceo-briefing.md` Skill file | PENDING — CEO (in this same goal — note CROSS-REPO) |

## Phases

```
Phase 1 (CEO codex, ~1 day):
  - Write impl-notes design
  - Section per sub-feature:
    a. AdapterResult return chain — which call sites need updates
    b. schemaVersion field — bump rule + version string
    c. --quiet behavior spec
    d. pre-commit hook script + installation pattern
    e. clippy inventory: grep -rn 'allow\|expect\|deny' + run clippy to enumerate exact issues
    f. Skill content outline based on prior 8 briefings

Phase 2 (CEO codex, ~2-3 days):
  - Order:
    a. AdapterResult plumbing FIRST (largest blast radius; biggest function-signature change)
    b. schemaVersion field (small append)
    c. doctor --quiet flag (small)
    d. pre-commit hook (separate file, no Rust)
    e. clippy cleanup (file-by-file, no behavior change)
    f. Skill file write

Phase 3 (CEO codex, ~0.5 day):
  - All tests green
  - cargo clippy --workspace -- -D warnings PASSES
  - Live: pre-commit hook fires on a fake parity mismatch (test it)
  - Live: aiplus doctor --quiet hides INFO lines
  - Live: a dispatch-log row has schemaVersion field
  - Skill file exists and is structurally sensible
```

**Total target**: ≤ 5 days.

## Out of scope

- New features (not a feature goal)
- coordinator.rs (Wave 2-B owns)
- commands.rs new subcommand (Wave 2-C owns; Wave 2-D may add `--quiet` flag to existing `doctor` subcommand only)
- New crate (Wave 2-C)
- CONTRACT.md (FROZEN)
- Adapter code
- Calibration fixture existing 16 entries (lockable; calibration auto-summon entries are Wave 2-B's territory)

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| AdapterResult return-type change cascades into many test files | Plan in Phase 1; if too disruptive, split D1 to its own follow-up goal |
| Clippy cleanup reveals real bugs (not just lint debt) | Treat as separate to-fix items; STOP and ping Advisor before silently changing semantics |
| pre-commit hook conflicts with existing git hooks | Detect existing hooks; merge with care; provide --force install flag |
| Skill file format unclear (not sure where Claude Code reads it) | Skill location is `~/Projects/AiPlus/aiplus-agent-team/skills/`; ensure it's accessible via the Claude Code conversation (project context) |
| schemaVersion change breaks something that parses dispatch-log | Field is purely additive; existing parsers ignore unknown fields cleanly |
| Wave 2-C touches commands.rs at the same time | D only adds --quiet flag (modifies existing `Doctor` subcommand definition); C only adds new `TokenCost` variant. Different parts of commands.rs. Append-only discipline avoids collision. |

## Acceptance criteria

- ✓ D1-D6 met
- ✓ `cargo test --workspace` PASS
- ✓ `cargo clippy --workspace --all-targets -- -D warnings` PASS (no warnings)
- ✓ Live: pre-commit hook fires on Cargo.toml-vs-install.sh mismatch
- ✓ Live: `aiplus doctor --quiet` output has no INFO lines
- ✓ Live: tail `.aiplus/agents/dispatch-log.jsonl` shows `schemaVersion` field
- ✓ Skill file at `aiplus-agent-team/skills/aiplus-ceo-briefing.md` exists and is loadable

## Dependencies

- Wave 1 (G-AT-AUDITOR-REMOVE-1) merged to main
- Existing route.rs (post-Wave-1) as baseline for AdapterResult plumbing
- 8 prior briefing files in `aiplus-agent-team/docs/decisions/` as source material for the Skill content

## Next action

After Wave 1 merges, Advisor dispatches CEO for this goal in parallel with B and C.

---

— Advisor, 2026-05-19, Wave 2-D of post-4-goal sprint.
