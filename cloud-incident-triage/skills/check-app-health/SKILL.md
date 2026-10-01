---
name: check-app-health
description: 'Use when the user asks to check health, verify status, run a quick infrastructure sweep, or confirm "is everything OK" for applications running on AWS or Azure, including after a deploy or before a release.'
argument-hint: '"prod", "nonprod", a specific service/layer name, or a tag like "app=checkout"'
---

# Check App Health

A fast, read-only sweep across an application's cloud resources that ends in a single severity-sorted report. For deep root-cause work on a specific problem, use the `investigate-incident` skill instead.

**All commands are read-only.** Do not restart, scale, or redrive anything during a health check. Recommend it in the report instead.

## Workflow

### Step 1: Determine scope

Infer from the request, or ask:

- **Provider:** AWS or Azure (or both)
- **Environment:** default to **production** if the user just says "check health"
- **Layer:** all resources, or one service/component

Read the workspace inventory file if present (e.g. `sre/inventory.md`, see `templates/inventory-template.md`) for account/subscription, regions, and resource names. Without one, discover resources by tag or name prefix:

```bash
# AWS: everything tagged for this app/env in the current region
aws resourcegroupstaggingapi get-resources --tag-filters Key=env,Values=prod Key=app,Values=<app> \
  --query 'ResourceTagMappingList[].ResourceARN' --output text

# Azure: everything in the app's resource group(s)
az resource list -g <rg> --query "[].{name:name, type:type}" -o table
```

### Step 2: Set context

```bash
aws sts get-caller-identity        # AWS: confirm account; pass --region on every call
az account set --subscription "<name-or-id>" && az account show -o table   # Azure
```

If authentication fails, stop and tell the user. Do not retry in a loop.

### Step 3: Sweep each resource type

Run the checks for the provider in scope:
- **AWS:** [references/aws-health.md](./references/aws-health.md)
- **Azure:** [references/azure-health.md](./references/azure-health.md)

Classify every resource with these thresholds:

| Check | Healthy | Warning | Critical / Down |
|---|---|---|---|
| Compute state | Running and healthy | Running but degraded / some targets unhealthy | Stopped, no healthy targets, crash looping |
| Request failure rate (last 30m) | < 1% | 1% to < 5% | >= 5% |
| Dead-letter / DLQ messages | 0 | 1 to 99 | >= 100 |
| Queue backlog age | Within normal | Growing | Oldest message older than its SLA |
| Database state | Available / Online | Maintenance pending, high CPU (> 80%) | Not available / failover in progress |
| Active alarms | None | Low-severity alarms firing | Any high-severity alarm firing |
| Certificates / secrets | > 30 days to expiry | 7 to 30 days | < 7 days or expired |

Use the same 30-minute window everywhere, and pass it explicitly (see the gotchas in each reference file).

### Step 4: Present report

```markdown
## Health Check Report: {environment} ({provider}, {account/subscription}, {region}) at {UTC timestamp}

### Summary

| Status | Count |
|---|---|
| Critical | X |
| Down | X |
| Warning | X |
| Healthy | X |

### Details

| Resource | Type | Location | Status | Notes |
|---|---|---|---|---|
| checkout-api | ECS service | us-east-1 | Critical | 0/3 tasks running, last stop: OOMKilled |
| orders-sub | Service Bus subscription | rg-orders | Warning | 12 dead-letter messages |
| orders-db | Azure SQL | rg-orders | Healthy | Online |

### Action Items

{For every non-Healthy resource: what to do next. For Critical/Down, an immediate action and a pointer to run `investigate-incident`.}
```

**Report rules:**
- Sort by severity: Critical, Down, Warning, Healthy
- Include real numbers (failure %, DLQ counts, running/desired)
- If everything is healthy, say so plainly with the timestamp and what was checked
- List anything you could not check (permissions, missing telemetry) as its own line, never as Healthy

## Common Mistakes

| Mistake | Fix |
|---|---|
| Reporting "Healthy" for a resource that emits no telemetry | Mark it "Unknown: no data" and check upstream health |
| Sweeping only the default region or subscription | Use the inventory or ask; check every region in scope |
| Checking DLQs but not backlog age | A queue with no DLQ messages can still be stuck |
| Using a default query window | Pass an explicit window; defaults are often 1 hour or less |
