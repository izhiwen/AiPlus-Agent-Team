# G-AT-INSTALL-SMOKE-1 — CEO Impl Briefing

**Drafted**: 2026-05-19 by Advisor
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-INSTALL-SMOKE-1.md`
- Auto-memory: `advisor_briefing_stop_rules.md`, `parallel_ceo_tag_gate.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.install-smoke/`
- Branch:    `feat/install-smoke-1`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.install-smoke -b feat/install-smoke-1`

## Scope

Tiny scope. install.sh ONLY (no aiplus-cli code, no release.yml).

### Phase 1 — Investigation (~20 min)

1. Read install.sh top-to-bottom; understand current `INSTALL_STATUS=PASS` emit point (~line 228 in main).
2. Identify the right place to insert smoke step: AFTER `chmod 755 $INSTALL_DIR/aiplus`, AFTER the defensive codesign (Darwin block, ~line 226), and BEFORE `INSTALL_STATUS=PASS` emit.
3. Decide error message format. Recommended:
   ```
   SMOKE_FAIL=exit=137 output=killed
   INSTALL_STATUS=FAIL
   ERROR: installed binary at $INSTALL_DIR/aiplus failed --version smoke test.
   ERROR: this may indicate (a) macOS Gatekeeper rejection, (b) corrupted download,
   ERROR: (c) missing system library, or (d) platform mismatch.
   ERROR: see https://github.com/izhiwen/AiPlus/issues for help.
   exit 1
   ```

### Phase 2 — Implementation (~30 min)

Insert smoke block after defensive codesign on Darwin (per main `1e3e5ea` location).

```sh
# Post-install smoke test: actually run the binary. If it can't execute
# (e.g., macOS Gatekeeper SIGKILL, corrupted download, missing library),
# fail the install loudly instead of reporting PASS and leaving the
# user with a binary that crashes on first real use.
SMOKE_OUTPUT="$("$INSTALL_DIR/aiplus" --version 2>&1)"
SMOKE_EXIT=$?
if [ $SMOKE_EXIT -ne 0 ]; then
  echo "SMOKE_FAIL=exit=$SMOKE_EXIT output=$SMOKE_OUTPUT" >&2
  echo "INSTALL_STATUS=FAIL" >&2
  echo "ERROR: installed binary at $INSTALL_DIR/aiplus failed --version smoke test." >&2
  echo "ERROR: this may indicate (a) macOS Gatekeeper rejection, (b) corrupted download," >&2
  echo "ERROR: (c) missing system library, or (d) platform mismatch." >&2
  echo "ERROR: see https://github.com/izhiwen/AiPlus/issues for help." >&2
  exit 1
fi
echo "SMOKE_PASS=version=$SMOKE_OUTPUT"
```

Then leave existing `INSTALL_STATUS=PASS` + `installed=...` lines unchanged.

If scope includes install.ps1 (Owner-confirm in Phase 1): add equivalent PowerShell smoke step. Recommend doing this — small additional cost, equivalent value on Intel Windows.

### Phase 3 — Evidence (~30 min)

3 sandbox tests:

1. **Good release**: `AIPLUS_VERSION=v0.6.13 AIPLUS_INSTALL_DIR=$(mktemp -d) sh ./install.sh` → smoke PASS, install exit 0, `SMOKE_PASS=version=0.6.13` line present.
2. **Broken binary** (chmod 000 after copy, simulate via env var or post-copy hook): smoke FAIL, install exit 1, clear error message.
3. **`sh -n install.sh`**: syntax PASS.

## STOP rules (per auto-memory)

- `_REGRESSION_` → STOP
- `_SCOPE-BREACH_` → STOP (touched forbidden file)
- `_TAG-BREACH_` → STOP (NEVER tag, NEVER bump Cargo.toml, NEVER push tags)
- Retry-once gate standard

**Forbidden files**: all crates/, all assets/, Cargo.toml, CHANGELOG.md, release.yml, CONTRACT.md.

## Scope fence

- **IN**: install.sh smoke block, (optional) install.ps1 smoke block, impl-notes
- **OUT**: all other code paths
- **GRAY**: cosmetic tweaks to existing install.sh comments adjacent to smoke block

## Deliverables (3-4)

1. `docs/proposals/install-smoke-1-impl-notes.md` — Phase 1 + Phase 3
2. `install.sh` smoke block (~12 lines)
3. (Optional, Owner-decided) `install.ps1` smoke block
4. Sandbox tests pass

## Time ceiling

Hard cap T+12h. Realistic estimate ~2h (Phase 1 + 2 + 3).

## Handoff

Report IMPL_VERDICT. NO TAG. NO VERSION BUMP. NO PUSH TO ORIGIN MAIN. Branch push for PR review OK. Advisor Phase C handles ratify + tag.

---

— Advisor, 2026-05-19. Smallest of the 3 parallel goals.
