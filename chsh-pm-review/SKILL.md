---
name: chsh-pm-review
description: Review NCS project quality and generate QA reports. Use when validating a project against its PRD, running automated checks, or filling out a QA report. Reads docs/product/PRD.md (single canonical PRD with Revision History table).
---

# NCS Project Review Skill

Quality assurance review for Nordic NCS projects. Validates implementation against `docs/product/PRD.md` (business requirements) and `docs/engineering/specs/` (technical specs).

## Quick Start

```bash
# Automated check
~/.claude/skills/chsh-pm-review/check_project.sh /path/to/project

# Manual QA report
# 1. Copy QA template
cp ~/.claude/skills/chsh-pm-review/QA_TEMPLATE.md /path/to/project/docs/qa/QA.md
# 2. Follow CHECKLIST.md
# 3. Fill out QA_TEMPLATE.md with findings
```

## Files in This Skill

| File | Purpose |
|------|---------|
| `check_project.sh` | Automated project health check script |
| `CHECKLIST.md` | Manual review checklist (100-point scale) |
| `QA_TEMPLATE.md` | Template for QA report output |
| `IMPROVEMENT_GUIDE.md` | How to feed review findings back into templates |

## Review Process

1. **Automated check** — run `check_project.sh` first (~1 min)
2. **Manual review** — follow `CHECKLIST.md` systematically
3. **Generate report** — fill `QA_TEMPLATE.md` with findings
4. **Address findings** — fix Critical issues before release
5. **Improve templates** — document lessons in `IMPROVEMENT_GUIDE.md`

## Scoring (100-point scale)

| Score | Grade |
|-------|-------|
| 90–100 | Excellent — ready for release |
| 80–89 | Good — minor improvements needed |
| 70–79 | Satisfactory — notable issues |
| 60–69 | Needs work — significant problems |
| < 60 | Fail — major rework required |

## Document Conventions

- **PRD**: `docs/product/PRD.md` — single file with Revision History table. Use the latest revision date to know what version was active during this QA run.
- **Specs**: `docs/engineering/specs/*.md` — each spec also has a Revision History table.
- **QA output**: `docs/qa/QA-YYYY-MM-DD.md` — dated files, one per review run (each is a standalone audit snapshot).

## Related Skills

- `chsh-pm-prd` — create/update `docs/product/PRD.md`
- `chsh-dev-spec` — create/update engineering specs in `docs/engineering/specs/`
- `chsh-dev-project` — implement code from specs
