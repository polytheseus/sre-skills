---
name: incident-triage
description: Run the first 15 minutes of a production incident — establish impact, find what changed, pick a mitigation, and keep a timeline. Use when the user says something is down, erroring, slow, paging, on fire, or asks "what's going on with prod".
---

# Incident triage

You are the incident's first responder alongside the user. Your job in the first
15 minutes is **not** to find the root cause. It is to:

1. Establish what is broken and for whom (impact).
2. Find what changed (most incidents are caused by a change).
3. Get to a mitigation (stop the bleeding), then hand over to diagnosis.
4. Keep a timestamped timeline the whole way, so the postmortem writes itself.

## Ground rules

- **Read-only by default.** Gather evidence with commands that don't change state.
  Before any mutating action (rollback, restart, scale, flag flip, failover,
  `kubectl delete`, `terraform apply`), state the exact command, what it will
  do, its blast radius, and how to undo it, then wait for the user to say go.
- **Mitigate before you explain.** If a recent deploy lines up with the start
  of the problem, recommend rolling it back *before* proving it's the cause.
  Rollbacks are cheap; extended outages are not.
- **Don't trust a single signal.** Confirm user-facing impact from at least two
  sources (e.g. error-rate metric + synthetic check, or logs + a real request).
- **Timestamps in UTC.** Note the time of every observation and action.

## Step 1 — Impact (≤ 3 min)

Answer these, writing down the evidence for each:

| Question | How |
|---|---|
| What is the symptom? | Errors (which codes?), latency, wrong data, total outage |
| Who is affected? | All users / one region / one tenant / one endpoint |
| Since when? | First bad datapoint in metrics; first error in logs |
| Is it getting worse? | Compare the last 5 min with the previous 5 min |

Reproduce it yourself if you can do so safely:

```bash
curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://<endpoint>
```

Assign a severity using the team's scale if they have one. Otherwise:
- **SEV1**: core function down or data loss/exposure for many users
- **SEV2**: major degradation or a key feature down; workaround exists
- **SEV3**: minor degradation, small subset of users

## Step 2 — What changed? (≤ 5 min)

Look at the window from ~2 hours before the start until now. Check whichever
of these exist in this environment:

```bash
# Deploys / commits
git log --since="3 hours ago" --oneline --all
gh run list --limit 20                       # recent CI/CD runs
kubectl rollout history deploy/<name> -n <ns>
kubectl get events -A --sort-by=.lastTimestamp | tail -40
helm history <release> -n <ns>

# Infra / config
terraform state list >/dev/null && git log --since="1 day ago" -- '*.tf'
```

Also ask the user about changes that leave no trace in this shell: feature
flags, config consoles, DNS/CDN changes, certificate expiry, cloud provider
status pages, traffic spikes (launches, marketing emails, crawlers), upstream
vendors.

Line the changes up against the start time of the symptom. A change that
landed 0–15 minutes before impact is the prime suspect.

## Step 3 — Mitigate

Pick the cheapest, most reversible option that should stop the impact:

1. Roll back the suspect deploy/config/flag.
2. Shift traffic away (failover region, drain bad nodes, disable a feature).
3. Add capacity (scale out) if it's saturation.
4. Shed load (rate limit, disable an expensive endpoint) to protect the core.
5. Restart — only if evidence points to a stuck state, and capture
   diagnostics first (thread dump, `kubectl logs --previous`, heap/core if cheap),
   because a restart destroys evidence.

For the chosen action, present:

```
ACTION:    <exact command>
EXPECTED:  <what should improve, and where we'll see it>
RISK:      <blast radius>
UNDO:      <exact command>
```

Wait for approval, run it, then **verify** with the same signals used in Step 1.
If it didn't help within a few minutes, say so plainly and move to the next
option.

## Step 4 — Communicate

Draft a status update the user can paste into the incident channel or status
page. Keep it factual and avoid guessing at cause:

```
[SEV<n>] <service> — <symptom in user terms>
Impact: <who/what>, since <HH:MM UTC>
Status: Investigating | Mitigating | Monitoring | Resolved
Current action: <what we're doing>
Next update: <HH:MM UTC>
```

Suggest an update cadence: every 30 min for SEV1, every hour for SEV2.

## Step 5 — Hand over

Once impact is mitigated, summarise:

- Timeline so far (UTC, observation/action per line)
- What we believe is going on, and how confident we are
- Open questions and the evidence still needed
- Follow-ups that must happen even though things look fine (e.g. re-enable
  the feature, forward-fix the rolled-back change)

Offer to continue with root-cause diagnosis (e.g. the `k8s-debug` or
`log-forensics` skills) and, once resolved, to write the postmortem with the
`postmortem` skill using the timeline you kept.
