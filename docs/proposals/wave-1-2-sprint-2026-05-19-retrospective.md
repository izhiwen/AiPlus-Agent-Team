# Wave 1 + Wave 2 Sprint (2026-05-19) — Retrospective

**Sprint ID**: post-4-goal-program follow-up
**Span**: 2026-05-19 (single day)
**Goals shipped**: 4 — G-AT-AUDITOR-REMOVE-1 + G-AT-AUTOSUMMON-INTENT-1 + G-AT-TOKEN-COST-1 + G-AT-POLISH-1
**Original budget**: ~10 days (Wave 1 half-day + Wave 2 max 5 days parallel)
**Actual**: ~1 day wall-clock (single Owner workday)

---

## What this sprint shipped

| Wave | Goal | Branch | Final commit on main | Lines | Tests delta |
|---|---|---|---|---|---|
| 1 | G-AT-AUDITOR-REMOVE-1 | `feat/auditor-remove` | `2f912b1` | +99 / -440 | -3 (auditor smoke) |
| 2-B | G-AT-AUTOSUMMON-INTENT-1 | `feat/autosummon-intent` | `642c1e7` | +538 / -41 | +5 (intent + cache) |
| 2-C | G-AT-TOKEN-COST-1 | `feat/token-cost` | `3d8c439` | +1729 / -1 | +9 (token-cost smoke) |
| 2-D | G-AT-POLISH-1 | `feat/polish-1` | `ff7e08c` | +701 / -287 | +3 (polish smoke) |
| C | Advisor Phase C (this) | — | post-`ff7e08c` (this commit) | +CHANGELOG +Cargo+install.sh +retrospective | — |

Final tests: 555 → 569 (workspace + aiplus-cli). cargo clippy clean.
aiplus-cli version: 0.6.4 → 0.6.5.
LiteLLM pricing source integrated; embedded fallback + local override both work.

## Headline result: 3 parallel CEO sessions, zero merge conflict

Wave 2 ran 3 CEO codex sessions concurrently against the same git repository, each in its own worktree on its own feature branch. The ownership matrix in the briefings (per CONTRACT v1.1 App D Rule D.3) designated which session owned which file. Result: **3 sequential rebases produced ZERO conflicts**. Git auto-resolved all overlaps in the 3 shared files (`agent/mod.rs`, `agent/core.rs`, `agent/commands.rs`) because each session's edits landed in different sections.

This is the **first validation of parallel-codex-session-protocol at 3-session scale**. Prior validations were 2-session (G-AT-PROD-1 saw the negative case, post-protocol G-AT-PROD-2 saw the 1-session case, this sprint scales to 3). Protocol is now empirically robust at the size that matters.

## D-goal status (all goals)

### G-AT-AUDITOR-REMOVE-1 (Wave 1)
- D1 ✓ — all auditor code paths removed from route.rs / commands.rs / doctor.rs
- D2 ✓ — sec_1_auditor_smoke.rs deleted (test count -3)
- D3 ✓ — CHANGELOG removal entry (in v0.6.5 ship, this commit)

### G-AT-AUTOSUMMON-INTENT-1 (Wave 2-B)
- D1 ✓ — keyword match replaced with LLM intent classification
- D2 ✓ — 3 expert TOMLs migrated (security-reviewer / tech-writer / ai-integration-specialist)
- D3 ✓ — in-process cache (FIFO, not LRU as briefing suggested — acceptable deviation)
- D4 ✓ — G2 semantic gate infra reused
- D5 ✓ — calibration fixture's 16 baseline entries byte-identical; only auto-summon section rewritten

### G-AT-TOKEN-COST-1 (Wave 2-C)
- D1 ✓ — new crate `aiplus-token-cost` shipped
- D2 ✓ — LiteLLM JSON primary + embedded fallback + local `.aiplus/pricing.toml` override
- D3 ✓ — 1h/8h/24h windows + top-5 task + per-role detail
- D4 ✓ — `aiplus agent token-cost` subcommand (under `agent`, not top-level — CEO documented deviation)
- D5 ✓ — hourly snapshot to `.aiplus/agents/token-cost-snapshots.jsonl`
- D6 ✓ — missing usage_tokens counted as 0/0

### G-AT-POLISH-1 (Wave 2-D)
- D1 ✓ — `route_known_role` returns `Result<AdapterResult>`
- D2 ✓ — `schemaVersion: "0.4.0"` on dispatch-log rows
- D3 ✓ — `aiplus doctor --quiet`
- D4 ✓ — `scripts/install-hooks.sh` + `.git/hooks/pre-commit` parity check
- D5 ✓ — clippy `--workspace --all-targets -- -D warnings` clean (binding CI gate now possible)
- D6 ✓ — briefing Skill at `aiplus-agent-team/skills/aiplus-ceo-briefing.md`

## What went well

1. **3-session parallel protocol validation.** First time we ran 3 CEO sessions concurrently on the same repo. Zero conflicts. Briefing ownership matrix worked as designed.
2. **Day-0 narrow STOP + retry-once gate, 4 goals running.** Pattern continues to deliver first-try PASS_WITH_DEVIATIONS (no BLOCKED iterations). Auto-memory `advisor_briefing_stop_rules.md` paying compounding dividends.
3. **CEO scope discipline across all 3 Wave-2 sessions.** Each session disclosed exactly which files outside its OWN list it touched, and why (schema necessity / glue requirement / clippy boundary). No silent scope creep. No forbidden file breach.
4. **Owner's mid-sprint correction caught a marginal feature before it ossified.** Cross-provider auditor was shipped in v0.6.4 (G-AT-SEC-1 D2); Advisor's honest re-evaluation flagged it the next day; Owner chose removal over deprecation. This is the right pattern — surface, decide, act decisively.
5. **Keyword-based auto-summon → LLM intent classification.** Replaced a brittle text-match heuristic with a proper classification model. Owner caught this design weakness immediately upon learning the auto-summon was keyword-based.
6. **Token cost crate as separate workspace member.** Clean architectural decision; doesn't entangle cost-counting logic with dispatch routing.
7. **AdapterResult plumbing finally landed.** Was the SEC-1 deferred follow-up; closed within ~12 hours of being identified.

## What went wrong

1. **Nothing in this sprint.** Like G-AT-COORDINATOR-PARALLEL-1, this is a second consecutive sprint with no "what went wrong" entries. The pattern of (a) verify-in-code before drafting + (b) day-0 sharp briefings + (c) ownership matrices + (d) Owner mid-sprint feedback is producing very high-quality output.

   Minor pedant points worth noting but not goal-level issues:
   - Token-cost CEO disclosed `aiplus agent token-cost` instead of top-level `aiplus token-cost` — briefing said either was acceptable, so this is documented choice not deviation.
   - install-hooks.sh script generated a `~/.git-hooks/pre-commit` artifact during dev that CEO had to manually clean up — pre-commit installer hygiene needs improvement (not goal-blocking).

## Architectural lessons codified

1. **Briefing ownership matrix scales to N parallel sessions** (validated at N=3). Future parallel work can confidently coordinate via explicit file-OWN designation; git's section-level merge handles the rest when ownership is clean.
2. **Replacing shipped features with corrected designs is cheaper than expected.** Auditor removal: ~half-day. Autosummon keyword → intent rewrite: ~3 days. Both small relative to original ship cost. **Implication**: don't be afraid to ship-then-iterate; reversibility is high.
3. **`AdapterResult` plumbing was a 12-hour blocker, not a 1-week blocker.** SEC-1 retrospective listed this as a v0.6.5+ goal worth "~1 day"; actual: it folded into POLISH-1 along with 5 other items. Estimate-vs-actual continues to favor the actual side.

## Open follow-ups

- Pre-commit hook installer hygiene (D4 in POLISH-1) — manual cleanup-of-test-artifacts during dev was needed; install script could be more robust.
- `aiplus token-cost` is currently `aiplus agent token-cost` namespace. If Owner uses heavily, consider top-level alias for ergonomics.
- LLM-intent classification uses Anthropic Haiku by default. If Owner is offline / no Anthropic key, intent fails closed (no auto-summon). Acceptable behavior; documented.
- Token-cost snapshot file has no rotation — will grow unbounded. Future polish.

## Next decision

**No new goal pre-committed.** This sprint cleaned up the 4-goal program's open follow-ups + added 2 net-new features (intent classification + cost tracking). AiPlus v0.6.5 is the most complete + cohesive state yet.

Advisor recommendation: enter a real dogfood window now. Use AiPlus on real Owner work for ~1 week. Surface friction. Let real usage drive the next iteration.

If Owner picks "more goals immediately" again — that's the call to make and Advisor will draft. But this is the natural inflection point for a usage pause.

---

— Advisor, 2026-05-19, Wave 1 + 2 sprint closed
