# SRE Skills

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

## How to use it

After you add the plugin, just describe the job in your own words, in Claude chat, Cowork or Claude Code.
Claude picks the right skill and asks for anything it needs.

## What it runs and what it sends

Nothing. The plugin is instructions only: Markdown skills with no MCP servers, hooks, scripts or
network calls. It collects, stores and sends no data. What you type is handled by Claude as usual.

## Links

- Website and guides: https://github.com/polytheseus/sre-skills
- Source and other AI tools (ChatGPT, Gemini, Codex, Cursor and more): https://github.com/polytheseus/sre-skills
- Help or a bug: https://github.com/polytheseus/sre-skills/issues or support@polytheseus.com
- A paid Pro pack with more skills is described on the website.

Made by polytheseus. MIT licence.
