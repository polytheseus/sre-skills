---
name: postmortem
description: Write a blameless incident postmortem from a timeline, chat log, alerts and notes, with impact, contributing factors, what went well, and specific owned action items. Use when the user asks for a postmortem, incident review, RCA, or incident report.
---

# Blameless postmortem

A postmortem exists to make the system safer. It isn't there to record who
made a mistake. Write it so that an engineer who wasn't involved can
understand what happened and why the actions will help.

## Inputs to gather

Ask for, or collect from the repo and terminal, whatever exists:
- The incident timeline (from `incident-triage` if it was used), the chat
  channel export, the pager or alert history
- Graphs or numbers for impact: error rate, latency, affected users or requests, revenue
- The changes involved: PRs, deploys, config changes
- Start, detection, mitigation and resolution times

If key facts are missing, write `TODO(owner?): …` in the doc instead of
guessing. **Never invent numbers or timestamps.**

## Blameless language

- Describe actions and the system, not people's character: write "the deploy
  pipeline allowed X". Don't write "Bob forgot X".
- Use roles, not names, in the narrative ("the on-call engineer").
- Treat a human error as the start of the inquiry, not the conclusion. Ask
  why the system made that mistake easy, and why it wasn't caught.
- Avoid counterfactual blame ("should have", "failed to"). State what
  happened and what information was available at the time.

## Analysis

Look for **contributing factors** rather than a single root cause. Real
incidents usually need several things to go wrong together. Use these lenses:

1. **Trigger**: what change or event started it?
2. **Why did it cause harm?** The missing safeguard: validation, canary, limits, isolation.
3. **Detection**: how long until we knew? Was it an alert or a customer? Why that long?
4. **Response**: what slowed mitigation down? Access, runbooks, tooling, unclear ownership.
5. **Recovery**: why did full recovery take as long as it did?

Use "5 whys" on each lens, and stop when you reach something the team can change.

## Action items

Each action item must be:
- **Specific and verifiable**: write "Add canary step to deploy pipeline for
  service X with automatic rollback on 5xx > 1%". Not "improve deploys".
- **Owned**: a team or a role, and a due date or priority.
- **Classified**: *prevent* (stop recurrence), *detect* (find it faster),
  or *mitigate* (reduce impact or time to recover).

Prefer a few high-leverage items to a long wish list. Mark the one or two
that matter most.

## Template

Produce the document in this structure (Markdown):

```markdown
# Postmortem: <short title>  —  <YYYY-MM-DD>

**Status:** Draft | In review | Final
**Severity:** SEV<n>    **Duration:** <detect→resolve>    **Authors:** <roles>

## Summary
<3–5 sentences: what happened, impact, cause in brief, how it was resolved.>

## Impact
- Users/customers affected: <number or %>, <which>
- Duration of user impact: <HH:MM–HH:MM UTC>
- Business impact: <failed requests, revenue, SLA/SLO budget consumed>

## Timeline (UTC)
| Time | Event |
|---|---|
| HH:MM | <trigger> |
| HH:MM | <first alert / report> — detection |
| HH:MM | <key decision> |
| HH:MM | <mitigation applied> — mitigation |
| HH:MM | <confirmed recovery> — resolution |

## Contributing factors
1. <factor> — <evidence>
2. ...

## What went well
- ...

## What was hard / where we got lucky
- ...

## Action items
| # | Action | Type | Owner | Priority/Due |
|---|---|---|---|---|
| 1 | ... | prevent/detect/mitigate | ... | P1 / YYYY-MM-DD |

## Supporting material
<links to graphs, PRs, alerts, chat>
```

Compute the key durations from the timeline and state them explicitly:
time to detect (trigger→detection), time to mitigate (detection→mitigation),
and total user impact.

After drafting, re-read it once as a skeptical reviewer would. Is any
statement unsupported by evidence? Does any wording assign blame? Would each
action item actually have prevented or shortened this incident? Fix what you
find, then tell the user which TODOs still need their input.
