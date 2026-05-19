# Token Cost Standalone — Phase B Implementation Briefing for CEO Codex

**Drafted**: 2026-05-19 by Advisor
**Target executor**: CEO codex session (Phase B only — Advisor handles Phase A repo setup + Phase C ratification)
**Authoritative sources**:
- Goal: `docs/proposals/goal-G-AT-TOKEN-COST-STANDALONE-1.md`
- Phase A prerequisite: `github.com/izhiwen/AiPlus-Token-Cost` exists at v0.1.0 with release pipeline working
- Auto-memory: `advisor_briefing_stop_rules.md`

---

## Worktree isolation (NON-OPTIONAL, per CONTRACT v1.1 App D)

- Worktree:  `~/Projects/AiPlus/aiplus-public.token-cost-bundle/`
- Branch:    `feat/token-cost-bundle`
- Setup:     `git -C ~/Projects/AiPlus/aiplus-public worktree add ../aiplus-public.token-cost-bundle -b feat/token-cost-bundle`

Shared-file ownership:
- This session OWNS: `aiplus-public/assets/aiplus-token-cost/` (new directory), `aiplus-public/.github/workflows/release.yml`, `aiplus-public/install.sh`, `aiplus-public/install.ps1`, `aiplus-public/README.md`, `aiplus-public/README.zh-CN.md`, related src changes in aiplus-cli for install logic
- This session must NOT modify: `crates/aiplus-token-cost/` workspace member (this is source-of-truth in AiPlus-Token-Cost repo; aiplus-public mirror updated via subtree pull in Phase C only — NOT Phase B), CONTRACT.md (FROZEN), adapter code, calibration fixture
- If you need to change crate behavior → STOP and ping Advisor; the change goes into AiPlus-Token-Cost repo first

## Pre-condition before Phase B starts

Phase A must be complete:
- `github.com/izhiwen/AiPlus-Token-Cost` exists
- v0.1.0 tag pushed
- Release pipeline produced 2 binaries (Apple Silicon Mac + Intel Windows)
- Standalone install verified

If Advisor hasn't confirmed this, STOP.

## Scope — bundle integration

3 main work areas:

### 1. `assets/aiplus-token-cost/` materialization

Add the new directory with files matching the existing 6 substrate module pattern (study `assets/aiplus-agent-memory/` as reference):

```
assets/aiplus-token-cost/
├── aiplus-module.json   # manifest matching schema used by other 6 modules
├── README.md            # mirror of standalone repo README (or focused subset)
├── README.zh-CN.md
├── LICENSE              # Apache-2.0
├── CHANGELOG.md         # mirror of standalone repo CHANGELOG
├── SECURITY.md
├── MODULES.md           # bundled-module fingerprint
└── RELEASE_CHECKLIST.md
```

`aiplus-module.json` example (CEO designs final form; below is starting template):

```json
{
  "schemaVersion": "0.1.0",
  "name": "token-cost",
  "displayName": "AiPlus Token Cost",
  "version": "0.1.0",
  "source": "bundled",
  "license": "Apache-2.0",
  "requiredFiles": ["README.md", "CHANGELOG.md", "LICENSE"],
  "managedFiles": [
    ".aiplus/modules/aiplus-token-cost"
  ],
  "binaryAssets": [
    "aiplus-token-cost"
  ],
  "doctorChecks": [
    {
      "id": "token-cost-binary",
      "shell": "command -v aiplus-token-cost",
      "description": "Standalone aiplus-token-cost binary is on PATH."
    }
  ]
}
```

Key new field: `binaryAssets` — signals this module ships a binary, unlike the data/template substrate modules. CEO can add other manifest fields as needed.

### 2. Dual-binary release pipeline

`aiplus-public/.github/workflows/release.yml` currently builds `aiplus` binary only. Extend to also include `aiplus-token-cost` in the same tar.gz / zip per target:

**Approach A** (preferred): aiplus-public release CI downloads the latest AiPlus-Token-Cost release binary, repackages into aiplus tar.gz. Cross-repo dependency at release time.

**Approach B**: aiplus-public's workspace already includes `crates/aiplus-token-cost/` as a path member. The CI also builds this crate's binary natively (if AiPlus-Token-Cost repo has a `[[bin]]` section, aiplus-public's workspace inherits it). Same-build, same-target. Simpler if available.

Recommendation: try Approach B first. If workspace `[[bin]]` doesn't fly (e.g., source-of-truth in standalone repo means the bin entry shouldn't be in aiplus-public's crate), fall back to Approach A. Phase 1 design captures the decision.

Tar.gz structure after change:
```
aiplus-aarch64-apple-darwin.tar.gz
├── aiplus              (existing main binary)
└── aiplus-token-cost   (new bundled binary)
```

### 3. `aiplus install` deploys both binaries

`install.sh` currently extracts `aiplus` to `~/.local/bin/aiplus`. After change:
- Extract both `aiplus` and `aiplus-token-cost`
- Deploy both to `~/.local/bin/`
- `aiplus doctor` adds INFO line confirming `aiplus-token-cost` on PATH

`install.ps1` analogous on Windows.

`aiplus install <runtime>` (the subcommand for project-local module setup) does its existing thing plus materializes `assets/aiplus-token-cost/` into `.aiplus/modules/aiplus-token-cost/`.

### 4. README updates ("6 → 7 bundled modules")

In aiplus-public/README.md (English):

- "## What you get" section: keep token-cost bullet under Agent Team subsection (current), AND add a note that token-cost is also installable standalone
- "## The six bundled standalone modules" → "## The seven bundled standalone modules"; add 7th entry:

> - [AiPlus-Token-Cost](https://github.com/izhiwen/AiPlus-Token-Cost) — **Token consumption + USD cost rollups for dispatch logs**. 1h/8h/24h windows + top-5 most-expensive tasks. Installable standalone OR auto-installed with `aiplus`.

- README.zh-CN.md mirror updates

### 5. CHANGELOG entry

aiplus-public CHANGELOG `## Unreleased` section adds:

```
- Added `aiplus-token-cost` as a 7th bundled module: source-of-truth at
  github.com/izhiwen/AiPlus-Token-Cost; auto-installed alongside `aiplus`
  via `aiplus install`; also installable standalone via the new repo's
  release pipeline. `aiplus agent token-cost` subcommand continues to
  work unchanged.
```

## Phase structure

### Phase 1 — Write `docs/proposals/token-cost-bundle-impl-notes.md` BEFORE coding

Required sections:
1. Approach A vs B decision for dual-binary release (Phase B section 2)
2. `aiplus-module.json` exact schema for token-cost (in particular: how `binaryAssets` is consumed)
3. install.sh / install.ps1 changes (dual-extract logic)
4. README update plan (both languages)
5. Test plan: standalone install + bundled install + `aiplus agent token-cost` regression

### Phase 2 — Implement

Order:
1. `assets/aiplus-token-cost/` materialization (lowest risk, mostly file copy)
2. README updates
3. release.yml dual-binary
4. install.sh / install.ps1 dual-extract
5. `aiplus doctor` INFO line

### Phase 3 — Evidence

- Tag a test release locally / via draft, verify CI produces dual-binary tar.gz
- Live test: install fresh in a temp dir, both binaries land
- `aiplus agent token-cost` still works
- `aiplus-token-cost` standalone binary also works
- Trailer in impl-notes

## STOP rule (day-0 narrow per `advisor_briefing_stop_rules`)

- **STOP only on `_REGRESSION_`** (existing tests fail) or `_SCOPE-BREACH_` (touched `crates/aiplus-token-cost/` workspace member or CONTRACT.md or adapter code)
- **Do NOT STOP on `_IMPL-BUG_`** — fix and continue
- **Retry-once gate** for any test FAIL

## Conformance / verification

- `cargo test --workspace` PASS
- `cargo clippy --workspace --all-targets -- -D warnings` PASS
- Standalone install works on a clean Apple Silicon Mac
- Bundled install (via `aiplus install`) deploys both binaries
- `aiplus agent token-cost` regression PASS

## Scope fence

- **IN**: `assets/aiplus-token-cost/` materialization, release.yml dual-binary, install.sh + install.ps1 dual-extract, README × 2 updates, CHANGELOG draft, doctor INFO line
- **OUT**: `crates/aiplus-token-cost/` workspace member changes (source-of-truth lives in AiPlus-Token-Cost repo now; sync via subtree in Phase C), CONTRACT.md, adapter code, scoring, calibration, `aiplus-cli` Cargo.toml version field

## Deliverables (6)

1. `docs/proposals/token-cost-bundle-impl-notes.md`
2. `assets/aiplus-token-cost/` full directory + manifest
3. release.yml dual-binary modifications
4. install.sh + install.ps1 dual-extract
5. README + README.zh-CN updates
6. CHANGELOG draft text in impl-notes

## Time ceiling

**Hard ceiling: T+3d**.

## Handoff endpoint

Report `IMPL_VERDICT={PASS | PASS_WITH_DEVIATIONS | BLOCKED}` + which sub-features deviated + dual-install demo evidence. Advisor handles Phase C (subtree mechanism setup + retrospective).

## Owner gates

Standard. The release pipeline change might affect future tagging behavior — Owner gates each future release as today.

## Dependencies

- **Phase A complete** (AiPlus-Token-Cost repo exists + v0.1.0 release working)
- aiplus-public main at post-platform-narrowing baseline
- BWS-backed key wrapping for any LLM-call dev test (probably not needed for this goal)

---

— Advisor, 2026-05-19, Phase B briefing for dual-mode aiplus-token-cost.
