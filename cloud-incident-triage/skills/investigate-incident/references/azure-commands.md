# Azure Investigation Commands

All commands are read-only. Replace `<placeholders>`. Confirm the subscription first with `az account show`.

## Gotchas (read first)

### Always pass an explicit time window to `az monitor app-insights query`

The CLI applies a **default timespan of 1 hour that overrides the `ago(...)` in your KQL**. The query below silently returns about 1 hour of data, not 24, with no warning:

```bash
# WRONG: silently truncated to 1 hour despite ago(24h)
az monitor app-insights query --app <app> -g <rg> \
  --analytics-query "exceptions | where timestamp > ago(24h) | summarize count() by bin(timestamp,1h)"

# RIGHT: relative window
az monitor app-insights query --app <app> -g <rg> --offset 48h \
  --analytics-query "exceptions | summarize count() by bin(timestamp,1h) | order by timestamp asc"

# RIGHT: absolute window (best for correlating against a deploy)
az monitor app-insights query --app <app> -g <rg> \
  --start-time 2026-01-15T02:00:00Z --end-time 2026-01-15T07:00:00Z \
  --analytics-query "exceptions | summarize count() by bin(timestamp,10m) | order by timestamp asc"
```

Without an explicit window you cannot build a baseline, so you cannot claim anything is "new". A chronic background error can look like a fresh incident, and a large deploy-caused spike can look like a handful of isolated failures.

### Other gotchas

- `-o tsv` mangles single-value results (e.g. `| count`). Use `-o json`, or `| summarize c=count()`.
- **Subscription scoping is silent.** A resource in another subscription simply does not appear.
- **Lost telemetry is not an outage.** If an app stops reporting, check its `availabilityState` and the gateway/load balancer view of its traffic before concluding it is down.
- **Service Bus sessions:** on session-enabled entities, a session ID mismatch stalls messages without producing errors. Check active message counts, not just dead-letters.

## Identity and context

```bash
az account show -o table
az account set --subscription "<name-or-id>"
```

## Change correlation

```bash
# Control-plane writes in a resource group over the last 24h
az monitor activity-log list -g <rg> --offset 24h \
  --query "[?!contains(operationName.value, '/read')].{time:eventTimestamp, op:operationName.localizedValue, status:status.value, caller:caller}" -o table

# App Service deployment history
az webapp log deployment list -n <app> -g <rg> -o table

# Pipelines (Azure DevOps or GitHub Actions)
az pipelines runs list --organization <org-url> --project <project> --pipeline-id <id> --top 5 \
  --query "[].{id:id, result:result, finish:finishTime, branch:sourceBranch}" -o table
gh run list --workflow <deploy-workflow>.yml --limit 10

# Feature flags in App Configuration
az appconfig feature list -n <appconfig-name> --query "[].{name:name, state:state}" -o table
az appconfig revision list -n <appconfig-name> --datetime "<start-iso>" --top 20
```

## Resource health

```bash
# Anything not "Available" across the subscription (requires the resource-graph extension)
az graph query -q "HealthResources
  | where type =~ 'microsoft.resourcehealth/availabilitystatuses'
  | where properties.availabilityState != 'Available'
  | project id, state=properties.availabilityState, reason=properties.reasonType"

# Platform metrics for any resource
az monitor metrics list --resource <resource-id> --metric "Http5xx" --interval PT5M --offset 2h -o table
```

## Compute

```bash
az webapp show -n <app> -g <rg> --query "{state:state, availability:availabilityState}"
az functionapp show -n <app> -g <rg> --query "{state:state, availability:availabilityState}"
az webapp log tail -n <app> -g <rg>

az containerapp show -n <app> -g <rg> --query "{running:properties.runningStatus, revision:properties.latestRevisionName}"
az containerapp revision list -n <app> -g <rg> --query "[].{name:name, active:properties.active, health:properties.healthState}" -o table

# AKS: cluster state, then use kubernetes-commands.md
az aks show -n <cluster> -g <rg> --query "{power:powerState.code, provisioning:provisioningState, version:kubernetesVersion}"
az aks get-credentials -n <cluster> -g <rg>

# Application Gateway backend health (why 502?)
az network application-gateway show-backend-health -n <gw> -g <rg> \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{address:address, health:health}"
```

## Data stores

```bash
az sql db show -g <rg> -s <server> -n <db> --query "{status:status, tier:currentSku.tier, capacity:currentSku.capacity}"
az cosmosdb show -g <rg> -n <account> --query "{state:provisioningState, locations:readLocations[].locationName}"
```

## Messaging

```bash
# Service Bus topic subscription: active vs dead-letter
az servicebus topic subscription show -g <rg> --namespace-name <ns> --topic-name <topic> --name <sub> \
  --query "{active:countDetails.activeMessageCount, deadLetter:countDetails.deadLetterMessageCount}"

# Service Bus queue
az servicebus queue show -g <rg> --namespace-name <ns> -n <queue> \
  --query "{active:countDetails.activeMessageCount, deadLetter:countDetails.deadLetterMessageCount}"
```

## Secrets (names and expiry only, never values)

```bash
az keyvault secret list --vault-name <vault> --query "[].{name:name, enabled:attributes.enabled, expires:attributes.expires}" -o table
az keyvault certificate list --vault-name <vault> --query "[].{name:name, expires:attributes.expires}" -o table
```

---

## KQL Query Library (Application Insights)

Run with `az monitor app-insights query --app <app> -g <rg> --offset <window> --analytics-query "<kql>"`. The `--offset` sets the window, so the queries below do not repeat `ago(...)`.

### Exceptions

```kql
// Top exceptions
exceptions | summarize count() by outerMessage, problemId | top 10 by count_

// Exception detail with stack
exceptions | where outerMessage contains "<term>"
| project timestamp, outerMessage, innermostMessage, details[0].rawStack | top 20 by timestamp desc

// Which app/role is throwing
exceptions | summarize count() by cloud_RoleName, outerMessage | top 20 by count_
```

### Requests

```kql
// Failed requests by endpoint
requests | where success == false | summarize count() by name, resultCode | top 20 by count_

// Failure rate over time
requests | summarize total=count(), failed=countif(success == false) by bin(timestamp, 5m)
| extend failRate = round(100.0 * failed / total, 2) | order by timestamp asc

// Latency percentiles
requests | summarize p50=percentile(duration,50), p95=percentile(duration,95), p99=percentile(duration,99) by name
| top 20 by p99 desc

// Auth failures
requests | where resultCode in ("401","403") | summarize count() by name, resultCode | top 20 by count_
```

### Dependencies (SQL, HTTP, queues, Cosmos DB)

```kql
// Failed dependency calls
dependencies | where success == false | summarize count() by type, target, name, resultCode | top 20 by count_

// Slow dependencies (>3s)
dependencies | where duration > 3000 | summarize count(), avg(duration) by type, target, name | top 20 by count_

// Cosmos DB throttling (429) and RU-heavy operations
dependencies | where type == "Azure DocumentDB"
| extend ru = todouble(customDimensions["RequestCharge"])
| summarize calls=count(), throttled=countif(resultCode == "429"), avgRU=avg(ru) by name | top 20 by throttled desc
```

### Traces

```kql
// Warnings and errors
traces | where severityLevel >= 3 | project timestamp, message, severityLevel, cloud_RoleName | top 50 by timestamp desc
```
