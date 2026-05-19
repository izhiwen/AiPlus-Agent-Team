# G-AT-SEC-1 Security Upgrades — Implementation Briefing for CEO Codex

**Drafted**: 2026-05-18 by Advisor (post G-AT-COORDINATOR-PARALLEL-1 close)
**Target executor**: CEO codex session (full impl — Advisor verifies + ratifies + closes 4-goal program)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-SEC-1.md`
- Design source: `aiplus-agent-team/DESIGN.md` §15.1 (GPG honest caveat + v0.3 mitigations) + §22 (Auditor independence, Layer 7)
- Existing dispatch-log: `aiplus-public/crates/aiplus-cli/src/agent/route.rs:291` (writes `.aiplus/agents/dispatch-log.jsonl`)
- Adapter contract: `aiplus-agent-team/adapters/CONTRACT.md` v1.1 FROZEN (no edits — sec layer is data-layer, not contract)
- Auto-memory: `advisor_briefing_stop_rules.md` (narrow STOP + retry-once gate baked in below)

---

## TL;DR

3 security sub-features per DESIGN.md §15.1/§22. Each independently shippable. Order by blast radius: tamper-evident log → cross-provider auditor → hardware signing. ~7-day hard ceiling.

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.sec-1/`
- Branch:    `feat/sec-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.sec-1 -b feat/sec-1`
- All work inside this worktree. Do NOT touch `~/Projects/AiPlus/aiplus-public/` main worktree.
- Push to `feat/sec-1` only. Advisor handles main merge after Phase 3 ratification.

Shared-file ownership:
- This session OWNS: `crates/aiplus-cli/src/agent/route.rs`, new `crates/aiplus-cli/src/agent/audit.rs`, `crates/aiplus-cli/src/agent/commands.rs` (new subcommands), `crates/aiplus-cli/src/agent/doctor.rs`, `crates/aiplus-cli/src/identity/*` (likely new module for hardware signing setup), new tests in `crates/aiplus-cli/tests/`
- This session must NOT modify: `adapters/CONTRACT.md` (FROZEN), `crates/aiplus-cli/src/agent/coordinator.rs` scoring rubric (calibration-locked), `crates/aiplus-cli/tests/fixtures/coordinator_calibration.toml` existing 26 entries, any adapter code

## Scope — 3 sub-features

### Sub-feature 1: Tamper-evident dispatch-log (D1)

**What to ship**:
1. Each new `coordinator_decision` event in `dispatch-log.jsonl` includes a `prev_hash` field — sha256 of the prior event's canonical-JSON serialization.
2. First-ever entry includes `genesis: true` marker (no `prev_hash` needed; verification walks forward from genesis).
3. New `aiplus agent audit verify-log` subcommand walks the chain:
   - Per entry: hash prior entry's bytes, compare to current entry's `prev_hash`
   - Output: `VERIFY_LOG=PASS` (all chain links match) or `VERIFY_LOG=FAIL line=N reason=<hash mismatch | malformed entry | missing prev_hash>`
4. Canonical JSON: lexicographic key order, no whitespace, UTF-8 — deterministic regardless of serde feature flags. Either use `serde_jcs` crate or hand-roll the canonical pass.
5. Pre-genesis entries (those that exist now in dispatch-log.jsonl from prior dispatches) are "legacy", out of audit scope. Genesis is the first new entry after this feature ships.
6. `aiplus doctor` adds INFO line: `dispatch_log_chain=valid` or `dispatch_log_chain=BROKEN line=N`.

**Why first**: smallest blast radius (only route.rs + new audit.rs + new subcommand + new test). No adapter touch. No CLI flag plumbing. Verification is read-only.

### Sub-feature 2: Cross-provider auditor (D2 — DESIGN.md §22 Layer 7)

**What to ship**:
1. New CLI flag: `aiplus agent route --auditor-provider <provider-name> "<task>"`.
   - `<provider-name>` is one of the configured adapter aliases (`claude-code`, `codex`, `opencode`, etc.)
   - MUST be different from the primary provider that would have run the task — Phase 1 design defines how to detect (compare `provider` field on AdapterResult, not just alias name).
2. Auditor workflow:
   - Primary adapter dispatches as today (per coordinator's tier + staffing decision)
   - After primary completes, route the primary's `final_text` + `tool_calls` summary + original task to the auditor provider
   - Auditor receives a structured "review this output" prompt and returns a verdict: `agree` / `disagree` / `flag`, plus reasoning
3. Auditor verdict written to dispatch-log as new event type: `auditor_verdict`. Schema:
   ```json
   {"event":"auditor_verdict","decisionId":"<links to coordinator_decision>","auditor_provider":"codex","primary_provider":"claude-code","verdict":"disagree|agree|flag","reasoning_summary":"<200 char excerpt>","prev_hash":"<sha256>"}
   ```
4. Auditor runs **sequentially after primary** (not in parallel with coordinator_batch peers). Adds 5-30s latency per audited task. Documented in `--auditor-provider --help`.
5. New test: deliberately ambiguous task → auditor verdict captures disagreement.
6. `aiplus doctor` adds INFO line: `auditor_provider_configured=<provider>` or `disabled`.

**Why second**: touches route.rs (auditor dispatch step), needs to call adapter directly, requires Phase 1 design for verdict schema + prompt template. More complex than #1 but smaller than #3.

### Sub-feature 3: Hardware-backed commit signing (D3)

**What to ship**:
1. New subcommand: `aiplus identity setup-signing` (or `aiplus identity install-secure-enclave-key`).
2. Detection:
   - macOS → use Secure Enclave via `ssh-keygen -t ecdsa-sk` (Apple's Secure Enclave is exposed as a hardware-backed FIDO2 device)
   - Linux/Windows → emit clear "not supported on this platform; see docs for YubiKey path"
3. Setup steps (on macOS):
   - `ssh-keygen -t ecdsa-sk -O resident -O verify-required -C aiplus-secure-enclave -f ~/.ssh/id_ecdsa_sk_aiplus`
   - `git config --global gpg.format ssh`
   - `git config --global user.signingkey ~/.ssh/id_ecdsa_sk_aiplus.pub`
   - `git config --global commit.gpgsign true`
   - Set `allowed_signers` file for `git log --show-signature` verification
4. Documentation in subcommand `--help` + docs/ on:
   - What got configured
   - How to verify a signed commit: `git log --show-signature -1` should show "Good signature"
   - How to unwind: `git config --global --unset commit.gpgsign`
   - YubiKey alternative path (briefly, with link to upstream docs)
5. Phase 1 includes: detect existing signing config (don't clobber if Owner already has GPG signing); detect non-macOS (fail with clear message); test that signing actually produces verifiable commits.
6. `aiplus doctor` adds INFO line: `commit_signing=secure_enclave|gpg|none`.

**Why last**: highest platform-specific complexity; mostly subprocess shell-out to `ssh-keygen` and `git config` but lots of edge cases (existing signing config, allowed_signers file management, ssh-agent integration). Better isolated after #1 + #2 are stable.

## Phase structure

### Phase 1 — `docs/proposals/sec-1-impl-notes.md` BEFORE coding

Required sections:

1. **Hash chain format** — exact serialization (which fields go into hash, ordering, escape rules), genesis marker design, verification algorithm, pre-genesis cutoff policy.
2. **Auditor workflow** — verdict schema (above is starting point; CEO finalizes), prompt template, different-provider detection mechanism, sequential-after-primary execution model, failure semantics (auditor errors → primary still succeeds).
3. **Secure Enclave setup** — exact `ssh-keygen` command + flags, git config keys, allowed_signers file location, idempotency (re-running setup-signing doesn't double-configure), detection of existing config.
4. **Test plan** — per-feature integration tests + one end-to-end "tamper detection" test (manually edit dispatch-log line → verify-log FAILs at that line).
5. **CHANGELOG ## 0.6.4 draft** — non-programmer friendly per Owner direction.

### Phase 2 — Impl 1 → 2 → 3 in order. Each ships with own integration test before next.

### Phase 3 — Evidence

- All 352+547 existing tests + new tests green
- End-to-end demos:
  - `aiplus agent audit verify-log` PASS on fresh dispatch-log
  - Manual tamper → `verify-log` FAIL with line number
  - `aiplus agent route --auditor-provider codex "<task>"` produces auditor_verdict event
  - `aiplus identity setup-signing` + subsequent commit signs verifiable via `git verify-commit`
- Evidence trailer in `sec-1-impl-notes.md`

## STOP rule (day-0 narrow, per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** — existing 352+547 test fails → STOP + ping Advisor
- **STOP only on `_SCOPE-BREACH_`** — touched CONTRACT/adapter/scoring/fixture-existing-26 → revert
- **Do NOT STOP on `_IMPL-BUG_`** — your new code has bug, fix and continue
- **Platform STOP** — if Phase 2 #3 (hardware signing) on non-Mac, emit clear error not panic. Don't STOP the goal; #3 degrades gracefully.

### Retry-once gate (mandatory)

Any first-attempt test FAIL → retry once with identical command + env → if retry PASS, record PASS-after-flaky-retry, continue. Especially important for #2 (LLM-based auditor verdict has non-determinism) and #3 (ssh-keygen interactive prompts).

## Conformance / verification

- All ACs in goal doc met
- 352+547 existing + new tests PASS
- Hash chain genuinely catches tampering (test deliberately mutates entry → FAIL)
- Auditor produces non-trivial signal (test with deliberately ambiguous task)
- Secure Enclave signing produces commits that `git verify-commit` accepts
- `aiplus doctor` PASS unchanged for default config

## Scope fence

- **IN**: `route.rs`, new `audit.rs`, new `identity/*` module if needed, `commands.rs`, `doctor.rs`, new tests
- **OUT**:
  - `adapters/CONTRACT.md` (FROZEN)
  - Adapter code
  - `coordinator.rs` scoring logic
  - Calibration fixture existing 26 entries
  - `aiplus-cli/Cargo.toml` version bump (Advisor handles)
  - `CHANGELOG.md` actual edit (Advisor handles, CEO drafts in notes)
  - `install.sh` fallback version (Advisor handles)
  - YubiKey integration code (documentation only)
- **GRAY**: small comment improvements in touched files OK.

## Deliverables (7)

1. `docs/proposals/sec-1-impl-notes.md` — Phase 1 design + Phase 3 evidence
2. `crates/aiplus-cli/src/agent/audit.rs` — hash chain + verify-log
3. `crates/aiplus-cli/src/agent/route.rs` — `prev_hash` emission + `--auditor-provider` flag + auditor dispatch step
4. `crates/aiplus-cli/src/agent/commands.rs` — new subcommand wiring
5. `crates/aiplus-cli/src/agent/doctor.rs` — 2 new INFO lines (chain + signing)
6. `crates/aiplus-cli/src/identity/setup_signing.rs` (or similar) — Secure Enclave configuration
7. `crates/aiplus-cli/tests/sec_1_tamper_evident_smoke.rs`, `sec_1_auditor_smoke.rs`, `sec_1_signing_smoke.rs` — three integration test files OR extension of existing tests (CEO's organizational choice; per Advisor lesson from v0.3.1, organizational is non-binding)

## Time ceiling

**Hard ceiling: T+7d** (1 week). Faster expected based on prior goals' under-budget pattern; budget here is conservative because security is platform-specific.

## Handoff endpoint

After Phase 3 evidence committed to `feat/sec-1` and pushed:

- Report back with one of three verdicts:
  - `IMPL_VERDICT=PASS` — all ACs met; all tests green; live demos work
  - `IMPL_VERDICT=PASS_WITH_DEVIATIONS` — ACs met but design deviated from spec; Advisor decides accept
  - `IMPL_VERDICT=BLOCKED` — `_REGRESSION_` or `_SCOPE-BREACH_` after retry
- Phase 3 summary in impl-notes
- Note any platform-specific behavior (D3 macOS only by design)

Advisor verifies + ff-merges + bumps aiplus-cli to v0.6.4 + writes CHANGELOG + writes retrospective + **closes the 4-goal program**.

## Owner gates (always active)

- NO push to `origin/main`. Push only `feat/sec-1`.
- NO `adapters/CONTRACT.md` edits (FROZEN).
- NO scoring rubric changes.
- NO calibration fixture existing 26 entries changes.
- NO global config edits beyond what `aiplus identity setup-signing` explicitly does for the Owner (and only after Owner runs the subcommand).
- NO force ops, NO `--amend`, NO hook bypass.
- BWS-backed secret broker: wrap with `aiplus secret-broker run --aliases anthropic,openai -- <cmd>`. NOT `secret-broker need`.

**Special note for D3**: `aiplus identity setup-signing` DOES modify the Owner's git global config (gpg.format / user.signingkey / commit.gpgsign). This is the Owner-authorized exception — the subcommand exists precisely to do this. But: detect existing config; don't clobber without asking; print exactly what will be changed before doing it; offer `--dry-run` flag to preview.

## Dependencies

- v0.3 coordinator parallelism shipped ✓ (`aiplus-public@1a9aae5`)
- CONTRACT v1.1 FROZEN ✓ (`aiplus-agent-team@711df8b`)
- Mac Secure Enclave (Owner Darwin 25.4.0 ✓)
- Multiple adapter providers installed (claude-code + codex + opencode) — needed for cross-provider auditor

---

— Advisor, 2026-05-18, against CONTRACT v1.1 FROZEN. Day-0 narrow STOP + retry-once gate per `advisor_briefing_stop_rules` auto-memory. Final goal in the 4-goal program.
