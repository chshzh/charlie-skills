---
name: chsh-sk-grill-me
description: Use when the user wants to stress-test a plan, decision, or idea before committing to it, or uses any "grill me" trigger phrase.
---

# Grill Me

Interview the user relentlessly about every aspect of the plan, decision, or
idea until reaching shared understanding. Walk down each branch of the
decision tree, resolving dependencies between decisions one at a time. For
each question, provide a recommended answer.

Ask questions one at a time, waiting for feedback on each before continuing
— asking multiple questions at once is bewildering.

If a *fact* can be found by exploring the environment (filesystem, tools,
etc.), look it up rather than asking. The *decisions*, though, are the
user's — put each one to them and wait for their answer.

Do not act on the plan until the user confirms shared understanding has
been reached.

## Gotchas

- Don't confuse with `chsh-sk-cs-engineer` Mode C — that's a
  sounding board for the user's own customer-care strategy; this skill is
  for stress-testing *any* plan/decision by relentless questioning, not
  specific to customer work.
- Source: adapted from `mattpocock/skills` (`grill-me` / `grilling`),
  imported 2026-07-17. Upstream splits this into a thin `grill-me` entry
  point that delegates to a separate `grilling` skill — merged into one
  file here since this vault only needs a single entry point.

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check
whether the interview approach in this skill needs adjustment based on
what worked or didn't during the session.

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Call `AskQuestion`:
   ```
   AskQuestion:
     prompt: "Apply these self-update changes to chsh-sk-grill-me?"
     options:
       - "Yes — apply all"
       - "Yes — apply selected (describe below)"
       - "No — skip for now"
   ```
3. Apply approved updates immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
