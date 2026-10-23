---
name: log-forensics
description: Find the signal in large log files or log queries — bucket by time, compare error rates before and after an event, surface new or spiking messages, and correlate request IDs across services. Use when the user has logs to dig through, asks "what's in these logs", or needs to know when and why errors started.
---

# Log forensics

Logs are too big to read. Use them like a database: **count, bucket,
compare, then read a few lines**. Read only the lines your numbers point to.

## Ground rules

- Never `cat` an unknown-size log into the conversation. Check the size first
  (`wc -l`, `ls -lh`) and sample with `head`, `tail`, `shuf -n`.
- Logs often contain PII, tokens and secrets. Quote the minimum needed, and
  redact tokens, emails and IPs in anything you summarise.
- Work in a scratch directory. Don't modify the original files.

## 1. Understand the format

```bash
ls -lh <files>; wc -l <file>
head -3 <file>; tail -3 <file>
```

Identify the timestamp field and format, the level or severity field, and
any request/trace ID. For JSON lines, list the keys:

```bash
head -200 <file> | jq -r 'keys[]' 2>/dev/null | sort | uniq -c | sort -rn
```

For compressed or rotated logs use `zcat`/`zgrep`. For journald, use
`journalctl -u <unit> --since "<time>" --until "<time>" -o short-iso`.
For containers, use `kubectl logs --since=…` or `docker logs --since …`.

## 2. Bucket over time

Count events per minute (adjust the cut to your timestamp format) and look
for the step change:

```bash
# Plain text, ISO timestamps at line start: 2026-01-01T12:34:56...
grep -E 'ERROR|error|Exception' <file> | cut -c1-16 | sort | uniq -c

# JSON lines
jq -r 'select(.level=="error") | .time[0:16]' <file> | sort | uniq -c
```

Find the **first bad minute**. Everything after it is effect. The cause is
usually in the minute or two before it.

## 3. What's new? (before vs after)

Normalise messages so that variable parts (numbers, IDs, IPs, hashes) don't
split identical errors, then diff the two windows:

```bash
norm() { sed -E 's/[0-9a-f]{8}-[0-9a-f-]{27}/<uuid>/g; s/\b[0-9a-f]{12,}\b/<hex>/g; s/[0-9]+(\.[0-9]+){3}/<ip>/g; s/[0-9]+/<n>/g'; }
awk '$0 < "<T_START>"' <file> | grep -iE 'error|warn|exception' | cut -d' ' -f3- | norm | sort | uniq -c | sort -rn > before.txt
awk '$0 >= "<T_START>"' <file> | grep -iE 'error|warn|exception' | cut -d' ' -f3- | norm | sort | uniq -c | sort -rn > after.txt
# Messages present after but not before:
comm -13 <(awk '{$1="";print}' before.txt | sort) <(awk '{$1="";print}' after.txt | sort) | head -20
head -20 after.txt
```

(Adjust the `cut` so it drops the timestamp/level prefix for your format.
The string comparison on `$0` works for ISO-8601 timestamps at the start of the line.)

New message types, and messages whose count jumped by an order of
magnitude, are your leads. Pull 3–5 full examples of each.

## 4. Follow a request

When you have one failing request ID or trace ID, gather its whole story
across files and services and sort it by time:

```bash
grep -h '<request-id>' <files...> | sort
```

Note where the latency or the error first appears. The first service to
log an error is usually closer to the cause than the last.

## 5. Common patterns

| Pattern | Suggests |
|---|---|
| Errors begin exactly at a deploy or restart timestamp | The change. Check for a version or config line at startup |
| Timeouts to one dependency, then cascading 5xx | A downstream outage or pool exhaustion. Check connection pool or thread metrics |
| Errors every N minutes | Cron jobs, cache expiry, token refresh, log rotation |
| Gradual increase over hours | A leak (memory, connections, file descriptors), queue backlog, disk filling |
| One tenant, user or IP dominates | An abusive client, a bad data record, or a hot key |
| `too many open files`, `connection reset`, `broken pipe` | Resource limits or a peer restarting |

## 6. Report

- **When**: the first bad minute (UTC), with the per-minute counts around it
- **What changed in the logs**: the new or spiking messages with their counts, plus one redacted example each
- **Correlation**: a deploy, restart, cron job, traffic or tenant that lines up
- **Hypothesis and confidence**, and the next query that would confirm or refute it
