# Cloud Incident Triage

An SRE agent and three skills for AI coding assistants (Claude Code, and any tool that reads `SKILL.md` skills) that triage production issues on **AWS** and **Azure**, including .NET multi-threading bugs.

## Contents

```
agents/
  sre.md                              Agent: routes to the skills, enforces safety rules
skills/
  investigate-incident/               Evidence-first incident triage and root-cause report
    references/aws-commands.md        CloudWatch, CloudTrail, ECS/EKS/Lambda, RDS, SQS
    references/azure-commands.md      App Insights KQL library, Activity Log, App Service, Service Bus
    references/kubernetes-commands.md EKS/AKS pod, rollout, and event triage
  check-app-health/                   Read-only health sweep with severity thresholds
    references/aws-health.md
    references/azure-health.md
  triage-dotnet-concurrency/          Hangs, deadlocks, thread pool starvation, context bleed, races
    references/runtime-diagnostics.md dotnet-counters, dotnet-dump, per-host capture
    references/code-patterns.md       13 anti-patterns with grep patterns and fixes
    references/repro-test.md          Deterministic concurrency repro recipe (verified xUnit example)
templates/
  inventory-template.md               Per-workspace map of accounts, resources, pipelines
```

## Design principles

- **Read-only by default.** Every mutating action needs explicit user approval.
- **Evidence before hypothesis, baseline before "new".** Reports separate proven causes from hypotheses.
- **Prove concurrency bugs with a test.** Logs show symptoms; a deterministic test proves the mechanism and validates the fix.
- **Gotchas are first-class.** Each reference opens with the traps that cause wrong conclusions (silent default query windows, region/subscription scoping, lost telemetry mistaken for outages).

## Install (Claude Code)

```bash
# Personal (all projects)
cp -r skills/* ~/.claude/skills/
cp agents/sre.md ~/.claude/agents/

# Or per project
cp -r skills/* .claude/skills/
cp agents/sre.md .claude/agents/
```

Then ask: `investigate 502s on checkout-api in prod`, `check health of prod`, or `the worker hangs under load`.

## Prerequisites

AWS CLI v2 and/or Azure CLI (with the `resource-graph` extension), `kubectl` for clusters, `gh` for deploy history, and the .NET diagnostic tools (`dotnet-counters`, `dotnet-dump`, `dotnet-stack`) for concurrency triage.
