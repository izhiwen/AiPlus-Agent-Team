# AiPlus Agent Team — Adapter Contract v0

**Status**: **v1 FROZEN** (validated by Claude Code spiral round 1 at commit `d04797e` and OpenCode spiral round 2 at commit `90500cb`; zero `_CONTRACT-BUG_` painpoints across both rounds — both rounds' painpoints classified as runtime-specific, addressed in adapter IMPLEMENTATION.md or upstream runtime fixes). Future contract changes require explicit Owner-authorized version bump and a new spiral round on at least one adapter.
**Frozen on**: 2nd adapter pass-through of conformance suite, not earlier.
**Pilot adapter**: Claude Code (`adapters/claude-code/`).
**Non-goals**: transport choice, per-arg tool granularity, cross-runtime parallelism, output streaming — out of v0 scope.

This contract defines the schema each adapter MUST satisfy. Transport (subprocess + JSON pipe / HTTP / PTY) and runtime-specific mechanisms (cwd injection flag, config isolation flag composition, token-extraction parser) are per-adapter, documented in `adapters/<runtime>/IMPLEMENTATION.md`. _A contract without a conformance suite is a PDF._ See `adapters/conformance/`.

---

## 1. `AdapterRequest` (CEO → Adapter)

All fields mandatory. Schema validation failure → `exit_status=ADAPTER_REQUEST_INVALID`.

```json
{
  "schema_version": "1.0",
  "session_id": "<UUIDv7>",
  "role": "engineer-a",
  "task": "<free-text>",
  "worktree_path": "<absolute path>",
  "memory": { "personal": "...", "team": "...", "project": "..." },
  "system_prompt": "<role persona, resolved from .toml + persona.md>",
  "allowed_tools": ["Bash", "Edit", "Read", "..."],
  "ttl_seconds": 1800,
  "continuation_hint": "new | continue",
  "override_dirty_worktree": false
}
```

- `worktree_path` — absolute. CEO resolves `{project_name}` template before dispatch.
- `memory` — already merged by CEO per §8 read order (personal < team < project). Adapter does NOT re-merge.
- `allowed_tools` — coarse list (lowest common denominator). Per-arg patterns (e.g. Claude's `"Bash(git *)"`) are runtime-native extensions, documented per-adapter; CONTRACT does not require them.
- `continuation_hint=continue` only valid if `session_id` has prior mapping entry (§4); else adapter MUST treat as `new`.

## 2. `AdapterResult` (Adapter → CEO)

5 mandatory fields. `null` valid for `tool_calls` and `usage_tokens`; never throw on unfillable.

```json
{
  "schema_version": "1.0",
  "session_id": "<echoed UUIDv7>",
  "stdout_raw": "<runtime's verbatim stdout>",
  "tool_calls": [{"tool": "Bash", "args_summary": "git status", "outcome": "ok|err"}] | null,
  "final_text": "<last user-visible text reply; tool-call outputs not included, those go to tool_calls[]>",
  "usage_tokens": {"input": 1234, "output": 567, "total": 1801} | null,
  "exit_status": "<see §3>",
  "partial": false
}
```

`partial=true` whenever the runtime did not naturally complete (interrupt, runtime crash, schema-validation error). CEO uses `partial` to decide retry vs use-as-is.

When `tool_calls` is non-null, each entry MUST have all three fields: `tool` (string, runtime-native tool name), `args_summary` (free-text ≤ 200 chars; longer values truncated with `...` suffix), `outcome` (`"ok"` or `"err"`).

Adapter MUST emit AdapterResult even on failure paths (e.g., `WORKSPACE_NOT_PROVISIONED` still emits a JSON with `partial=true` and other fields null where unknowable).

## 3. Exit status enum

| code | meaning |
|---|---|
| `OK` | runtime completed naturally |
| `RUNTIME_FAILURE` | runtime errored (non-zero, model error, network fail) |
| `WORKSPACE_NOT_PROVISIONED` | worktree verify failed (§5) |
| `ISOLATION_BREACH` | config isolation failed (§6) |
| `ADAPTER_REQUEST_INVALID` | AdapterRequest schema violation — CEO bug |
| `OWNER_INTERRUPTED` | SIGINT received, or SIGTERM after grace period |
| `OWNER_HARD_INTERRUPTED` | SIGKILL — parent infers via wait-status, no AdapterResult on stdout |
| `RUNTIME_NOT_FOUND` | runtime binary not on PATH → triggers print-prompt fallback (§7) |
| `UNKNOWN_RUNTIME_FAILURE` | catch-all; requires patch to expand mapping |

## 4. Session ID + cross-call continuity

CEO generates `session_id` (UUIDv7) at dispatch. Adapter maintains mapping at `.aiplus/agent-team/sessions/<adapter>.json`:

```json
{
  "schema_version": "1.0",
  "entries": {
    "<UUIDv7>": {
      "runtime_session_id": "<runtime-internal id>",
      "created_at": "<ISO8601>",
      "last_used_at": "<ISO8601>",
      "role": "engineer-a"
    }
  }
}
```

- First dispatch with new UUIDv7 → adapter starts new runtime session, records mapping.
- Repeat dispatch → adapter looks up `runtime_session_id`, passes to runtime via native flag (Claude `--resume`, Codex `resume`, OpenCode `-s`).
- **Aging**: entries with `last_used_at > 24h` MAY be purged (per-adapter policy, documented in IMPLEMENTATION.md §6). Entries with `last_used_at > 30 days` MUST be purged. Mapping file SHOULD stay ≤ 10 MB; when exceeded, adapter purges oldest-first until under threshold.
- **Corruption**: JSON parse fail → adapter drops the file, CEO cold-starts with new session. NEVER silently continue with broken mapping. Surfaced via `aiplus agent doctor`.

### 4.3 Runtime-session-ID discovery fallback

If adapter cannot discover the runtime's internal session ID after spawn (runtime doesn't emit one, or emits in unparseable form), adapter MAY use the CEO `session_id` (UUIDv7) as the `runtime_session_id` in the mapping. This allows continuity to work even when the runtime is opaque about session identity. Per-adapter behavior documented in IMPLEMENTATION.md §6.

## 5. Working directory

- `worktree_path` MUST be absolute.
- Adapter MUST verify, before any spawn:
  1. directory exists,
  2. `git -C <path> worktree list` lists this path,
  3. working tree clean (no uncommitted `M`) OR `override_dirty_worktree=true`.
- Failure → `exit_status=WORKSPACE_NOT_PROVISIONED`, no spawn attempted.
- Adapter does NOT auto-create worktrees. That is CEO's responsibility via `aiplus agent integrate / prune-worktrees`.
- Adapter injects cwd via per-adapter mechanism (Appendix A).

## 6. Config isolation

Adapter MUST NOT inherit Owner's global config directory (`~/.claude/`, `~/.codex/`, `~/.config/opencode/`, etc.). Isolation mechanism is per-runtime; see Appendix A.

### 6.1 Auth-channel taxonomy

| Channel | Allowed? | Notes |
|---|---|---|
| Config-dir inheritance | **FORBIDDEN** | includes `~/.claude/`, `~/.codex/`, `~/.config/opencode/`, keychain reads via global config |
| Env-var credentials | **ALLOWED** | e.g., `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`. Adapter subprocess env restricted to `PATH` + named credential var(s); all other env stripped |
| `apiKeyHelper` via `--settings` (Claude Code) or per-runtime equivalent | **ALLOWED** | settings file must itself be isolated, NOT Owner-global |

Each adapter's `IMPLEMENTATION.md` §2 MUST document: (a) which channels it uses, (b) how CEO provides credentials. **Recommended provisioning pattern**: `aiplus secret-broker run --aliases <alias> -- <adapter cmd>` — backend-agnostic (works for keyring AND BWS). NOTE: `aiplus secret-broker need <alias>` is keyring-only and will FAIL on BWS or other non-keyring backends — do NOT recommend it as the default pattern.

### 6.2 Validation requirement

Adapter MUST validate its isolation artifacts (settings file, MCP config, plugin dir) BEFORE spawn — runtimes like Claude Code's `--print` may silently ignore invalid settings, hiding isolation breaches behind a successful spawn. Validation includes JSON parse + adapter-marker presence + writability check. Validation failure → `exit_status=ISOLATION_BREACH`, no spawn attempted.

### 6.3 Sentinel attestation

Conformance #03 validates isolation via positive sentinel test: harness seeds isolated config with `sentinel: "isolation-test-<timestamp>"`, adapter spawns runtime instructed to dump loaded config, assertion checks sentinel present AND Owner's known global markers absent.

### 6.4 Failure mode

Failure to isolate → adapter MUST fail-fast at startup with `exit_status=ISOLATION_BREACH`. Silent fallback to Owner config is forbidden.

## 7. Owner-interrupt protocol

Adapter installs SIGINT and SIGTERM handlers. Standard Unix two-stage:

| Signal sequence | Adapter behavior | Result exit_status |
|---|---|---|
| 1st SIGINT (Ctrl-C) at T+0 | adapter requests runtime stop, flushes stdout into `stdout_raw`, sets `partial=true`. **The 5s timer is the runtime-exit deadline (adapter→runtime SIGTERM escalation window); adapter's own emit/trace timeline is NOT bounded by this window.** | `OWNER_INTERRUPTED` (if flush succeeds) |
| Runtime has not exited by T+5s | adapter sends SIGTERM to runtime | (transitions) |
| Runtime has not exited by T+10s (5s more after SIGTERM), OR 2nd SIGINT within 3s of 1st | adapter sends SIGKILL to runtime | `OWNER_INTERRUPTED` if best-effort flush succeeded; else parent-infers `OWNER_HARD_INTERRUPTED` |
| SIGKILL received by adapter itself | parent (CEO) infers from wait-status | `OWNER_HARD_INTERRUPTED`; no AdapterResult written to stdout; trace file's last AdapterRequest is recoverable |

Partial stdout buffer is ALWAYS retained (`partial=true`) — never silently discarded.

## 8. Print-prompt fallback (v0.2 trust anchor)

If `RUNTIME_NOT_FOUND` or any unrecoverable spawn failure → adapter MUST print:

```
AIPLUS_ADAPTER_FALLBACK=print-prompt
session_id: <UUIDv7>
role: <role>
worktree: <abs path>
---
<full system_prompt + memory blocks + task, copy-pasteable>
---
Action: paste above into <runtime> manually; report result via `aiplus agent report-manual <session_id>`.
```

Exit code 75 (POSIX `EX_TEMPFAIL`). This is v0.2's commitment: installing the adapter is never worse than v0.1.

## 9. Trace output

Adapter MUST append `(AdapterRequest, AdapterResult)` to `.aiplus/agent-team/adapter-trace/<session_id>.jsonl`, project-local.

- Path is project-local (sibling to `sessions/<adapter>.json`). NOT `~/.aiplus/` — aligns with §15 boundary.
- Trace entries MUST pass through `aiplus-agent-memory`'s current redaction pipeline before write. Redaction failure on one entry → skip that entry, DO NOT block dispatch. Surface as `aiplus agent doctor` warning.
- **Rotation**: rotate when current file > 100 MB OR oldest entry > 7 days. Threshold values are conformance-tested (#04 sub-case verifies rotation).
- **Kill switch**: `aiplus agent trace --disable` writes `.aiplus/agent-team/trace-disabled` sentinel; adapter MUST honor (no-op on trace write). Re-enable via `--enable`.

## 10. AdapterResult emission

`--output-file <path>` is an adapter CLI flag (NOT a JSON field in AdapterRequest). Two emission modes:

- **Without `--output-file`** (default): adapter writes AdapterResult as the FINAL JSON line of stdout. Lines before it are the runtime's raw output.
- **With `--output-file <path>`** (CEO chooses for clean log separation): adapter writes AdapterResult to `<path>` only; stdout contains only the runtime's raw output, no trailing AdapterResult JSON.

Schema validation MUST pass on every emission, including failure paths (`WORKSPACE_NOT_PROVISIONED` etc. still emit valid JSON with `partial=true` and unknowable fields null). Schema validation failure on emission → log to trace + exit code 75 (treated as `RUNTIME_NOT_FOUND` → fallback applies).

## 11. Schema version migration

Adapter handles `schema_version` mismatch between AdapterRequest and adapter's own pinned support:

- **Major mismatch** (adapter at `1.x`, request `2.x` or vice versa) → reject with `exit_status=ADAPTER_REQUEST_INVALID`.
- **Minor forward-compat** (adapter at `1.0`, request `1.1`) → MAY accept, ignoring unknown fields. MUST log a `schema_forward_compat_used` warning entry to trace.
- **Downgrade** (adapter at `1.1`, request `1.0`) → reject with `ADAPTER_REQUEST_INVALID`. Mirrors session-mapping behavior in §4 / conformance 07e.

AdapterResult's `schema_version` is always the adapter's own pinned version, NOT echoed from request.

---

## Appendix A — Runtime capability speed-table (Step 1, 2026-05-18)

| Dimension | Claude Code v2.1.143 | Codex v0.130.0 | OpenCode v1.15.3 |
|---|---|---|---|
| **Non-interactive** | `claude -p / --print` | `codex exec` | `opencode run` |
| **JSON output** | `--output-format json / stream-json` + `--json-schema` (richest: hook events, partial messages, structured tool calls) | `--json` JSONL + `--output-schema <file>` + `--output-last-message <file>` | `--format json` raw events |
| **cwd injection** | subprocess `cwd=...` (standard Unix) | `-C, --cd <DIR>` flag | `[project]` positional arg |
| **Tool scope** | `--allowedTools "Bash(git *) Edit"` + `--disallowedTools` (per-tool + per-arg, finest) | `--sandbox` policy levels (coarse) | `--dangerously-skip-permissions` (binary on/off, coarsest) |
| **Config isolation** | `--bare --settings <isolated.json> --mcp-config <iso> --agents '{}' --plugin-dir <empty>` (multi-flag composition) | `-c key=value` per-key + `--profile <name>` + likely `CODEX_HOME` env (verify) | `--pure` + likely `XDG_CONFIG_HOME` env (verify) |
| **Session continue** | `-c, --continue`, `--resume`, `--fork-session` | `resume` subcommand + `--last` | `-c, --continue`, `-s, --session`, `--fork` |
| **Token usage** | Stream-json includes usage per turn | JSONL events include token counts | `opencode stats` separate (per-call may be absent) |

## Appendix B — `adapters/<runtime>/IMPLEMENTATION.md` required sections

Each runtime's IMPLEMENTATION.md MUST document:

1. **Transport** — subprocess + JSON pipe / HTTP / PTY / other
2. **Config isolation** — exact flag composition or env vars used
3. **cwd injection** — exact mechanism (flag / positional / subprocess cwd)
4. **Tool whitelist translation** — how `allowed_tools` list maps to runtime-native form
5. **`usage_tokens` extraction** — where in runtime output, or "always null"
6. **Session continuation** — exact flag name for passing `runtime_session_id`
7. **Trace event mapping** — which runtime stdout events become AdapterResult fields

## Revisions

- v0 — initial draft. Spiral round 0.
- v0.1 — pre-implementation tighten pass: A1+A2 signal-timing aligned with conformance #04; B1-B6 (final_text semantics, tool_calls field requirements, mapping aging hard limits, --output-file flag clarity, §11 schema-version migration, redaction wording); C1 Codex `-C, --cd` confirmed in Appendix A. Still round 0.
- v1 — after Claude Code reference adapter spiral round 1 (commit `d04797e` + painpoints): added §4.3 runtime-session-ID discovery fallback; restructured §6 into 6.1 auth-channel taxonomy (FORBIDDEN config-dir / ALLOWED env-var / ALLOWED apiKeyHelper) + 6.2 isolation-validation requirement (catches runtimes that silently ignore invalid settings) + 6.3 sentinel attestation (unchanged from v0.1) + 6.4 failure mode (unchanged from v0.1). Recommended provisioning pattern is now backend-agnostic `secret-broker run` (NOT keyring-only `need`). Conformance specs `01-07.spec.md` unchanged — semantics didn't shift.
- **v1 FROZEN (this) — 2026-05-18.** Validated via OpenCode spiral round 2 (commit `90500cb`): conformance #01→#02→#03→#06→#04→#05→#07 produced 5 PASS + 2 PASS_WITH_OPENCODE_BUG + 0 FAIL, with both round-2 painpoints classified as `_OPENCODE-BUG_` (upstream OpenCode non-interactive tool-loop instability) — zero `_CONTRACT-BUG_`. Per the Frozen criterion below, the two-adapter validation is complete. No CONTRACT changes required; v1 enters stable phase.

### What Frozen means (and does not mean)

- **Means**: no `_CONTRACT-BUG_` painpoint was found across 2 independent runtime adapters' spiral conformance runs. CONTRACT v1 is the production contract for adapter implementations going forward; future adapters (Codex, future runtimes) SHOULD pass against v1 without contract amendments.
- **Does NOT mean**: v1 is bug-free, complete, or final-for-all-time. New runtimes with significantly different process models, security models, or output formats may surface gaps requiring v1.x patch or v2. Owner spot-checks of adapter behavior in real workloads remain the higher-authority signal.
- **To unfreeze**: any future adapter encountering a true `_CONTRACT-BUG_` follows the spiral protocol — STOP, log painpoint, return to Advisor → Owner-authorized version bump → new round.

### Frozen criterion (history)

- **Frozen** = v1 passes full conformance suite on 2 adapters with no `_CONTRACT-BUG_` painpoints. **Met by Claude Code round 1 + OpenCode round 2.**
