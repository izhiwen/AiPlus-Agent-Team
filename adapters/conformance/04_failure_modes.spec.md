# Conformance #04 — failure_modes

**Validates**: CONTRACT §3 (exit_status enum) + §7 (Owner-interrupt protocol).

Each sub-case sends a deliberately broken request OR triggers a specific signal sequence; adapter MUST emit the expected `exit_status` and respect `partial` semantics.

**04a — WORKSPACE_NOT_PROVISIONED**
- Pre: pass `worktree_path` to a directory that exists but is NOT a git worktree.
- Pass: `exit_status=WORKSPACE_NOT_PROVISIONED`, no spawn attempted (verify no runtime process spawned during 2s window), `partial=true`.

**04b — ISOLATION_BREACH**
- Pre: deliberately break isolation config (e.g., point `--settings` to a non-existent file in a runtime that rejects this).
- Pass: `exit_status=ISOLATION_BREACH`, adapter exits fast (< 3s), no Owner global config touched.

**04c — RUNTIME_NOT_FOUND**
- Pre: PATH stripped of the runtime binary (e.g., `PATH=/nonexistent`).
- Pass: triggers fallback (see #06), `exit_status=RUNTIME_NOT_FOUND`, exit code 75.

**04d — OWNER_INTERRUPTED via SIGINT**
- Action: dispatch long task ("count slowly to 100, one number per second"). Send SIGINT after 5s.
- Pass within 5s of signal: `exit_status=OWNER_INTERRUPTED`, `partial=true`, `stdout_raw` contains numbers up to interrupt point (not empty).

**04e — OWNER_INTERRUPTED via SIGTERM escalation**
- Action: dispatch long task. Send SIGINT; if runtime hasn't exited in 5s, adapter sends SIGTERM internally (test harness observes adapter→runtime signal via process tree).
- Pass: same as 04d, plus verify SIGTERM was sent before SIGKILL.

**04f — OWNER_HARD_INTERRUPTED via SIGKILL to adapter**
- Action: dispatch long task. Send SIGKILL to the adapter process directly.
- Pass: no AdapterResult on stdout (process killed before emit), parent (CEO harness) sees wait-status indicating SIGKILL, trace file has the AdapterRequest entry written (proves request was received before kill).

**Notes**: 04d/04e/04f cover Owner-kill survival per DESIGN.md §14. Signal sequence: **SIGINT @ T+0 → SIGTERM @ T+5s → SIGKILL @ T+10s** (5s more after SIGTERM). Authoritative source is CONTRACT §7; this spec mirrors it. If they disagree in the future, CONTRACT wins.
