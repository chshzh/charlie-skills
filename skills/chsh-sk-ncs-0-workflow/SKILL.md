---
name: chsh-sk-ncs-0-workflow
description: Use when you need a project status dashboard or when a drift is detected (code ahead of specs, specs ahead of PRD). Scans PRD/Specs/Code version timestamps, identifies which artifact is newest, and routes to the right skill. Not needed for normal forward-flow work — each phase skill handles its own handoff.
---

# chsh-sk-ncs-0-workflow — NCS Project Lifecycle Orchestrator

Single entry point for any NCS project work. Scans the project state, presents a
status dashboard, and guides you through each phase — invoking the right skill at each step.

---

## The Four-Phase Cycle

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  PHASE 1 — PRODUCT DEFINITION         skill: chsh-sk-ncs-1-pm-prd            │
│                                                                              │
│  Product Manager defines device requirements:                                │
│  • What should the device do, for whom, and why?                             │
│  • Which Wi-Fi modes, UI behaviours, connectivity features?                  │
│  • Success metrics and release criteria                                      │
│                                                                              │
│  Input:  stakeholder ideas, user feedback, bug reports                       │
│  Output: docs/pm-prd/PRD.md  (Changelog updated)                             │
└───────────────────────────┬──────────────────────────────────────────────────┘
                            │ PRD approved → triggers Phase 2
                            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  PHASE 2 — TECHNICAL DESIGN           skill: chsh-sk-ncs-2-dev-spec          │
│                                                                              │
│  Developer Engineer translates PRD into technical specs:                     │
│  • Architecture choice (SMF+Zbus or multi-threaded)                          │
│  • docs/dev-specs/0-overview.md                                              │
│  • docs/dev-specs/1-architecture.md                                          │
│  • docs/dev-specs/<name>-module.md (one per feature)                         │
│                                                                              │
│  Input:  docs/pm-prd/PRD.md                                                  │
│  Output: docs/dev-specs/*.md (PRD Version field set)                         │
└───────────────────────────┬──────────────────────────────────────────────────┘
                            │ Specs approved → triggers Phase 3
                            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  PHASE 3 — IMPLEMENTATION   coding · debug · memopt                          │
│                                                                              │
│  Coding: src/modules/, prj.conf, CMakeLists.txt, Kconfig                     │
│  Debug:  fix runtime failures, crashes, WiFi issues                          │
│  Memopt: RAM/Flash budget, stack overflow, memory usage                      │
│                                                                              │
│  Input:  docs/dev-specs/*.md                                                 │
│  Output: code, clean build, UART log evidence                                │
└───────────────────────────┬──────────────────────────────────────────────────┘
                            │ Implementation done → triggers Phase 4
                            ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  PHASE 4 — VERIFICATION & VALIDATION (V&V)                                   │
│  Verification skill: chsh-sk-ncs-4-test-verification                         │
│  Validation skill: chsh-sk-ncs-4-test-validation                             │
│                                                                              │
│  Verification (always, no HW): code review, build, docs                      │
│  Validation (HW): shell-first UART + ZView watermarks, Router                │
│  • Output: docs/qa-test/ (VERIFICATION + VALIDATION)                         │
│                                                                              │
│  P0 → Phase 3  |  Spec gap → Phase 2  |  New req → Phase 1                   │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Step 0 — Scan Project State

Run this before anything else. Provide the project path if not already in it.

```bash
# Existence check
ls docs/pm-prd/PRD.md 2>/dev/null             && echo "PRD: YES"        || echo "PRD: NO"
ls docs/dev-specs/0-overview.md 2>/dev/null   && echo "SPECS: YES"      || echo "SPECS: NO"
ls src/main.c 2>/dev/null                     && echo "CODE: YES"       || echo "CODE: NO"
ls docs/qa-test/VERIFICATION-*.md 2>/dev/null     && echo "VERIFICATION: YES" || echo "VERIFICATION: NO"
ls docs/qa-test/VALIDATION_REPORT.md 2>/dev/null  && echo "VALIDATION: YES"   || echo "VALIDATION: NO"

# Doc versions (only when all three exist)
PRD_VER=$(awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/pm-prd/PRD.md 2>/dev/null | sort | tail -1)
SPEC_VER=$(awk -F'|' '/## Changelog/{f=1} f && $2 ~ /^ *[0-9]{4}-[0-9]{2}-[0-9]{2}/{gsub(/ /,"",$2); print $2}' docs/dev-specs/0-overview.md 2>/dev/null | sort | tail -1)

# Code version = the declared sync pins in prj.conf (the same pins chsh-ag-git and
# chsh-sk-ncs-4-test-verification already read) — NOT a git-log timestamp of src/.
# A commit that touches src/ for an unrelated reason (typo fix, refactor) must not
# look like "code drifted ahead of specs"; only a bumped pin is a real declaration.
CODE_PRD_PIN=$(grep -oE 'CONFIG_ZEGO_APP_PRD_VERSION="[0-9-]+"' prj.conf 2>/dev/null | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}-[0-9]{2}-[0-9]{2}')
CODE_SPEC_PIN=$(grep -oE 'CONFIG_ZEGO_APP_SPECS_VERSION="[0-9-]+"' prj.conf 2>/dev/null | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}-[0-9]{2}-[0-9]{2}')

# Secondary integrity check only — does not drive routing. If src/ has commits
# newer than the last commit that touched prj.conf, the pins above may not
# reflect the actual code state (someone edited src/ without bumping a pin).
SRC_COMMIT=$(git log -1 --date=format:'%Y-%m-%d-%H-%M' --format='%cd' -- src/ 2>/dev/null)
PIN_COMMIT=$(git log -1 --date=format:'%Y-%m-%d-%H-%M' --format='%cd' -- prj.conf 2>/dev/null)

echo "PRD_VER:       $PRD_VER"
echo "SPEC_VER:      $SPEC_VER"
echo "CODE_PRD_PIN:  $CODE_PRD_PIN"
echo "CODE_SPEC_PIN: $CODE_SPEC_PIN"
[ -n "$SRC_COMMIT" ] && [ -n "$PIN_COMMIT" ] && [ "$SRC_COMMIT" \> "$PIN_COMMIT" ] && \
  echo "⚠ src/ has commits after the last prj.conf pin update — pins may be stale"
```

### Version format

- **PRD version** = newest timestamp row in `docs/pm-prd/PRD.md` Changelog (`YYYY-MM-DD-HH-MM`)
- **Specs version** = newest timestamp row in `docs/dev-specs/0-overview.md` Changelog (`YYYY-MM-DD-HH-MM`)
- **Code version** = the two sync pins in `prj.conf` — `CONFIG_ZEGO_APP_PRD_VERSION` and
  `CONFIG_ZEGO_APP_SPECS_VERSION`. These are what the code *declares* it was written against,
  not when `src/` was last touched — a commit that doesn't bump a pin hasn't declared any new
  spec/PRD sync, even if it changed files under `src/`.
  - ⚠️ If `src/` has **uncommitted changes**, invoke **chsh-sk-ncs-git-commit** first, then re-read the pins.
  - ⚠️ If the integrity check above flagged `src/` as touched after the last pin update, treat
    the pins as unverified until a human confirms whether that change should have bumped one.

### Incomplete projects (fewer than three artifacts)

The staleness table below and the A/B/C case labels apply **only when all three artifacts
(PRD, Specs, Code) exist**. Before consulting the table:

- If fewer than three artifacts exist, or both `prj.conf` pins are empty/missing, do **not**
  compute a newest-artifact verdict. Print `Newest artifact: n/a (incomplete project)` and
  route by the **Quick-Entry Rules** table from the artifacts that are present (e.g. PRD-only → Phase 2).
- If exactly two of the three exist, run the same newest-timestamp comparison on that 2-of-3
  subset and sync the older one forward; reserve the A/B/C labels for the all-three case.

### Staleness decision table (all phases exist)

This is **two independent checks**, not one three-way "newest wins" comparison — the code
side is a *declared pin*, not a timestamp on the same axis as `PRD_VER`/`SPEC_VER`.

**Check 1 — PRD vs Specs** (compare `PRD_VER` and `SPEC_VER` directly; a tie falls through to Check 2):

| Result | What drifted | Required sync | Recommended flow |
|--------|-------------|---------------|-------------------|
| `PRD_VER` newer | Specs lag PRD | Read PRD Changelog entries newer than `SPEC_VER`; update specs, then code | **Case A: 1 → 2 → Coding → Verification** |
| `SPEC_VER` newer | PRD lags Specs | Read Specs Changelog entries newer than `PRD_VER`; propagate intent to PRD, then code | **Case B: 2 → 1 → Coding → Verification** |
| Equal | No drift on this axis | — | fall through to Check 2 |

**Check 2 — Code vs its declared pins** (compare `CODE_SPEC_PIN` to `SPEC_VER`, and `CODE_PRD_PIN` to `PRD_VER`):

| Result | Meaning | Recommended flow |
|--------|---------|-------------------|
| Either pin **behind** its doc version | Code hasn't caught up yet | Already resolved by Case A/B's Coding step above — no separate case needed |
| Either pin **ahead of** its doc version | Commits declared a spec/PRD version that was never actually written | **Case C: Coding → 2 → 1 → Verification** |
| Both pins equal their doc version, and Check 1 was also equal | Nothing drifted | Skip to V&V — **Verification** |

Case C only fires when a pin is *ahead of* its document — Check 1 alone can't detect that,
since PRD/Specs would still look mutually in sync while code has quietly gotten ahead of both.

### Sync procedure by case

**Case A — PRD is newest (flow: 1 → 2 → Coding → Verification)**

1. Read PRD Changelog rows newer than `SPEC_VER`.
2. Load **chsh-sk-ncs-2-dev-spec**: update `0-overview.md` and affected `<module>.md` files.
   Set `PRD Version` to `PRD_VER`.
3. Load **chsh-sk-ncs-3-dev-coding**: implement spec changes in `src/`.
4. Proceed to Verification.

**Case B — Specs are newest (flow: 2 → 1 → Coding → Verification)**

1. Read Specs Changelog rows newer than `PRD_VER` *and* rows newer than `CODE_SPEC_PIN`.
2. Load **chsh-sk-ncs-1-pm-prd**: surface any new requirements or behaviour changes back into
   `PRD.md` (add Changelog entry, bump PRD version).
3. Load **chsh-sk-ncs-3-dev-coding**: implement the spec changes in `src/`.
4. Proceed to Verification.

**Case C — Code is newest (flow: Coding → 2 → 1 → Verification)**

1. Run: `git log --oneline src/` — read commits from `HEAD` back to the commit that bumped
   the ahead pin (`CODE_SPEC_PIN` and/or `CODE_PRD_PIN`), or to `SPEC_VER` if unclear.
2. Infer the engineering intent from those commits.
3. Load **chsh-sk-ncs-2-dev-spec**: document the implemented behaviour in specs; bump
   `0-overview.md` Changelog.
4. Load **chsh-sk-ncs-1-pm-prd**: if the commits introduced user-visible behaviour changes,
   update `PRD.md`; add Changelog entry.
5. Proceed to Verification.

---

Present the project status dashboard to the user:

```
═══════════════════════════════════════════════════════
  PROJECT STATUS: <project name>
═══════════════════════════════════════════════════════
  Phase 1 — PRD           [ YES / NO ]  version: YYYY-MM-DD-HH-MM
  Phase 2 — Specs         [ YES / NO ]  version: YYYY-MM-DD-HH-MM
  Phase 3 — Code          [ YES / NO ]  PRD pin: YYYY-MM-DD-HH-MM  Specs pin: YYYY-MM-DD-HH-MM
  Phase 4 — Verification  [ YES / NO ]
  Phase 4 — Validation    [ YES / NO ]
═══════════════════════════════════════════════════════
  Newest artifact: <PRD | Specs | Code | all equal | n/a (incomplete project)>
  → Recommended flow: <A | B | C | Verification only | (see Quick-Entry Rules)> — <reason>
  [⚠ src/ has commits after the last prj.conf pin update — pins may be stale]
═══════════════════════════════════════════════════════
```

Ask: *"Proceed with recommended flow, or choose a different starting point?"*

---

## Phase 1 — Product Definition  `skill: chsh-sk-ncs-1-pm-prd`

**Enter Phase 1 when:**
- No PRD exists yet (new project or undocumented existing project)
- A feature is being added or changed
- Code changed but PRD was not updated

**Run:** Load skill **chsh-sk-ncs-1-pm-prd** and follow its workflow.

**Output:** `docs/pm-prd/PRD.md` with new Changelog entry

---

## Phase 2 — Technical Design  `skill: chsh-sk-ncs-2-dev-spec`

**Enter Phase 2 when:**
- PRD exists and was just updated (or is newer than current specs)
- No engineering specs exist yet

**Run:** Load skill **chsh-sk-ncs-2-dev-spec** and follow its workflow.

**Outputs:**
- `docs/dev-specs/0-overview.md` — spec index, PRD-to-spec mapping
- `docs/dev-specs/1-architecture.md` — module map, Zbus channels, memory budget
- `docs/dev-specs/<name>-module.md` — one per feature module

Each spec's `PRD Version` field must match the latest PRD Changelog timestamp.

---

## Phase 3 — Implementation

| Sub-phase | Skill | Enter when |
|-----------|-------|------------|
| **Coding** | `chsh-sk-ncs-3-dev-coding` | Specs approved, code needs writing or updating |
| **Debug** | `chsh-sk-ncs-3-dev-debug` | Runtime failure, crash, WiFi issue, boot hang |
| **Memopt** | `chsh-sk-ncs-3-dev-memopt` | RAM/Flash overflow, stack corruption |

**⛔ Phase 3 guard — check before touching `src/`:**
Before writing any code, verify that all agreed design changes are captured in docs.
If the conversation contains discussed-but-undocumented changes:
- Requirements changed → route to **Phase 1** first (update PRD)
- Only spec/implementation details changed → route to **Phase 2** first (update specs)
Never touch `src/` until PRD and specs are current and approved.
A user saying "start implementation" does **not** bypass this guard.

**Enter Phase 3 when:**
- Specs exist and are approved
- Specs were updated and code needs to catch up
- A bug fix or runtime issue is found during testing

**Default flow:** Start with **Coding** → if runtime issues arise → **Debug** → if
memory budget exceeded → **Memopt** → back to Coding. Iterate until clean build + UART
log evidence of correct behavior.

---

## Phase 4 — Verification & Validation (V&V)  `chsh-sk-ncs-4-test-verification` · `chsh-sk-ncs-4-test-validation`

**Normal trigger:** The `chsh-sk-ncs-git-commit` / `chsh-ag-git` Step 8 `AskQuestion` offers to run Verification automatically after every commit in repos with `docs/qa-test/`. In normal forward-flow you do not need to enter Phase 4 manually.

**Manual entry when:**
- Skipped at commit time and want to run it now
- Before a release or demo
- After any merge to main

**Run:** Load skill **chsh-sk-ncs-4-test-verification** and follow its workflow.

| Sub-phase | Document | Hardware? |
|-----------|----------|-----------|
| **Verification** (always) | `docs/qa-test/VERIFICATION-YYYY-MM-DD-HH-MM.md` | No |
| **Validation** (always) | `docs/qa-test/VALIDATION_PLAN.md` + `docs/qa-test/VALIDATION_REPORT.md` | Yes |

**Feedback routing after Phase 4:**

| Finding | Route back to |
|---------|---------------|
| Security finding or P0 code issue | Phase 3 — fix code |
| Spec gap / undocumented behaviour | Phase 2 — update spec, then Phase 3 |
| New requirement found | Phase 1 — add to PRD, then Phase 2, then Phase 3 |
| All P0 checks pass (Verification) + all P0 TCs pass (Validation) | ✅ Ready for release |

---

## Document Conventions

### Document Information header

Every `PRD.md` and engineering spec opens with a **Document Information** table whose
fields come **verbatim from the matching template** (`PRD_TEMPLATE.md`, `0-OVERVIEW_TEMPLATE.md`,
`1-ARCHITECTURE_TEMPLATE.md`, `MODULE_TEMPLATE.md`) and stay identical from creation through every
maintenance edit — no ad-hoc variants like `Latest Version`.

- **`Version`** = the document's **own** latest edit timestamp (the current time, = its
  newest Changelog row). Bump it on every edit.
- **`PRD Version`** (specs only) = the PRD Changelog timestamp the spec is synced to.
- `Version` and `PRD Version` are different timestamps and must **not** be set equal —
  if they match, `Version` was filled in wrong.

### Living documents — Changelog table

`PRD.md` and all engineering specs are **living documents** with a single canonical filename.
Changes are tracked in a built-in Changelog table:

```markdown
## Changelog

| Version          | Summary of changes          |
|------------------|-----------------------------|
| 2026-04-09-10-00 | Initial draft               |
| 2026-04-15-14-30 | Added P2P mode requirements |
```

- Version is a timestamp `YYYY-MM-DD-HH-MM` — time included so same-day edits are distinguishable.
  Generate with: `date +%Y-%m-%d-%H-%M`
- Never delete rows — append-only.
- Git provides the full diff; the Changelog is the human-readable log.

### Audit snapshots — dated filenames

Verification reports are **point-in-time snapshots**. Each run creates a new dated file:

```
docs/qa-test/VERIFICATION-2026-04-09-14-30.md
```

Validation uses two **living** documents (fixed names, internal Changelog), not dated snapshots:
`docs/qa-test/VALIDATION_PLAN.md` and `docs/qa-test/VALIDATION_REPORT.md`.

---

## Document Ownership

| Document | Location | Owned by | Answers |
|----------|----------|----------|---------|
| `PRD.md` | `docs/pm-prd/` | Product Manager | What & why — features, behaviour, success metrics |
| `0-overview.md` | `docs/dev-specs/` | Developer | Spec index, PRD-to-spec map, design decisions |
| `1-architecture.md` | `docs/dev-specs/` | Developer | System design, module map, memory budget |
| `<module>.md` | `docs/dev-specs/` | Developer | State machines, Kconfig, APIs |
| `VALIDATION_PLAN.md` / `VALIDATION_REPORT.md` | `docs/qa-test/` | Tester / PM | Test plan + PRD acceptance pass/fail, UART evidence, ZView memory watermarks |

**Division of responsibility:**
- PRD: behaviour in user terms — no Kconfig flags, no memory numbers.
- Specs: translate PRD into implementation detail — Kconfig, state machines, APIs.
- Developer needs both: PRD for "what", specs for "how".

---

## Quick-Entry Rules

| Situation | Start at |
|-----------|---------|
| Brand new project | Phase 1 |
| Existing code, no docs | Phase 1 (Mode A with code scan) |
| PRD done, no specs | Phase 2 |
| Specs done, no code | Phase 3 |
| Code done, need validation | Phase 4 |
| Small bug fix only | Phase 3 — Coding (or Debug if a runtime crash) |
| Feature request | Phase 1 (Mode B) |
| Post-merge validation | Phase 4 |
| All phases exist — PRD is newest | Step 0 Case A → flow 1→2→Coding→Verification |
| All phases exist — Specs are newest | Step 0 Case B → flow 2→1→Coding→Verification |
| All phases exist — Code is newest | Step 0 Case C → flow Coding→2→1→Verification |
| Architecture change | Phase 1 + 2 |
| Before a demo or release | Phase 4 (V&V) |
| Ready to tag and publish a release | `chsh-sk-ncs-git-release` |
| Upgrading to a newer NCS SDK version | `chsh-sk-ncs-migrate` |

---

## Skill Reference

| Skill | Invoke when | Output |
|-------|-------------|--------|
| `chsh-sk-ncs-0-workflow` | **Starting any project work** | Status dashboard + phase guidance |
| `chsh-sk-ncs-1-pm-prd` | Defining or updating product requirements | `docs/pm-prd/PRD.md` |
| `chsh-sk-ncs-2-dev-spec` | Translating PRD to engineering specs | `docs/dev-specs/*.md` |
| `chsh-sk-ncs-3-dev-coding` | Implementing code from specs | `src/`, `prj.conf`, passing build |
| `chsh-sk-ncs-3-dev-debug` | Debugging firmware failures, UART log analysis | Root cause identified and fixed |
| `chsh-sk-ncs-4-test-verification` | Verification (code review, build, docs audit — no hardware) | `docs/qa-test/VERIFICATION-*.md` |
| `chsh-sk-ncs-4-test-validation` | Hardware validation (shell-first + ZView watermarks against PRD acceptance criteria) | `docs/qa-test/VALIDATION_PLAN.md` + `VALIDATION_REPORT.md` |
| `chsh-sk-ncs-git-commit` | Preparing git commits | Clean, logical commit history |
| `chsh-sk-ncs-git-release` | Tagging a release, watching CI, publishing firmware to GitHub | GitHub release with published artifact |
| `chsh-sk-ncs-migrate` | Upgrading the project to a newer NCS version (single hop or multi-hop) | Migrated app, clean build, verified on hardware |
| `chsh-sk-ncs-3-dev-memopt` | Diagnosing memory usage | Heap / stack recommendations |

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check whether any facts in this skill are new, corrected, or outdated (e.g. new phase transitions, skill routing changes, project lifecycle patterns).

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Present a summary to the user and ask for approval using `AskQuestion`.
3. Apply approved updates to this file immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
