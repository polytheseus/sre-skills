---
name: terraform-plan-review
description: Review a Terraform/OpenTofu plan before apply — flag destroys and forced replacements, data-loss risk, IAM/permission widening, public exposure, drift and state surprises, and produce a go/no-go summary. Use when the user shares a plan, asks "is this safe to apply", or is about to run terraform/tofu apply.
---

# Terraform plan review

Your output is a **risk-ranked review** of exactly what the plan will do, with
a clear recommendation: *safe to apply*, *apply with care*, or *do not apply*.

## Ground rules

- Never run `terraform apply`, `destroy`, `import`, `state rm/mv`, or
  `force-unlock` yourself unless the user explicitly asks, and even then
  confirm the workspace and target first.
- `plan`, `show`, `validate`, `fmt -check`, `state list/show`, and
  `providers` are fine to run.
- Treat plan output as possibly containing secrets. Don't echo
  sensitive values back to the user.

## 1. Get a machine-readable plan

```bash
terraform workspace show
terraform plan -out=tfplan -input=false -lock=false
terraform show -json tfplan > tfplan.json
```

(Use `tofu` instead of `terraform` if that's what the repo uses. If the user
pasted a text plan, work from that.)

Summarise the actions:

```bash
jq -r '.resource_changes[] | select(.change.actions != ["no-op"])
  | "\(.change.actions | join("/"))\t\(.address)"' tfplan.json | sort | uniq -c | sort -rn
```

Pull the dangerous ones out first:

```bash
# Deletes and replacements (["delete","create"] or ["create","delete"])
jq -r '.resource_changes[] | select(.change.actions | index("delete"))
  | "\(.change.actions | join("/"))\t\(.address)"' tfplan.json

# Which attribute forces each replacement
jq -r '.resource_changes[] | select(.action_reason != null or (.change.replace_paths // [] | length > 0))
  | "\(.address)\t\(.action_reason // "")\t\(.change.replace_paths // [] | tostring)"' tfplan.json
```

## 2. Risk checklist

Rank the findings **High / Medium / Low**.

**High: stop and confirm**
- Destroy or replace of anything stateful: databases, buckets, disks and
  volumes, file systems, queues with messages, KMS keys, DNS zones, and
  identity pools. Note whether `prevent_destroy`, deletion protection, or
  snapshots/backups exist.
- Replacements caused by an innocuous-looking attribute change (name, AZ,
  engine version, subnet, `user_data`). Suggest an in-place alternative or
  a `moved {}` block if it's really a refactor.
- Resource address changes that show up as delete + create because of a
  rename or a module move. The fix is `moved {}`, not an apply.
- IAM: new `*` actions or resources, `AdministratorAccess`/`Owner`/`Editor`,
  new trust relationships or principals (especially cross-account or
  `"*"`), and `iam:PassRole`.
- Network exposure: `0.0.0.0/0` or `::/0` on non-HTTP ports, public IPs,
  public buckets or ACLs, removed `block_public_access`, publicly
  accessible databases, and load balancers changed from internal to external.
- Encryption disabled, logging or audit trails removed, and backup or
  retention reduced.

**Medium**
- In-place changes that cause a restart or downtime (instance type, DB
  parameter groups that need a reboot, node pool changes).
- Changes to shared or core modules that affect many resources at once.
- Provider or module version bumps in the same plan as resource changes.
- Large counts: "47 to change" for what should be a small PR.

**Low**
- Tag and label changes, descriptions, and additive resources with no exposure.

## 3. Drift and state surprises

- Changes the PR didn't intend (diff the `.tf` changes against the plan).
  Unintended changes usually mean drift (someone clicked in the console) or
  version-induced diffs. Call out each one and ask whether to keep or revert it.
- `known after apply` on identifiers that other systems depend on (IPs, ARNs,
  endpoints) means downstream breakage is likely.
- `data` sources reading live values can make plans non-deterministic.

## 4. Report

```
Recommendation: SAFE TO APPLY | APPLY WITH CARE | DO NOT APPLY
Workspace: <name>   Totals: +<add> ~<change> -<destroy> (±<replace> replacements)

HIGH
- <address>: <what happens> — <why it matters> — <suggested fix>
MEDIUM
- ...
LOW / informational
- ...

Unintended changes (drift?):
- ...

Before applying:
- [ ] <backup/snapshot X>
- [ ] <add moved{} / lifecycle prevent_destroy / targeted plan>
- [ ] <maintenance window if downtime>
```

Where a fix is code, give it as a concrete HCL snippet (e.g. the `moved {}`
block, a `lifecycle { prevent_destroy = true }`, or a narrowed IAM policy).
