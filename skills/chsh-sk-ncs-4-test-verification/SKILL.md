---
name: chsh-sk-ncs-4-test-verification
description: >-
  Load when running Verification for an NCS project — code review,
  clean build, and documentation consistency audit. No hardware required.
  Code review is delegated to chsh-ag-code-reviewer for an independent pass.
---

# chsh-sk-ncs-4-test-verification — Verification (no hardware)

Verification phase of the NCS project lifecycle. Runs after implementation is complete,
before any release or demo, and after any merge to main. No hardware required —
can run in CI.

```
Verification
├── Code review (structure, config, standards) — delegated to chsh-ag-code-reviewer
├── Code format check (clang-format --dry-run)
├── Build verification (README "### Build" commands only)
├── Host test run (native_sim / ztest, no board — skipped if tests/ doesn't exist)
├── PRD satisfaction check (FR-by-FR code reading)
└── Documentation consistency audit
```

> **Knowledge sources**: Call `mcp__claude_ai_Nordic_MCP__nordicsemi_workflow_ncs` at the start of each session — loads `nrfutil-manual` and `embedded-code-guidance-ncs-zephyr`. Use `mcp__claude_ai_Nordic_MCP__nordicsemi_search_sources` before checking any Kconfig symbol or board capability.

**Output**: `docs/qa-test/VERIFICATION-YYYY-MM-DD-HH-MM.md`

---

## Step 0 — Check Inputs

```bash
# Extract version chain — record all three before proceeding
# PRD_VERSION: newest timestamp row in the PRD Changelog table
awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/pm-prd/PRD.md | sort | tail -1

# SPECS_VERSION: newest timestamp row in 0-overview.md Changelog table
awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/dev-specs/0-overview.md | sort | tail -1

# Code version: prj.conf carries both version tags
grep -E "APP_(PRD|SPECS)_VERSION" prj.conf

# Remaining inputs
ls docs/dev-specs/                                         # spec files present
git status --short                                         # check for uncommitted changes
grep -n "^### Build" README.md                            # locate canonical build commands
```

Record `PRD_VERSION` (latest PRD Changelog entry), `SPECS_VERSION` (latest 0-overview.md Changelog entry), and the version tags in `prj.conf` — all three go into every report header.

> **Note on uncommitted changes**: if `git status` shows modified source files, the code version is ahead of the last commit. Build and test anyway — but note in the report that the build reflects uncommitted state. The version chain must satisfy:

```
PRD_VERSION  →  specs/0-overview.md written against PRD_VERSION
                  ↓
             SPECS_VERSION  →  src/main.c SPECS_VERSION tag matches
                                  ↓
                             README features reflect current PRD features
```

---

## Verification

No hardware required. Can be run independently at any point.

### 4.1.1 Code Review (delegated)

Delegate to **chsh-ag-code-reviewer** for an independent pass — deliberately without
sharing this conversation's rationale, so the review isn't the same mind checking its
own work twice.

```
Use chsh-ag-code-reviewer to review <target paths — e.g. src/modules/, or the modules
touched since the last Verification pass>.

Inputs: docs/dev-specs/1-architecture.md + affected <module>.md, docs/pm-prd/PRD.md.
```

Do **not** pass along why any implementation choice was made — only paths and doc
locations. The subagent checks structure, config, coding standards, and security
against the same P0/P1/P2 scale used everywhere else in this lifecycle, calls
`ReportFindings`, and returns a findings table. Fold that table into
**`## 1. Code Review`** of the report below.

Any P0 (security is always P0) → stop, route to Phase 3 immediately.

### 4.1.2 Code Format Check

Run clang-format in check mode over the project sources using the NCS toolchain binary (never system clang-format). This feeds the **`## 2. Code Format`** report section.

```bash
# Dry-run only — report violations, do not rewrite files here.
# Use the NCS toolchain launcher (see chsh-sk-ncs-clang-format for the exact binary).
find src -name '*.c' -o -name '*.h' | xargs clang-format --dry-run --Werror
```

Record each file with violations. Clean = ✅; any violation = ⚠️ (P2). For auto-fixing, route to `chsh-sk-ncs-clang-format`.

### 4.1.3 Build Verification

**Build command policy (strict):**
- Use only the commands listed in the application's `README.md` under `### Build`.
- Do not invent or simplify build commands (no ad-hoc `west build -p`, no custom flags)
  unless the README command itself requires parameter substitution.
- If README has multiple board targets, run each command exactly as documented.
- If `### Build` is missing, fail verification as documentation gap (P1) and stop build checks.

```bash
# Run exactly the build commands from README "### Build" section.
# Example policy:
#   1) copy each documented command verbatim
#   2) execute via the configured NCS toolchain launcher
#   3) record pass/fail, warnings, and binary size for each target
```

Zero **compiler** warnings required. Record binary size. Any compiler warning = P1.

> **Known non-P1 Kconfig notices**: `warning: Experimental symbol WIFI_NM_WPA_SUPPLICANT is enabled` and similar NCS Wi-Fi experimental warnings appear on every nRF70 build and are expected. Do not flag these as P1 — they are upstream NCS status notices, not project issues. Only flag new or project-specific Kconfig warnings.

### 4.1.4 Host Test Run (native_sim / ztest)

No hardware required — runs entirely on the host. Executes whatever `tests/modules/<name>/`
ztest suites exist for pure-logic modules (written during **chsh-sk-ncs-3-dev-coding**'s
host-test step — see that skill for what qualifies as a candidate module).

```bash
# Guard — skip this step entirely if no host tests exist yet. Absence is not a failure;
# host testing is opt-in per module, not mandatory project-wide.
[ -d tests/modules ] || echo "No host tests present — skip this step"

# Run every suite under tests/, native_sim only
west twister -p native_sim -T tests/ --inline-logs
```

| Result | Verdict | Route |
|--------|---------|-------|
| No `tests/modules/` directory | ➖ N/A | Not a gap on its own — note in the report which modules could be host-test candidates but aren't covered yet |
| All suites pass | ✅ Pass | Proceed |
| Any suite fails | ❌ P0 | Phase 3 immediately — a failing host test is a proven functional bug, not a style nit |
| Suite fails to build | ⚠️ P1 | Check `CMakeLists.txt`'s relative path into `src/modules/<name>/` — the module likely moved since the test was written |

Record pass/fail counts and the failing test names (if any). This feeds the
**`## 4. Host Tests`** report section.

> **Gotcha**: `west twister` exits nonzero if *any* suite in the run fails — the exit
> code alone doesn't say which one. Read the summary table it prints, not just the
> return code.

### 4.1.5 PRD Satisfaction Check (Code Reading)

Read the code FR-by-FR against the PRD acceptance criteria. No hardware — confirm by inspecting source, Kconfig, and module wiring. This feeds the **`## 5. PRD Satisfaction Check`** report section and the routing verdicts below.

For each Functional Requirement in `docs/pm-prd/PRD.md`, find the code evidence and assign a verdict:

| Verdict | Meaning | Route |
|---------|---------|-------|
| ✅ Implemented | Acceptance criterion satisfied in code | — |
| ⚠️ Partial | Some criteria met, gaps remain | Phase 3 (next iteration) |
| ❓ Not visible | Cannot confirm without running on hardware | Carry as a priority TC to `chsh-sk-ncs-4-test-validation` |
| ❌ Mismatch | Code contradicts the criterion | P0 — Phase 3 immediately |

> A ❓ verdict is not a failure — it is the hand-off list to hardware validation. A ❌ is a P0 code bug.

### 4.1.6 Documentation Consistency Audit

Work through the version chain in order. A failure at an earlier step blocks the steps below it.

**Step A — PRD version → Specs**

```bash
# 1. Read the newest PRD Changelog entry to get PRD_VERSION
awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/pm-prd/PRD.md | sort | tail -1

# 2. Confirm 0-overview.md PRD Version field matches PRD_VERSION
grep "PRD Version" docs/dev-specs/0-overview.md
```

| Check | Pass condition |
|-------|----------------|
| Specs reference current PRD version | `docs/dev-specs/0-overview.md` header/changelog explicitly references `PRD_VERSION`; no older PRD version cited |
| PRD FR/NFR coverage | Every Functional Requirement and Non-Functional Requirement in PRD traceable to at least one spec requirement |

**Step B — Specs version → Code** *(only if Step A passes)*

```bash
# 1. Read the newest Specs Changelog entry to get SPECS_VERSION
awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/dev-specs/0-overview.md | sort | tail -1

# 2. Confirm prj.conf carries the matching tags
grep -E "APP_(PRD|SPECS)_VERSION" prj.conf
```

| Check | Pass condition |
|-------|----------------|
| `CONFIG_APP_SPECS_VERSION` in `prj.conf` | Matches the latest `docs/dev-specs/0-overview.md` Changelog entry |
| `CONFIG_APP_PRD_VERSION` in `prj.conf` | Matches the latest `docs/pm-prd/PRD.md` Changelog entry |
| Spec modules → code | Every `docs/dev-specs/<name>-module.md` has a `src/modules/<name>/` counterpart |

**Step C — PRD features → README** *(only if Steps A & B pass)*

```bash
# Extract feature list from PRD (FR section) and compare against README
grep -E "^\*\*?FR[0-9]|^-.*feature" docs/pm-prd/PRD.md | head -20
grep -i "feature\|capability\|support" README.md | head -20
```

| Check | Pass condition |
|-------|----------------|
| README feature list | Every PRD feature (FR items) mentioned in README; no stale or removed features listed |
| No undocumented features in README | README does not describe features absent from the PRD |

### 4.1.7 Generate Verification Report

Create `docs/qa-test/VERIFICATION_REPORT-YYYY-MM-DD-HH-MM.md` using `VERIFICATION_REPORT_TEMPLATE.md` as the base:

| Section | Content |
|---------|---------|
| Document Info | PRD version, Specs version, reviewer, date |
| Code Review | Findings per check, severity (P0/P1/P2), from `chsh-ag-code-reviewer` (4.1.1) |
| Code Format | clang-format check result (4.1.2) — files with violations |
| Build Result | Pass/Fail, warning count, binary size (4.1.3) |
| Host Tests | Suite pass/fail counts, failing test names, uncovered candidate modules (4.1.4) |
| PRD Satisfaction | FR-by-FR verdicts from code reading (4.1.5): ✅/⚠️/❓/❌ |
| Docs Audit | Version chain: PRD→Specs (Step A), Specs→Code (Step B), PRD features→README (Step C); coverage gaps |
| Routing | P0 → Phase 3 / spec gap → Phase 2 / ❓ → Validation / ✅ proceed to Validation |

---

## Feedback Routing

| Finding | Priority | Route |
|---------|----------|-------|
| Security finding (P0) | P0 | Phase 3 — fix code immediately |
| P0 test case fails (code bug) | P0 | Phase 3 (`chsh-sk-ncs-3-dev-coding`) |
| Host test suite fails | P0 | Phase 3 — proven functional bug |
| Build failure or warning | P0/P1 | Phase 3 |
| PRD criterion mismatch (❌) | P0 | Phase 3 |
| Spec gap / undocumented behaviour | P1 | Phase 2 (`chsh-sk-ncs-2-dev-spec`) |
| New requirement found | P1 | Phase 1 (`chsh-sk-ncs-1-pm-prd`) → Phase 2 → Phase 3 |
| P1/P2 issues only | P2 | Phase 3 (next iteration) |
| All P0 checks pass | ✅ | Proceed to hardware validation (`chsh-sk-ncs-4-test-validation`) |

After reporting, ask:
> "Verification complete. Proceed to hardware validation with **chsh-sk-ncs-4-test-validation**, or route issues to the appropriate phase?"

---

## Gotchas

| Gotcha | Detail |
|--------|--------|
| clang-format version mismatch | Use NCS toolchain clang-format (`nrfutil sdk-manager toolchain launch -- clang-format`), not system clang-format |
| PRD "not visible" ≠ "not implemented" | Auto-reconnect via Zephyr WiFi manager has no visible app code — check `CONFIG_WIFI_CREDENTIALS` / `CONFIG_WIFI_NM` before flagging |
| Version grep matches table header | `grep -m1 '\| [0-9]'` captures the `\| Version \|` header row, not a real version. Use `grep -E '^\| [0-9]{4}'` to match only timestamp rows. |
| PRD Changelog entries out of order | Authors sometimes append entries non-chronologically. Always read **all** rows and pick the lexicographically latest timestamp, not just the last row. |
| Uncommitted changes affect code version | `git log` reflects the last commit, not on-disk state. Run `git status --short` — if modified files exist, note in the report that the build reflects uncommitted code. |

> Code-review-specific false positives (security grep noise, SoftAP default password) now live in `chsh-ag-code-reviewer`'s "Known false positives" section — that agent owns the code-review checks.

---

## Related Skills

| Task | Skill |
|------|-------|
| Implement code (Phase 3) | `chsh-sk-ncs-3-dev-coding` |
| Debug firmware failures | `chsh-sk-ncs-3-dev-debug` |
| Auto-fix clang-format violations | `chsh-sk-ncs-clang-format` |
| Hardware validation | `chsh-sk-ncs-4-test-validation` |
| Tag and publish release | `chsh-sk-ncs-git-release` |
| Full lifecycle orchestration | `chsh-sk-ncs-0-workflow` |

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check
whether any facts in this skill are new, corrected, or outdated (e.g. new
security risk patterns, Zephyr coding standards, or documentation checks).

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Present a summary to the user and ask for approval using `AskQuestion`.
3. Apply approved updates to this file immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
