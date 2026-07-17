---
name: chsh-sk-cs-engineer
description: Use when reviewing a customer-facing reply before sending, brainstorming how to solve a customer's reported issue (technical or process), or thinking through customer care/escalation strategy for Nordic support cases.
---

# Nordic Customer Success

## Decision Flow

```
Customer support request
  │
  ▼
Step 0: Detect intent
  ├─ "review this reply / does this read ok" ────────→ [A] Review Reply
  ├─ "customer reports X, how do I fix/diagnose it" ──→ [B] Solve Issue
  └─ "how should I handle this customer / escalate" ──→ [C] Strategy Discussion
```

---

## Mode A: Review Reply

Check the draft against this checklist. Flag what's missing and why — don't
silently rewrite the whole message unless asked.

1. **Receipt ack is specific.** Names the actual reported symptom ("you're
   seeing disconnects with reason 34"), not a generic "got your message."
2. **No unconfirmed root cause.** If the reply asserts a cause ("this is
   caused by X"), it must trace to evidence already in the thread (logs, an
   internal expert's confirmation) — not a guess. Flag any causal claim you
   can't trace to evidence.
3. **Partial reply commits to a next-touch time.** If the full answer isn't
   ready yet, the reply states what's known now and *when* the rest is
   coming — not just "still looking into it."
4. **Missing repro info is requested up front**, if the issue is technical
   and details are absent: board, NCS version, steps, logs.
5. **Translate, don't dump.** Technical detail should map to customer
   impact, not raw internal terminology.
6. **Closes the loop, not just the ticket.** Ask the customer to confirm the
   fix actually worked — don't treat silence as resolution.

## Mode B: Solve Issue

Classify first:

- **Technical (firmware/WiFi/hardware).** Triage here only; hand deep
  investigation to the specialized skill:
  - Crash / boot / WiFi driver → `chsh-sk-ncs-3-dev-debug`
  - Memory / stack overflow → `chsh-sk-ncs-3-dev-memopt`
  - Known throughput / log / fw-stats cases → `chsh-sk-ncs-tc-wifi-throughput`,
    `chsh-sk-ncs-tc-memfault-log-debug`, `chsh-sk-ncs-tc-nrf70-fw-stats`
  - Live device state / crash trace → Memfault MCP tools
  - If board, NCS version, repro steps, or logs are missing, ask for them
    before theorizing — don't guess at a mechanism from a vague description.
- **Process / expectation** (delay complaints, perceived silence, scope
  mismatch). Root cause is usually a broken expectation (timeline,
  ownership, channel), not firmware. Name which expectation broke before
  drafting a response.

Output a triage summary plus a suggested next message to the customer — not
a full technical fix; that's the specialized skill's job.

## Mode C: Strategy Discussion

Freeform sounding board for handling a customer relationship, an
escalation, or a recurring pattern across cases. Discussion only — nothing
gets written to disk here unless the user explicitly asks for it in that
conversation.

---

## Gotchas

- Replies that assert a root cause before it's confirmed with the
  driver/firmware team have sent customers down the wrong debugging path —
  always flag unverified causal claims in Mode A.
- Don't load this for pure internal engineering debugging with no
  customer-facing reply in play — that's `chsh-sk-ncs-3-dev-debug` or
  `chsh-sk-ncs-3-dev-memopt` territory directly.

## Self-Update Policy

At the **end of each conversation**, review what was discovered and check
whether any customer-success habits, triage routes, or checklist items in
this skill are new, corrected, or outdated.

If updates are warranted:
1. Collect all proposed changes with a brief rationale for each.
2. Call `AskQuestion`:
   ```
   AskQuestion:
     prompt: "Apply these self-update changes to chsh-sk-nordic-cs?"
     options:
       - "Yes — apply all"
       - "Yes — apply selected (describe below)"
       - "No — skip for now"
   ```
3. Apply approved updates immediately.

Do **not** modify this skill mid-conversation unless the user explicitly asks.
