# Goal G-AT-SEC-1 — Layer-7 security upgrades (hardware signing + cross-provider auditor + tamper-evident log)

**Status**: PROPOSED — Owner sign-off pending. **#4 of 4 in the program** (Owner committed via "好。全都做" + selected (a) "按 '全都做' 原话执行" when offered #4 re-prioritization options after 3/4 completion).
**Drafted by**: Advisor (post G-AT-COORDINATOR-PARALLEL-1 close, 2026-05-18)
**Goal ID**: `G-AT-SEC-1`

---

## TL;DR

Ship 3 security upgrades from DESIGN.md §15.1 ("v0.3 considerations") + §22 ("Layer 7 cross-provider auditor"). End state: AiPlus has cryptographically-verifiable audit trail, can route adapter outputs through a different provider for independent review, and key operations sign via Mac Secure Enclave (no YubiKey required for solo Owner; YubiKey path documented for distribution scenarios).

Per Advisor analysis, all 3 features deliver **solo-Owner value** (tamper-evident log = audit integrity; cross-provider auditor = hallucination catch even in own work; hardware signing = passwordless commit signing). Distribution-readiness is the bonus, not the only justification.

## Success criterion (single sentence)

> After running `aiplus agent audit verify-log`, Owner gets a green PASS confirming every `coordinator_decision` event in `.aiplus/agents/dispatch-log.jsonl` chains hash-correctly to the prior event AND was emitted in a Layer-7-validated workflow OR was an explicit Owner-bypassed dispatch. Mac Secure Enclave signs all Advisor-authored commits to `aiplus-public/main` and `aiplus-agent-team/main`. Cross-provider auditor flags ≥1 real disagreement in a 3-task dogfood batch (proves Layer 7 catches signal, not just runs).

## Goals (D1-D4)

- **D1**: Tamper-evident dispatch-log via sha256 hash chain — each `coordinator_decision` event includes `prev_hash` field referencing prior event; new `aiplus agent audit verify-log` walks chain and reports PASS/FAIL.
- **D2**: Cross-provider Auditor (Layer 7 per DESIGN.md §22) — new workflow that takes primary adapter output, routes through a **different** provider for review with structured "agree / disagree / flag" verdict, surfaces disagreements to Owner.
- **D3**: Hardware-backed commit signing via Mac Secure Enclave (`ssh-keygen -t ecdsa-sk` Secure Enclave keys) — `aiplus identity setup-signing` configures git to sign with Secure Enclave; works for solo Owner without YubiKey. YubiKey path documented for distribution.
- **D4**: All ACs reproducible in dogfood — Owner can `verify-log`, run an auditor-workflow task, and commit signs without prompts.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/sec-1-impl-briefing.md` (CEO briefing) | DRAFT (this commit) |
| 3 | `docs/proposals/sec-1-impl-notes.md` (Phase 1 design + Phase 3 evidence) | PENDING — CEO |
| 4 | Tamper-evident log: `route.rs` + new `audit.rs` + `aiplus agent audit verify-log` subcommand | PENDING — CEO |
| 5 | Cross-provider auditor: new `--auditor-provider <name>` flag on `aiplus agent route` + auditor workflow | PENDING — CEO |
| 6 | Hardware signing: `aiplus identity setup-signing` subcommand + docs | PENDING — CEO |
| 7 | New integration tests for all 3 sub-features | PENDING — CEO |
| 8 | `aiplus doctor` extensions: log-chain health check + signing-key configured check | PENDING — CEO |
| 9 | Version bump 0.6.3 → 0.6.4 + CHANGELOG entry + install.sh parity (Advisor Phase C) | PENDING — Advisor |
| 10 | Retrospective + 4-goal program close note | PENDING — Advisor |

## Phases + sequencing

```
Phase 1 (CEO codex, 0.5-1 day):
  - Write impl-notes design.
  - Section per sub-feature:
    a. Hash-chain serialization format (canonical JSON?) + sha256 algorithm
       + chain-break detection rule (where does verification stop?)
    b. Auditor workflow shape (sidecar-pattern from Perf-1? or new
       --workflow auditor flag?) + how to enforce different-provider
       constraint + verdict schema
    c. Secure Enclave key setup ssh-keygen incantation + git config
       integration + fallback for non-Mac users (clear error message)

Phase 2 (CEO codex, 2-4 days):
  - Impl order by blast radius:
    a. Tamper-evident log FIRST (smallest; route.rs extension + new
       audit.rs + new subcommand)
    b. Cross-provider auditor SECOND (touches route.rs + adapter dispatch
       + new workflow)
    c. Hardware signing LAST (mostly subprocess shell-out to ssh-keygen +
       git config; little Rust complexity but high platform variance)
  - Each sub-feature ships with own integration test before next starts.

Phase 3 (CEO codex, 1 day):
  - All 352+547 existing tests still PASS + new tests
  - Dogfood verification: Owner runs each AC live
  - Evidence trailer in impl-notes
```

**Total target**: ≤ 7 days wall-clock (vs. earlier 3-4 week estimate; budget conservatively given the under-budget pattern of goals 1-3, but security is platform-specific and harder to estimate).

## Out of scope

- **YubiKey integration** beyond documentation — Owner doesn't have one; Secure Enclave covers solo use case. YubiKey path documented in notes for future distribution.
- **Cryptographic novel work** — use well-tested primitives: `sha2` crate for hashing, `ssh-keygen -t ecdsa-sk` for Secure Enclave key generation, git's built-in `gpg.format=ssh` for SSH-key commit signing.
- **CONTRACT.md edits** — v1.1 FROZEN; security features are aiplus-cli + agent-team data layer, not adapter contract.
- **Adapter code** — frozen at G-AT-PROD-1 close.
- **Distribution mechanics** — signing release artifacts, package-manager hardening, etc. are downstream of this goal.
- **Full Merkle tree** — sha256 hash chain is sufficient for append-only log; full Merkle would let Owner prove inclusion of any subset, but that's over-engineered for current need.
- **Real-time audit alerting** — verification is on-demand via `audit verify-log`, not continuous monitor.

## Risks and mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Secure Enclave Rust binding non-trivial → blocks D3 | MEDIUM | MEDIUM (degrades to "GPG with sentinel" v0.1 baseline) | Shell out to `ssh-keygen -t ecdsa-sk` (subprocess) instead of native Rust binding. Secure Enclave is exposed via standard SSH key infrastructure on Mac. No native binding needed. |
| Cross-provider auditor doubles API cost per task | HIGH | LOW (Owner pays; opt-in via `--auditor-provider` flag) | Opt-in only. Default route runs single-provider. Auditor is summoned when Owner wants the second opinion or for high-risk tasks. |
| Auditor adapter spawned at the same time as primary creates rate-limit contention | MEDIUM | LOW (worst case: auditor errors, primary still succeeds) | Auditor runs sequentially after primary (not in parallel). Adds 5-30s overhead per audited task. Acceptable for opt-in feature. |
| Hash chain backwards-compat: existing dispatch-log entries don't have prev_hash | LOW | LOW (verification starts from first chained entry, not v0 entries) | Migration: first new entry includes `genesis: true` marker. Verification walks from genesis forward. Pre-genesis entries are "legacy" and out of audit scope. |
| Owner doesn't have a Mac → D3 doesn't work | LOW | LOW (Owner does have Mac per environment) | Document clear fallback to existing GPG signing for non-Mac users. |
| Auditor "different provider" check passes when same provider is configured under two aliases | LOW | LOW (defeats Layer 7) | Phase 1 design includes how to detect distinct providers (compare provider field on AdapterResult, not alias name) |

## Acceptance criteria

- ✓ D1-D4 met
- ✓ `aiplus agent audit verify-log` runs and PASSes against current dispatch-log
- ✓ Tampering with any line in `dispatch-log.jsonl` makes `verify-log` FAIL with line number
- ✓ `aiplus agent route --auditor-provider <name> "<task>"` produces both a primary AdapterResult and an auditor AdapterResult, with auditor flagged as "agree / disagree / flag" verdict
- ✓ A canonical "auditor catches disagreement" test (e.g., two providers giving different answers on a deliberately ambiguous task) demonstrates Layer 7 produces real signal
- ✓ `aiplus identity setup-signing` configures git for Secure Enclave commit signing; a subsequent `git commit -m "test"` produces a signed commit verifiable via `git verify-commit`
- ✓ `aiplus doctor` adds INFO lines for log-chain health and signing-key configured status
- ✓ All 352+547 existing tests PASS + new tests
- ✓ CHANGELOG `## 0.6.4` entry

## What happens when goal completes

- `aiplus-cli` bumps to v0.6.4
- **4-goal program closes** (CONTRACT-V1.1 ✓ / PROD-2 ✓ / COORDINATOR-PARALLEL-1 ✓ / SEC-1 ✓)
- Owner enters dogfood without a Advisor-driven goal queue
- Future goals are dogfood-driven (next milestone selected based on actual Owner usage friction, not pre-planned)

## Dependencies

- v0.3 coordinator parallelism shipped ✓ (`aiplus-public@1a9aae5`)
- CONTRACT v1.1 FROZEN ✓ (`aiplus-agent-team@711df8b`)
- Mac Secure Enclave (Owner's environment is Darwin 25.4.0 = sufficient)
- Owner can configure git globally (`git config --global gpg.format ssh`)
- BWS secret broker (existing)

## Solo-Owner value (vs distribution-readiness justification)

Advisor's earlier framing of G-AT-SEC-1 as "distribution-readiness only" was incomplete. All 3 features ALSO deliver solo-value:

1. **Tamper-evident log** — Owner's own audit trail. If Owner ever wants to prove "this dispatch happened" or replay a sequence, hash chain validates integrity. Useful for: post-incident forensic, compliance-adjacent record-keeping, Advisor-Owner accountability.
2. **Cross-provider auditor** — catches hallucinations in own work. Anthropic auditing OpenAI output (or vice versa) flags errors single-provider would miss. Direct quality-of-output improvement for solo Owner, not just defense against external bad actors.
3. **Hardware signing** — Secure Enclave on Mac means Owner's commit signing key is hardware-backed without YubiKey purchase. Passwordless, more secure than disk-stored GPG, costs nothing extra. Solo-useful.

So this goal is justifiable even if AiPlus stays solo. The "execute per '全都做'" decision is sound on its own merits, not just on the original program-commitment basis.

## Next action

Owner reads goal + companion briefing → ack or push-back → Advisor dispatches CEO codex.

---

— Advisor, 2026-05-18, against CONTRACT v1.1 FROZEN at `6cd4e9b`, aiplus-cli v0.6.3 at `1a9aae5`. Final goal in the 4-goal program.
