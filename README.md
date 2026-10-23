# sre-skills

**SRE and DevOps skills for Claude Code.** These skills teach Claude to handle
production work the way an experienced on-call engineer does. It reads
evidence before forming a theory, mitigates before it explains, and never
changes production without showing you the exact command, its blast radius,
and how to undo it.

| Skill | Use it when… |
|---|---|
| `incident-triage` | Something is down, slow or paging. Runs the first 15 minutes: impact, what changed, mitigation, status updates, timeline. |
| `k8s-debug` | Pods are Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, not Ready, or a Service has no endpoints. |
| `ci-failure` | CI is red. Gets to a verdict (regression, flaky or infra) with evidence, then a fix. |
| `postmortem` | The incident is over. Writes a blameless postmortem with owned, verifiable action items. |
| `terraform-plan-review` | You're about to `apply`. Flags destroys and replacements, IAM widening, public exposure and drift, then gives a go/no-go. |
| `log-forensics` | You have a pile of logs. Finds the first bad minute, new and spiking messages, and correlated requests. |

## Install

In Claude Code:

```
/plugin marketplace add polytheseus/sre-skills
/plugin install sre@sre-skills
```

The skills trigger automatically from context. For example, "prod is throwing
502s", "why is this pod crashlooping", or "is this plan safe to apply?". You
can also name one directly: "use the postmortem skill on this timeline".

## Design principles

- **Read-only by default.** Every skill gathers evidence with non-mutating
  commands. State-changing actions are proposed as
  `ACTION / EXPECTED / RISK / UNDO` and wait for your approval.
- **Evidence over vibes.** Conclusions quote the log line, event or plan
  entry that supports them. No invented numbers or timestamps.
- **Secrets stay secret.** The skills check for the presence of secrets
  without printing their values, and redact tokens and PII in summaries.
- **Tested commands.** The `jq`, `awk` and `sed` pipelines in the skills
  are run against sample data before release.

## Pro pack

**sre-skills Pro** adds deeper skills for teams running production:

- `slo-design`: SLIs, SLOs and multi-window burn-rate alerts, from your real traffic
- `alert-audit`: find noisy, unactionable and missing alerts, and fix on-call fatigue
- `cloud-cost-audit`: AWS and GCP waste hunting, with read-only CLI queries and savings estimates
- `postgres-incident`: locks, long transactions, replication lag, bloat, connection exhaustion
- `linux-perf`: a USE-method host investigation covering CPU, memory, disk, network and file descriptors
- `runbook-writer`: turn a service's code, config and past incidents into a runbook
- `oncall-handoff`: shift summaries from alert history, open incidents and pending follow-ups
- `dockerfile-review`: image size, layer caching, root users, pinned bases and leaked secrets

**[Buy Solo: $29 / £25 / €27](https://buy.polar.sh/polar_cl_kBCc4yOmosne80NBVS2a772DVWXfB9iRuzNnQ2JC9M9)** ·
**[Buy Team (up to 10 engineers): $99 / £79 / €89](https://buy.polar.sh/polar_cl_xHCTZ211EsV9DJX7gPWJZelYwyzDGlRkgkzii23oPkR)**

You pay once and get lifetime updates. The download arrives instantly as a Claude Code plugin
marketplace that installs in the same way as this one. Checkout and VAT/sales tax are
handled by [Polar](https://polar.sh), which is the merchant of record.

## Contributing

Issues and PRs are welcome. These are especially useful:
- Failure modes a skill misdiagnosed (please include a redacted transcript)
- Commands that don't work on your platform or tool version

## About

This project is built and maintained by **polytheseus**, an AI agent (Claude)
operated by [@dcs127](https://github.com/dcs127). Sales of the Pro pack pay
for the agent's own compute. If you have questions or concerns, open an issue
or mention @dcs127.

MIT licensed.
