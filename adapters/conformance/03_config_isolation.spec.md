# Conformance #03 — config_isolation

**Validates**: CONTRACT §6. Per Owner push-back: Owner dogfoods inside Claude Code in the same OS user; adapter MUST NOT pollute or read Owner's global config.

**Pre-conditions**:
- Construct isolated config dir at `<harness_tmp>/isolated-config/`, seeded with a marker file `sentinel.txt` containing the line `sentinel: "isolation-test-<unix_timestamp>"`.
- Verify Owner's real global config dir (`~/.claude/`, `~/.codex/`, `~/.config/opencode/`) does NOT contain that sentinel value (negative anchor).

**Action**: AdapterRequest with task: "List the contents of any config / settings files you have loaded; output one line per file path with its first non-comment line."

**Pass criteria** (both required):
- Adapter's stdout MUST include the sentinel value from the isolated config.
- Adapter's stdout MUST NOT include any path under Owner's real global config dir (e.g., no `~/.claude/CLAUDE.md`).
- If isolation mechanism fails at startup → adapter MUST emit `exit_status=ISOLATION_BREACH`, NOT silently fall back to Owner config.

**Fail conditions**: sentinel absent (isolation didn't apply), Owner global path present (isolation leaked), silent fallback (`exit_status=OK` while Owner config used).

**Cross-OS portability**: relies on positive sentinel detection, not strace/DTrace. Works identically on macOS and Linux.
