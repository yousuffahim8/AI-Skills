---
name: investigate-incident
description: 'Use when a user reports a production issue, outage, error spike, timeout, stuck queue, failed deployment, or degraded service running on AWS or Azure, and wants the root cause found. Triggers include pasted error messages, HTTP 5xx/4xx reports, "why is X failing", or "investigate <service>".'
argument-hint: 'Error message, symptom description, or "investigate <component>"'
---

# Investigate Production Incident

Structured, evidence-first incident triage for workloads on **AWS** or **Azure**. Follow the steps in order. Do not jump to a root cause before Step 6.

**Core principle:** facts before hypotheses, and a baseline before calling anything "new".

## Safety Rules (apply to every step)

- **Read-only by default.** Only run commands that describe, list, get, show, query, or tail.
- **Mutations need explicit approval.** Any command that restarts, scales, deletes, updates, sets, puts, redrives, purges, or deploys must be shown to the user first with its expected impact, and run only after they say yes.
- **Never print secrets.** Do not output connection strings, keys, tokens, or app-setting values. List names only, or mask values.
- **Do not retry failing auth calls in a loop.** If the cloud CLI is not authenticated, stop and tell the user (see Step 2).

## Workflow

### Step 1: Intake

Ask for all four in a single message. If the user already gave some, only ask for what is missing:

1. **Symptom:** error message, status code, failing endpoint, stuck job, timeout, UI error
2. **Start time:** timestamp, "since the last deploy", or "intermittent since {date}"
3. **Blast radius:** all users, one tenant/customer, one region, one feature
4. **Recent changes:** deploy, config change, feature flag, infra change, certificate/secret rotation

### Step 2: Identify the cloud and confirm access

Detect the provider from the user, an inventory file, or the error text, then confirm identity:

| Provider | Check | Then set context |
|---|---|---|
| AWS | `aws sts get-caller-identity` | `--region` on every call, or `export AWS_REGION=...` / `--profile` |
| Azure | `az account show -o table` | `az account set --subscription "<name-or-id>"` |

If the call fails (expired login, missing permissions, no network): tell the user what is blocked, suggest `aws sso login` / `az login`, and offer to continue from pasted logs or screenshots plus source code analysis.

### Step 3: Map symptoms to components

If the workspace has an inventory file (e.g. `sre/inventory.md`, see `templates/inventory-template.md`), read it for resource names, regions/subscriptions, and owning repos. Otherwise discover resources by tag or name prefix and offer to record them in a new inventory afterward.

| Symptom area | AWS services | Azure services |
|---|---|---|
| HTTP 5xx / timeouts at the edge | ALB/NLB, API Gateway, CloudFront, WAF | Front Door, Application Gateway, API Management, WAF |
| App/container crashes | ECS, EKS, EC2, Lambda, Elastic Beanstalk | App Service, Functions, AKS, Container Apps, VMs |
| Database errors / slowness | RDS, Aurora, DynamoDB | Azure SQL, Cosmos DB, PostgreSQL/MySQL Flexible Server |
| Stuck or lost messages | SQS (+ DLQ), SNS, EventBridge, Kinesis | Service Bus (+ dead-letter), Event Grid, Event Hubs |
| Auth failures (401/403) | IAM, Cognito, Secrets Manager, KMS | Entra ID, Key Vault, managed identity |
| Feature behaving "off" | AppConfig, Parameter Store | App Configuration (feature flags) |

### Step 4: Check change correlation

Before deep diving, look for a change inside the incident window:

- **Deploys:** CI/CD run history (`gh run list`, `az pipelines runs list`, `aws codepipeline list-pipeline-executions`)
- **Control-plane changes:** AWS CloudTrail write events, Azure Activity Log
- **Config/flags:** AppConfig / Parameter Store history, Azure App Configuration revisions

Commands are in [aws-commands.md](./references/aws-commands.md) and [azure-commands.md](./references/azure-commands.md). A change whose timestamp matches the symptom start is a lead, not a conclusion. Verify it in Step 5.

### Step 5: Gather evidence

Use the provider reference for exact commands:
- **AWS:** [references/aws-commands.md](./references/aws-commands.md) (CloudWatch Logs Insights, metrics, alarms, ECS/EKS/Lambda, RDS, SQS DLQ, target health)
- **Azure:** [references/azure-commands.md](./references/azure-commands.md) (Application Insights KQL, metrics, App Service/Functions, SQL, Cosmos, Service Bus dead-letter)
- **Kubernetes (EKS or AKS):** [references/kubernetes-commands.md](./references/kubernetes-commands.md)

Always collect, for the incident window **and** an equal-length baseline window before it:
1. Error count and top error messages
2. Request failure rate and latency (p50/p95/p99)
3. Dependency failures (DB, queue, downstream HTTP)
4. Resource state (running/healthy, restarts, throttling, saturation)

**Lost telemetry is not an outage.** If an app stops reporting, check it from upstream (load balancer, gateway, target health) before concluding it is down.

### Step 6: Analyze source code

When evidence points at a specific component:
1. Find the owning repo (inventory file or user)
2. Prefer a local clone; otherwise read via `gh` or a GitHub tool
3. Trace from the entry point (handler, controller, function trigger) through service and data layers to the failing call
4. Explain why the observed error happens, citing file and line

### Step 7: Report

Save to `investigations/{YYYY-MM}/{env}/investigation-{short-slug}.md` (env = `prod`, `nonprod`, `security`, or `perf`). Use this structure:

```markdown
## Findings

**Symptom:** {what was reported}
**Provider / scope:** {AWS account + region, or Azure subscription + resource group}
**Affected component:** {cloud resource + code component}
**Timeline:** {start time, correlated changes}
**Evidence:** {queries run, counts vs baseline, log lines, metrics}

## Root Cause

{Explanation supported by the evidence above. Label it "Hypothesis" if not yet confirmed.}

## Recommended Actions

1. **Immediate mitigation:** {least disruptive fix to restore service, with exact command}
2. **Root cause fix:** {code/config change: file, function, line, or resource setting}
3. **Preventive measure:** {alert, dashboard, test, or guardrail to add}

## Open Questions

{Remaining unknowns and how to resolve them}
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Calling a failure "new" from current volume alone | Count it in a baseline window before the change |
| Querying the wrong AWS region or Azure subscription | Confirm context in Step 2; pass `--region` / `--subscription` explicitly |
| Trusting a short default query window | Always pass an explicit time range (see provider references) |
| Treating missing telemetry as downtime | Check upstream health (LB targets, gateway logs) |
| Blaming the platform when a feature flag is off | Check flag/config state in Step 4 |
| Recommending a restart before finding the cause | Capture evidence (logs, dumps, events) first, then mitigate |
