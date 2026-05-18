# CONTRACT v0.1 Painpoints

## [conformance #01] CONTRACT §6 silent on auth-provision channel

Where stuck: CONTRACT §6 forbids config-dir inheritance but does not name allowed auth channels. Claude Code `--bare` requires `ANTHROPIC_API_KEY` env or `apiKeyHelper` via `--settings`.

Why blocked: Literal §6 read allows env-var auth because env is not config-dir inheritance, but intent is unclear. First-time read of §6 says "isolate everything", so the adapter had no auth and returned `RUNTIME_FAILURE` on #01 with `apiKeySource:"none"` and `Not logged in · Please run /login`.

v1 fix required: §6 must add explicit auth-channel taxonomy:

- FORBIDDEN: config dir inheritance (`~/.claude/`, `~/.codex/`, `~/.config/opencode/`)
- ALLOWED: env-var credentials (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.)
- ALLOWED: `apiKeyHelper` via `--settings` for Claude Code or per-runtime equivalent
- Each `IMPLEMENTATION.md` §2 documents channels used and how CEO provides them

Workaround applied (Advisor-authorized 2026-05-18, v0.1 letter compliant): see `adapters/claude-code/IMPLEMENTATION.md` §2.

## [conformance #01 retry] Advisor unblock used keyring-only `secret-broker need`

Where stuck: Advisor's first unblock instruction said `aiplus secret-broker need anthropic --export-as ANTHROPIC_API_KEY`. Owner backend is BWS; `need` is keyring-only, so it returned `SECRET_NEED_STATUS=FAIL`.

Why blocked: Advisor instruction error. It assumed the default keyring backend and ignored Owner's BWS configuration.

v1 fix required (IMPLEMENTATION.md level, not CONTRACT): each adapter's `IMPLEMENTATION.md` §2 should reference `secret-broker run --aliases <alias> -- <cmd>` because it is backend-agnostic, rather than `need`, which is keyring-only. CONTRACT §6 stays silent on which secret-broker subcommand CEO uses because it is an orchestration concern, not a contract concern. If v1 §6 includes an example, it should use `run`, not `need`.

Workaround applied (Advisor-corrected 2026-05-18): see `adapters/claude-code/IMPLEMENTATION.md` §2.
