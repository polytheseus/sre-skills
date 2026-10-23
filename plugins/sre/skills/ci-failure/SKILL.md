---
name: ci-failure
description: Diagnose a failing CI pipeline — pull the failing job's logs, find the first real error, decide whether it's a genuine regression, a flaky test, or an infrastructure problem, and propose the fix. Use when the user says CI/the build/a check/GitHub Actions is red, failing or flaky.
---

# CI failure diagnosis

The goal is a **verdict** with evidence, followed by a fix:

- **Regression**: the code change broke something. Fix the code (or the test, if the test was wrong).
- **Flaky**: it fails intermittently with no relevant change. Quarantine it and file the flake with evidence.
- **Infra/environment**: runner, network, registry, cache, secret, quota or
  dependency drift. Fix the pipeline, not the code.

## 1. Find the failing run

GitHub Actions (adapt the commands for GitLab `glab`, CircleCI, Buildkite and so on):

```bash
gh pr checks                                   # on a PR branch
gh run list --branch <branch> --limit 10
gh run view <run-id>                           # jobs and steps, with failures marked
gh run view <run-id> --log-failed > /tmp/ci-failed.log
wc -l /tmp/ci-failed.log
```

Logs are long, so don't read them top to bottom. Search for the **first**
error. Later errors are usually cascades of the first one:

```bash
grep -nE '(^|\s)(error|ERROR|Error:|FAIL|FAILED|panic:|Traceback|exit code [1-9]|Exception)' /tmp/ci-failed.log | head -40
```

Then read about 40 lines of context around the first hit.

## 2. Classify

Ask these in order:

1. **Does it fail the same way on a re-run?**
   `gh run rerun <run-id> --failed` (this costs CI minutes, so ask first if the pipeline is long or expensive).
   If it passes on re-run without any change, it's flaky or infra.
2. **Does it fail on the base branch too?**
   `gh run list --branch main --workflow <wf> --limit 5`. If main is also red
   with the same error, this PR didn't cause it.
3. **Does the error relate to the diff?**
   `gh pr diff` or `git diff origin/main...HEAD --stat`. A failing test in a
   module the PR touched is most likely a regression.
4. **Did something outside the repo change?** Floating dependency versions
   (`^`, `~`, `latest`, unpinned actions, `:latest` images), expired tokens,
   a runner image update (`ubuntu-latest` moves), rate limits, or a service
   outage (`githubstatus.com`, the registry's status page).

### Signatures

| Log signal | Usually |
|---|---|
| Assertion failure in a test touched by the diff | Regression |
| Timeout, `ECONNRESET`, `502`, `429`, `rate limit`, DNS failure | Infra / flaky network |
| Test fails at random, order-dependent, depends on time or date | Flaky: shared state, real clock, unawaited async, port collisions |
| `No space left on device`, OOM kill, exit 137 | Runner resources |
| `Resource not accessible by integration`, 401/403 | Token permissions; forks don't receive secrets |
| Lockfile mismatch, `npm ci` / `pip` resolution error | Dependency drift or an uncommitted lockfile |
| Passes locally, fails in CI | Env differences: OS, timezone, locale, case-sensitive FS, tool versions, missing service container |

## 3. Reproduce locally (when it's a regression)

Run only the failing target, using the CI's command (copy it from the workflow
file, not from memory):

```bash
grep -nE 'run:' .github/workflows/<wf>.yml
```

Match the versions the CI uses (`node --version`, `python --version`, the
image tag). For flakiness, loop it, for example
`for i in $(seq 20); do <test cmd> || break; done`, and run it with the
test runner's random-order or seed option.

## 4. Fix

- **Regression** → make the minimal code fix. Run the failing test plus its
  neighbours, then push.
- **Flaky** → find the nondeterminism (time, randomness, ordering, network,
  shared fixtures). If you can't fix it now, quarantine it in the way the repo
  already does (skip marker plus an issue link) rather than adding blind retries.
- **Infra** → pin what drifted (action SHAs, image digests, tool versions),
  fix the permissions in the workflow's `permissions:` block, add caching, or
  move to a bigger runner.

Never "fix" CI by deleting the assertion, raising every timeout, or adding
`continue-on-error` to hide the failure.

## 5. Report

```
Verdict:  Regression | Flaky | Infra
Evidence: <first error line + link/run id>, <rerun result>, <base-branch status>
Cause:    <one or two sentences>
Fix:      <diff or pipeline change>
Follow-up:<issue to file, pinning to add, flake tracker entry>
```
