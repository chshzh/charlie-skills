---
name: chsh-sk-ncs-0-workflow
description: Optional audit skill, not the default entry point — for normal work, start directly in the phase skill you need (each one now checks its own neighbor and offers to route you). Use this for a full project status dashboard, or to run the drift audit (Check 0-2, Case A-D) before a release or when you suspect something drifted silently without anyone noticing.
---

# chsh-sk-ncs-0-workflow — NCS Project Lifecycle Orchestrator

Reference for the project lifecycle, plus an optional status dashboard and drift
audit. **Not the normal entry point** — each phase skill now checks its own neighbor
and routes you directly. Load this skill for a dashboard, or for the Audit Mode below.

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

## Normal Work — Start Directly in the Phase Skill

For regular work, you don't need this skill at all. Go straight to whichever phase
you're working in — see **Quick-Entry Rules** below. Each phase skill checks its own
immediate neighbor and offers to route you there first if something's behind:

- **chsh-sk-ncs-1-pm-prd** always hands off to Specs; its **Mode D** handles "code
  moved, PRD didn't."
- **chsh-sk-ncs-2-dev-spec** checks PRD-vs-specs on the way in (**Mode B**) and offers
  a PRD update on the way out if the spec work surfaced something user-visible.
- **chsh-sk-ncs-3-dev-coding** guards against touching `src/` before docs are current,
  and checks its own output against docs before Handoff — offering Mode B / Mode D if
  something doesn't match.

This skill covers the two things that per-skill check doesn't: an overall **status
dashboard**, and **Audit Mode** below — a fuller forensic scan for when you suspect
something drifted silently in a way no single session would have caught (a hand-edit
made outside the skills, a declined doc-sync prompt you're not sure you followed up
on, or returning to a project after time away). The per-skill checks only see the
*current* session's changes; Audit Mode looks at the full project history.

---

## Audit Mode (optional) — Full Project Scan

Run this for the dashboard below, or when you suspect drift the per-skill checks
wouldn't catch. Not required for normal forward-flow work. Provide the project path
if not already in it.

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
CODE_PRD_PIN=$(grep -oE 'CONFIG_APP_PRD_VERSION="[0-9-]+"' prj.conf 2>/dev/null | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}-[0-9]{2}-[0-9]{2}')
CODE_SPEC_PIN=$(grep -oE 'CONFIG_APP_SPECS_VERSION="[0-9-]+"' prj.conf 2>/dev/null | grep -oE '[0-9]{4}-[0-9]{2}-[0-9]{2}-[0-9]{2}-[0-9]{2}')

# Check 0 — commit-based, not timestamp-based. Finds src/ commits that landed after
# the last commit that actually changed each pin *line* (-G matches the pickaxe regex
# against the diff, so an unrelated prj.conf edit doesn't move the anchor forward and
# hide drift). This is the primary drift signal: Check 1/2 below compare doc/pin
# timestamps and are structurally blind to "nobody touched a pin or a doc at all."
PRD_PIN_BUMP_SHA=$(git log -1 --format='%H' -G'CONFIG_APP_PRD_VERSION' -- prj.conf 2>/dev/null)
SPEC_PIN_BUMP_SHA=$(git log -1 --format='%H' -G'CONFIG_APP_SPECS_VERSION' -- prj.conf 2>/dev/null)

UNDECLARED_VS_PRD_PIN=""
UNDECLARED_VS_SPEC_PIN=""
[ -n "$PRD_PIN_BUMP_SHA" ]  && UNDECLARED_VS_PRD_PIN=$(git log --oneline "${PRD_PIN_BUMP_SHA}..HEAD" -- src/ 2>/dev/null)
[ -n "$SPEC_PIN_BUMP_SHA" ] && UNDECLARED_VS_SPEC_PIN=$(git log --oneline "${SPEC_PIN_BUMP_SHA}..HEAD" -- src/ 2>/dev/null)

echo "PRD_VER:       $PRD_VER"
echo "SPEC_VER:      $SPEC_VER"
echo "CODE_PRD_PIN:  $CODE_PRD_PIN"
echo "CODE_SPEC_PIN: $CODE_SPEC_PIN"
[ -z "$PRD_PIN_BUMP_SHA" ] && [ -z "$SPEC_PIN_BUMP_SHA" ] && \
  echo "⚠ Cannot find either pin's bump commit (shallow clone / squashed history?) — Check 0 skipped, relying on Check 1/2 only"
[ -n "$UNDECLARED_VS_PRD_PIN" ]  && echo "⚠ Commits touch src/ after the last PRD pin bump:"   && echo "$UNDECLARED_VS_PRD_PIN"
[ -n "$UNDECLARED_VS_SPEC_PIN" ] && echo "⚠ Commits touch src/ after the last Specs pin bump:" && echo "$UNDECLARED_VS_SPEC_PIN"
```

### Version format

- **PRD version** = newest timestamp row in `docs/pm-prd/PRD.md` Changelog (`YYYY-MM-DD-HH-MM`)
- **Specs version** = newest timestamp row in `docs/dev-specs/0-overview.md` Changelog (`YYYY-MM-DD-HH-MM`)
- **Code version** = the two sync pins in `prj.conf` — `CONFIG_APP_PRD_VERSION` and
  `CONFIG_APP_SPECS_VERSION`. These are what the code *declares* it was written against,
  not when `src/` was last touched — a commit that doesn't bump a pin hasn't declared any new
  spec/PRD sync, even if it changed files under `src/`.
  - ⚠️ If `src/` has **uncommitted changes**, invoke **chsh-sk-ncs-git-commit** first, then
    re-read the pins — this also blinds Check 0 below, since it only sees committed history.
  - ⚠️ **Check 0** (below) is commit-based and runs before Check 1/2 — it catches the case
    where nobody touched a pin or a doc at all, which timestamp comparison alone cannot see.

### Incomplete projects (fewer than three artifacts)

The staleness table below and the A/B/C/D case labels apply **only when all three artifacts
(PRD, Specs, Code) exist**. Before consulting the table:

- If fewer than three artifacts exist, or both `prj.conf` pins are empty/missing, do **not**
  compute a newest-artifact verdict. Print `Newest artifact: n/a (incomplete project)` and
  route by the **Quick-Entry Rules** table from the artifacts that are present (e.g. PRD-only → Phase 2).
- If exactly two of the three exist, run the same newest-timestamp comparison on that 2-of-3
  subset and sync the older one forward; reserve the A/B/C/D labels for the all-three case.

### Staleness decision table (all phases exist)

Three checks, run in order — not one three-way "newest wins" comparison. Check 0 is
commit-content-based and runs first, because it catches a failure mode the other two are
structurally blind to: code changed but **nobody touched a pin or a doc at all** — a
far more common omission than Check 2's rarer active mistake (a pin fabricated ahead of
its real doc value). If Check 0 finds anything doc-relevant, remediate it first (**Case
D**, below), then **re-run this whole scan** on the now-synced state before evaluating
Check 1/2 — don't skip them; a real PRD-vs-specs split can exist independently of the
code drift Check 0 just found.

**Check 0 — Undeclared `src/` commits since the last pin bump** (from Audit Mode's
`UNDECLARED_VS_PRD_PIN` / `UNDECLARED_VS_SPEC_PIN`):

| Result                                                                                                                            | Meaning                                                                | Recommended flow                                              |
| --------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------- |
| Commits found, none match a doc-relevant pattern (pure `fix:`/`refactor:`/`chore:` — see `chsh-ag-git`'s Step 2.5 severity table) | Nothing to sync                                                        | Fall through to Check 1                                       |
| Commits found, at least one matches a doc-relevant pattern                                                                        | Code moved past what's documented and no pin was ever bumped to say so | **Case D: read commits → 2 → 1 → Verification, then re-scan** |
| No commits found (or anchor unresolvable)                                                                                         | Check 0 has nothing to report                                          | Fall through to Check 1                                       |

**Check 1 — PRD vs Specs** (compare `PRD_VER` and `SPEC_VER` directly; a tie falls through to Check 2):

| Result | What drifted | Required sync | Recommended flow |
|--------|-------------|---------------|-------------------|
| `PRD_VER` newer | Specs lag PRD | Read PRD Changelog entries newer than `SPEC_VER`; update specs, then code | **Case A: 1 → 2 → Coding → Verification** |
| `SPEC_VER` newer | PRD lags Specs | Read Specs Changelog entries newer than `PRD_VER`; propagate intent to PRD, then code | **Case B: 2 → 1 → Coding → Verification** |
| Equal | No drift on this axis | — | fall through to Check 2 |

**Check 2 — Code vs its declared pins** (compare `CODE_SPEC_PIN` to `SPEC_VER`, and `CODE_PRD_PIN` to `PRD_VER`):

| Result                                                        | Meaning                                                             | Recommended flow                                                           |
| ------------------------------------------------------------- | ------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Either pin **behind** its doc version                         | Code hasn't caught up yet                                           | Already resolved by Case A/B's Coding step above — no separate case needed |
| Either pin **ahead of** its doc version                       | A pin was bumped to a timestamp with no matching doc entry — rare; usually a hand-typed `date +...` instead of a copied Changelog value | **Case C (secondary): Coding → 2 → 1 → Verification** |
| Both pins equal their doc version, and Check 1 was also equal | Nothing drifted                                                     | Skip to V&V — **Verification**                                             |

Case C only fires when a pin is *ahead of* its document — a rarer, active mistake. Check 0
is the check that catches the far more common one: nobody touched a pin at all.

### Sync procedure by case

**Case D — Undeclared commits (flow: read commits → 2 → 1 → Verification, then re-scan)**

1. Run `git log --oneline ${PRD_PIN_BUMP_SHA}..HEAD -- src/` and/or
   `git log --oneline ${SPEC_PIN_BUMP_SHA}..HEAD -- src/` (whichever fired in Check 0)
   to list every undeclared commit.
2. Classify each commit against the same severity table `chsh-ag-git` uses at its
   Step 2.5 ("Step A — Classify the change severity") — read the pattern list from
   there rather than re-deriving it. Separate Phase 1+2 (user-visible) from Phase
   2-only (implementation-boundary) commits; ignore any with no pattern match.
3. Load **chsh-sk-ncs-2-dev-spec** for every Phase-2-relevant commit: update
   `0-overview.md` and affected `<module>.md` files against what was actually built.
4. Load **chsh-sk-ncs-1-pm-prd** for every Phase-1+2 commit: add the resulting
   user-visible behaviour to `PRD.md`.
5. Bump the pins in `prj.conf` to the new Changelog timestamps once docs catch up.
6. **Re-run Audit Mode's scan and this decision table** on the now-synced state — Check 0 should
   come up empty; proceed through whatever Check 1/2 verdict results. Treat that as a
   fresh evaluation, not an assumption — a PRD-vs-specs split may still be there.

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

**Case C (secondary) — pin ahead of its document (flow: Coding → 2 → 1 → Verification)**

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
  → Recommended flow: <D | A | B | C | Verification only | (see Quick-Entry Rules)> — <reason>
  [⚠ Case D — N undeclared commit(s) touch a doc-relevant pattern since the last pin bump]
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
This guard now lives directly in **chsh-sk-ncs-3-dev-coding**'s Step 0, since that's
normally where you actually enter — restated here for the dashboard/Audit Mode view.
Before writing any code, verify that all agreed design changes are captured in docs:
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
| All phases exist — undeclared `src/` commits since last pin bump | Audit Mode Case D → read commits→2→1→Verification, then re-scan |
| All phases exist — PRD is newest | Audit Mode Case A → flow 1→2→Coding→Verification |
| All phases exist — Specs are newest | Audit Mode Case B → flow 2→1→Coding→Verification |
| All phases exist — pin ahead of its doc (rare) | Audit Mode Case C (secondary) → flow Coding→2→1→Verification |
| Architecture change | Phase 1 + 2 |
| Before a demo or release | Phase 4 (V&V) |
| Ready to tag and publish a release | `chsh-sk-ncs-git-release` |
| Upgrading to a newer NCS SDK version | `chsh-sk-ncs-migrate` |

---

## Skill Reference

| Skill | Invoke when | Output |
|-------|-------------|--------|
| `chsh-sk-ncs-0-workflow` | Status dashboard, or a pre-release / suspected-drift audit | Status dashboard + Audit Mode findings |
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
