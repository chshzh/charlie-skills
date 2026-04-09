# NCS Project & Product Management Workflow

This document defines the end-to-end workflow for managing NCS projects using the
ProductManager skill, Developer skill, and openspec together.

---

## The Five-Phase Cycle

```
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 1 — PRODUCT DEFINITION  (ProductManager skill)           │
│                                                                  │
│  @ProductManager → generates/updates pm/PRD.md                  │
│  • What to build, why, success metrics                          │
│  • Feature selection table (☑/☐)                               │
│  • High-level constraints (RAM budget, NCS version, board)      │
│  • Architecture CHOICE (not deep implementation detail)         │
│  • Roadmap (current / near-term / future)                       │
│                                                                  │
│  Input:  stakeholder requirements, bug reports, roadmap ideas   │
│  Output: pm/PRD.md (approved)                                   │
└────────────────────────┬────────────────────────────────────────┘
                         │ PRD.md approved → triggers Phase 2
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 2 — TECHNICAL DESIGN  (openspec)                         │
│                                                                  │
│  cd pm/                                                         │
│  openspec propose "feature or fix description"                  │
│    → pm/openspec/specs/<feature>.md                             │
│    • How to build it (data flows, framing protocol, Kconfig)    │
│    • Latency / memory budget impact                             │
│    • Alternatives considered                                    │
│    • Pass criteria for verification                             │
│                                                                  │
│  openspec tasks "feature description"                           │
│    → Ordered dev task list referencing the spec                 │
│                                                                  │
│  Input:  pm/PRD.md + pm/openspec/config.yaml (auto-read)        │
│  Output: pm/openspec/specs/<feature>.md + task list             │
└────────────────────────┬────────────────────────────────────────┘
                         │ Spec approved → triggers Phase 3
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 3 — DEVELOPMENT  (Developer NCS skill)                   │
│                                                                  │
│  Developer reads (in order):                                    │
│    1. pm/PRD.md              ← what & why, constraints          │
│    2. pm/openspec/specs/     ← how, Kconfig, data flows         │
│    3. openspec task list     ← ordered dev steps                │
│                                                                  │
│  Implements → builds both roles → flashes → checks UART logs    │
│                                                                  │
│  On finish: update spec if implementation differed from design  │
│                                                                  │
│  Input:  PRD.md + specs/*.md + task list                        │
│  Output: code changes, updated specs (if needed)                │
└────────────────────────┬────────────────────────────────────────┘
                         │ Dev complete → triggers Phase 4
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 4 — VERIFICATION  (manual + CI)                          │
│                                                                  │
│  • west build -p both gateway and headset roles                 │
│  • Check memory budget (headset RAM < 93%)                      │
│  • Flash both boards, observe UART logs                         │
│  • Confirm pass criteria from openspec proposal                 │
│  • Listen test / functional test per spec                       │
│                                                                  │
│  Input:  openspec pass criteria                                 │
│  Output: test results (pass/fail per criterion)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │ Tests pass → triggers Phase 5
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  PHASE 5 — QA AUDIT  (ProductManager skill)                     │
│                                                                  │
│  @ProductManager → updates pm/QA.md                             │
│  • Scores project across 8 categories                           │
│  • P1/P2/P3 issue list                                          │
│  • Build evidence (flash%, RAM%)                                │
│  • Pass/Fail recommendation                                     │
│                                                                  │
│  RETROSPECTIVE — feed findings back:                            │
│  cd pm/                                                         │
│  openspec propose "<P1 issue title from QA.md>"                 │
│  openspec propose "<P2 issue title from QA.md>"                 │
│    → Each issue becomes a scoped proposal for the next cycle    │
│                                                                  │
│  Input:  code + pm/PRD.md + pm/openspec/specs/*                 │
│  Output: pm/QA.md + new openspec proposals (loop → Phase 2)     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Document Ownership

| Document | Lives in | Owned by | Answers |
|----------|----------|----------|---------|
| `pm/PRD.md` | project repo | ProductManager skill | What & why — features, success metrics, constraints, roadmap |
| `pm/openspec/config.yaml` | project repo | openspec | Project context fed to openspec AI automatically |
| `pm/openspec/specs/*.md` | project repo | openspec + Developer | How — data flows, Kconfig, latency/memory impact |
| `pm/openspec/PROPOSAL_TEMPLATE.md` | project repo | openspec | Per-change scope, alternatives, pass criteria |
| `pm/QA.md` | project repo | ProductManager skill | Scores, P1/P2/P3 issues, build evidence |

**Split of technical detail between PRD.md and specs:**
- PRD.md holds the *architecture choice* (e.g. "multi-threaded with data_fifo") and *constraints* (RAM < 93%, latency < 50 ms) but not implementation specifics.
- specs/*.md hold the *implementation detail* (exact Kconfig symbols, byte-level framing, FIFO sizing math).
- The Developer skill needs both — PRD.md for context and non-negotiables, specs for how.

---

## Phase Trigger Rules

| Trigger | Start at Phase |
|---------|---------------|
| New feature idea | 1 — update PRD.md feature table first |
| Bug fix (no new requirements) | 2 — write `openspec propose` directly |
| Architecture change | 1 + 2 — update PRD.md section 2.4 AND the relevant spec |
| QA P1/P2 issue | 2 — `openspec propose "<issue title>"` |
| After any merge to main | 5 — refresh QA.md |
| Before demo or release | 5 — verify all P1s in QA.md are closed |

---

## openspec CLI Quick Reference

All commands run from `pm/` directory:

```bash
cd pm/

# From a PRD.md new feature row:
openspec propose "multi-headset UDP multicast support"

# From a bug:
openspec propose "fix SoftAP inactivity timeout drops headset during quiet audio"

# From a QA.md P1/P2 issue (paste title verbatim):
openspec propose "move CONFIG_BUF_WIFI_RX_PACKET_NUM to Kconfig"

# Refine or generate a technical spec:
openspec spec "audio-pipeline"

# Break an approved proposal into dev tasks:
openspec tasks "fix SoftAP inactivity timeout"
```

openspec reads `pm/openspec/config.yaml` automatically for project context — never paste context manually.

---

## Input → Output Map

```
PRD.md new feature      ──openspec propose──►  pm/openspec/specs/<feature>.md
                                                          │
QA.md P1/P2 issue       ──openspec propose──►            │ (same path)
                                                          │
Bug report              ──openspec propose──►            │ (same path)
                                                          │
Approved spec           ──openspec tasks───►  dev task list
                                                          │
Developer NCS skill  ◄── reads PRD.md + specs/ ──────────┘
        │
        └──► implements code
                    │
                    └──► west build → flash → UART → functional test
                                │
                    ProductManager NCS skill ◄── reads code + PRD + specs
                                │
                                └──► updates QA.md
                                            │
                                            └──► P1/P2 ──► openspec propose (loop)
```

---

## pm/ Directory Structure (reference)

```
pm/
├── PRD.md                          ← Phase 1 output: what & why
├── QA.md                           ← Phase 5 output: audit report
└── openspec/
    ├── config.yaml                 ← Project context (auto-read by openspec)
    ├── PROPOSAL_TEMPLATE.md        ← Template for new proposals
    └── specs/
        ├── architecture.md         ← Thread map, roles, memory budget
        ├── audio-pipeline.md       ← TX/RX data flow, latency, FIFO
        └── wifi-transport.md       ← UDP framing, DNS-SD, nRF70 tuning
```
