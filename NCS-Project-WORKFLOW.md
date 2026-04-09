# Skills Workflow — NCS Project Lifecycle

This document describes the end-to-end workflow connecting the three project skills.
It is a coordination reference; each skill has its own detailed instructions.

---

## The Four-Phase Cycle

```
┌────────────────────────────────────────────────────────────────┐
│  PHASE 1 — PRODUCT DEFINITION         skill: chsh-pm-prd       │
│                                                                 │
│  Product Manager guides the device requirements:               │
│  • What should the device do, for whom, and why?               │
│  • Which Wi-Fi modes, UI behaviours, connectivity features?    │
│  • Success metrics and release criteria                        │
│                                                                 │
│  Input:  stakeholder ideas, user feedback, bug reports         │
│  Output: docs/product/PRD-YYYY-MM-DD.md (versioned, approved)  │
└───────────────────────────┬────────────────────────────────────┘
                            │ PRD approved → triggers Phase 2
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 2 — TECHNICAL DESIGN           skill: chsh-dev-project   │
│                                                                 │
│  Developer Engineer translates PRD into technical specs:       │
│  • Architecture choice (SMF+Zbus or multi-threaded)            │
│  • Per-module specs: state machines, Kconfig, APIs, memory     │
│  • docs/engineering/config.yaml (project context)              │
│  • docs/engineering/specs/architecture.md                      │
│  • docs/engineering/specs/<module>.md (one per feature)        │
│                                                                 │
│  Input:  docs/product/PRD-YYYY-MM-DD.md                        │
│  Output: docs/engineering/specs/*.md (approved by PM)          │
└───────────────────────────┬────────────────────────────────────┘
                            │ Specs approved → triggers Phase 3
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 3 — IMPLEMENTATION             skill: chsh-dev-project   │
│                                                                 │
│  Developer Engineer implements code from specs:                │
│  • src/modules/<name>/ per module                              │
│  • prj.conf, CMakeLists.txt, Kconfig wired up                  │
│  • Build verified: west build -p -b <board>                    │
│  • Spec updated if implementation differs from design          │
│                                                                 │
│  Input:  docs/engineering/specs/*.md                           │
│  Output: code, passing build, UART log evidence                │
└───────────────────────────┬────────────────────────────────────┘
                            │ Implementation done → triggers Phase 4
                            ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 4 — QA REVIEW                  skill: chsh-pm-review     │
│                                                                 │
│  Reviewer validates implementation against both PRD and specs: │
│  • Does code meet PRD acceptance criteria?                     │
│  • Does code match the engineering specs?                      │
│  • Automated check: chsh-pm-review/check_project.sh            │
│  • Manual review: chsh-pm-review/CHECKLIST.md                  │
│                                                                 │
│  Input:  code + docs/product/PRD-*.md + docs/engineering/      │
│  Output: docs/qa/QA-YYYY-MM-DD.md (scored, with issues)        │
│                                                                 │
│  Issues loop back:                                             │
│  • P0 critical issue → fix immediately (Phase 3)              │
│  • New requirement → update PRD (Phase 1)                      │
│  • Design change → update spec (Phase 2)                       │
└────────────────────────────────────────────────────────────────┘
```

---

## Document Ownership

| Document | Location | Owned by | Answers |
|---|---|---|---|
| `PRD-YYYY-MM-DD.md` | `docs/product/` | Product Manager | What & why — features, behaviour, success metrics |
| `architecture.md` | `docs/engineering/specs/` | Developer Engineer | System design, module map, memory budget |
| `<module>.md` | `docs/engineering/specs/` | Developer Engineer | How — state machines, Kconfig, APIs |
| `config.yaml` | `docs/engineering/` | Developer Engineer | Project context fed to dev skill |
| `QA-YYYY-MM-DD.md` | `docs/qa/` | Reviewer / PM | Scores, issues, pass/fail verdict |

**Division of responsibility:**
- PRD describes behaviour in user terms — no Kconfig flags, no memory numbers.
- Engineering specs translate PRD into implementation detail — Kconfig, state machines, APIs.
- The developer needs both: PRD for the "what", specs for the "how".

---

## Phase Trigger Rules

| Situation | Start at |
|---|---|
| New product or major new feature | Phase 1 — update PRD first |
| Small code fix with no new user-visible behaviour | Phase 3 — fix code, update spec if needed |
| Architecture change | Phase 1 + 2 — update PRD section then spec |
| QA critical issue (P0) | Phase 3 — fix code |
| QA finding reveals missing requirement | Phase 1 — add to PRD, then Phase 2 for spec |
| Before a demo or release | Phase 4 — run QA review |
| After any merge to main | Phase 4 — refresh QA |

---

## Skill Quick Reference

| Skill | Invoke when | Output |
|---|---|---|
| `chsh-pm-prd` | Defining or updating product requirements | `docs/product/PRD-YYYY-MM-DD.md` |
| `chsh-dev-project` | Translating PRD to specs or implementing code | `docs/engineering/specs/`, `src/` |
| `chsh-pm-review` | Validating a build against PRD and specs | `docs/qa/QA-YYYY-MM-DD.md` |
| `chsh-dev-commit` | Preparing git commits | Clean, logical commit history |
| `chsh-dev-mem-opt` | Diagnosing memory usage | Heap / stack recommendations |
