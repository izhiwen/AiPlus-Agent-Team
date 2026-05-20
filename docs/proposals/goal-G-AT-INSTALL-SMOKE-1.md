# Goal G-AT-INSTALL-SMOKE-1 — install.sh post-install smoke test

**Status**: PROPOSED — Owner-authorized 2026-05-19 after macOS 26.4 SIGKILL incident where install.sh reported PASS but `aiplus` was unrunnable.
**Drafted by**: Advisor 2026-05-19.
**Goal ID**: `G-AT-INSTALL-SMOKE-1`
**Ship target**: v0.6.14 (bundled with FRESHEN-1)

---

## TL;DR

Today's macOS 26.4 SIGKILL incident: `install.sh` printed `INSTALL_STATUS=PASS`, then `aiplus --version` was killed by the OS. Install believed itself successful even though the binary was unrunnable. Owner only discovered it failed when they tried to use lobby.

Fix: after `cp $BIN $INSTALL_DIR/aiplus`, install.sh runs `$INSTALL_DIR/aiplus --version` as a smoke check. Non-zero exit → `INSTALL_STATUS=FAIL` + clear error message + non-zero install exit code. No more silently broken installs.

## Success criterion (single sentence)

> `install.sh` exits non-zero with a clear error message if the installed binary cannot execute `--version` successfully, instead of reporting PASS.

## Goals (D1-D3)

- **D1**: After cp + chmod + (today's) defensive codesign on Darwin, run `"$INSTALL_DIR/aiplus" --version` and capture exit code + output.
- **D2**: On non-zero exit, print `INSTALL_STATUS=FAIL` + `SMOKE_FAIL=exit=$? output=<output>` + remediation hint, then `exit 1`.
- **D3**: On zero exit, print `SMOKE_PASS=version=<version>` line before existing `INSTALL_STATUS=PASS`.

## Sub-deliverables

| # | Artifact | Status |
|---|---|---|
| 1 | This goal doc | DRAFT (this commit) |
| 2 | `docs/decisions/install-smoke-1-impl-briefing.md` | DRAFT (this commit) |
| 3 | `docs/proposals/install-smoke-1-impl-notes.md` | PENDING — CEO |
| 4 | `install.sh` smoke step after copy | PENDING |
| 5 | (Optional) similar step in `install.ps1` for Windows | PENDING — verify if Windows has equivalent risk |

## Out of scope

- Behavior change for the defensive codesign step (already done in v0.6.13 main)
- The macOS 26.4 SIGKILL workaround itself (already in main)
- CI / release.yml changes
- Any aiplus CLI code (this is install.sh only)
- Forbidden files: CONTRACT.md, crates/, assets/, Cargo.toml, CHANGELOG.md, release.yml

## Acceptance criteria

- ✓ D1-D3 met
- ✓ `sh -n install.sh` syntax PASS
- ✓ Live test sandbox 1: install.sh against good v0.6.13 release → smoke step PASS, install exit 0
- ✓ Live test sandbox 2: install.sh against a deliberately broken binary (e.g., `chmod 000` after copy OR replace with non-executable) → smoke step FAIL, install exit 1
- ✓ install.ps1 corollary (if scope includes Windows): same behavior on Windows

## Why this exists

Defense in depth. Today's bug would have been caught at install time (clear error to Owner: "binary doesn't run") instead of first-run time (cryptic `zsh: killed aiplus`). Future macOS / Linux / Windows platform changes will likely break aiplus binaries in subtle ways too. Smoke test catches them at install boundary.

---

— Advisor, 2026-05-19. Half-day goal. Hard cap T+12h.
