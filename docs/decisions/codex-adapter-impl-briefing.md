# Codex Adapter Reference Implementation — Briefing for CEO Codex (Round 3)

**Drafted**: 2026-05-18 by Advisor (CONTRACT v1 FROZEN author)
**Target executor**: CEO codex session
**Authoritative source**: CONTRACT v1 FROZEN at commit `2affaa9`
**Predecessor rounds**: Claude Code round 1 (`d04797e`), OpenCode round 2 (`90500cb`)

---

You are CEO of AiPlus Agent Team. Implement `adapters/codex/` as **spiral round 3** against CONTRACT v1 FROZEN.

## Why round 3 matters (READ THIS FIRST)

Round 3 is **completeness validation**, not stress-testing. CONTRACT v1 was Frozen after 2 adapters (Claude Code + OpenCode) produced zero `_CONTRACT-BUG_` painpoints. Round 3 confirms the Frozen state holds against a 3rd runtime with different process model / output format.

**Expected outcome**: PASS on all 7 conformance specs with **zero `_CONTRACT-BUG_` painpoints**. Codex has a clean process boundary, structured JSON output (`--json` JSONL + `--output-schema`), and explicit `-C, --cd <DIR>` cwd flag. It is the *easiest* of the three runtimes to adapt — that's the point: if v1 fails on Codex, the failure is highly informative.

**If any `_CONTRACT-BUG_` painpoint surfaces**: STOP, return to Advisor. This would trigger v1 **unfreeze** + v2 spiral.

## Version lock

- **CONTRACT v1 FROZEN** at commit `2affaa9`: `adapters/CONTRACT.md` + `adapters/conformance/*.spec.md`.
- DO NOT edit CONTRACT or specs.

## Strict working order — sequential, not parallel

**Phase 1 — write `adapters/codex/IMPLEMENTATION.md` ONLY. All 7 sections (CONTRACT Appendix B). NO adapter code yet.**

Cheat sheet — Codex's clean ABI makes most sections low-friction:

1. Transport: subprocess + `codex exec --json` (JSONL events) + `--output-schema <file>` for structured final response + `--output-last-message <file>` if cleaner stdout separation needed
2. Config isolation: `-c key=value` per-key overrides + `CODEX_HOME=<isolated-dir>` env override (verify name during impl) + `--profile <isolated>` if profile-based isolation is cleaner
3. cwd: `-C, --cd <worktree_path>` flag (per CONTRACT Appendix A; confirmed in Step 1 fact-finding)
4. Tool whitelist: `--sandbox` policy levels are coarser than Claude Code's per-tool; map `allowed_tools[]` to nearest sandbox policy + document granularity gap as IMPLEMENTATION-level (NOT a `_CONTRACT-BUG_` — already established in OpenCode round 2 painpoint that coarse tool whitelist is per-adapter detail)
5. `usage_tokens`: parse `codex exec --json` JSONL events for token counts (Codex includes them; verify field names during impl)
6. Session continuation: `codex exec resume <session-id> --last` for resume; map per CONTRACT §4 + §4.3
7. Trace: same project-local pattern as prior adapters — `.aiplus/agent-team/adapter-trace/<session_id>.jsonl` with redaction

**Phase 2 — implement adapter binary/script. IMPLEMENTATION.md MUST be complete before this phase starts.**

**Phase 3 — run conformance + capture painpoints.**

## Conformance sequence + STOP rule

Run in this order:

`#01 → #02 → #03 → #06 → #04 → #05 → #07`

**Painpoint classification (same as round 2)**:

- `_CODEX-BUG_` (rare; Codex is clean): runtime-specific issue → log, work around, continue
- `_CONTRACT-BUG_`: STOP IMMEDIATELY, return to Advisor (triggers unfreeze)

Painpoints append to **`adapters/CONTRACT-v1-painpoints.md`** (same file as round 2 — DO NOT create new file).

## Scope fence

- **IN**: `adapters/codex/*` only
- **OUT**: other adapters, CONTRACT.md, conformance specs, new specs
- **GRAY**: append additional entries to `adapters/conformance/_harness/global_config_markers.txt` if Codex global config lives in non-standard paths

## Deliverables (4 — all required)

1. `adapters/codex/IMPLEMENTATION.md` — 7 sections per Appendix B
2. `adapters/codex/<binary or script>` — the adapter
3. Conformance 1-7 evidence → paste at IMPLEMENTATION.md trailer
4. `adapters/CONTRACT-v1-painpoints.md` — appended round-3 painpoints (or `## Round 3 (Codex): Painpoints (none)` if clean)

## Time ceiling

**Hard ceiling: T+2d** (Codex is cleaner than the prior two; expect faster). STOP-on-CONTRACT-BUG overrides.

## Success signal

- **Zero `_CONTRACT-BUG_` painpoints** = CONTRACT v1 FROZEN status confirmed across 3 adapters; production readiness ratified.
- **Any `_CONTRACT-BUG_` painpoint** = unfreeze + v2 spiral round needed.

## Handoff endpoint

After all 4 deliverables exist → **report back to Advisor**. Advisor confirms Frozen status holds OR drafts v2.

## Owner gates (always active)

No push, no publish, no global config edits, no force operations, no editing files outside IN scope.

---

— Advisor (CONTRACT v1 FROZEN author), 2026-05-18, against commit `2affaa9`
