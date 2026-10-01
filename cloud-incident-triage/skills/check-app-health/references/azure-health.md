# Azure Health Sweep

Read-only. Set the subscription first: `az account set --subscription "<name-or-id>"`.

**Always pass `--offset` to `az monitor app-insights query`.** The CLI defaults to a 1-hour window that silently overrides `ago(...)` inside the KQL.

## Resource Health (start here: one call for the whole subscription)

```bash
# Requires: az extension add --name resource-graph
az graph query -q "HealthResources
  | where type =~ 'microsoft.resourcehealth/availabilitystatuses'
  | where properties.availabilityState != 'Available'
  | project id, state=properties.availabilityState, reason=properties.reasonType" -o table
```

## Active alerts

```bash
az graph query -q "AlertsManagementResources
  | where properties.essentials.monitorCondition == 'Fired'
  | project name, severity=properties.essentials.severity, target=properties.essentials.targetResourceName, fired=properties.essentials.startDateTime" -o table
```

## App Service and Function Apps

```bash
az webapp list -g <rg> --query "[].{name:name, state:state, availability:availabilityState}" -o table
az functionapp list -g <rg> --query "[].{name:name, state:state, availability:availabilityState}" -o table
```

| Healthy | Degraded | Down |
|---|---|---|
| `state == Running` and `availabilityState == Normal` | `Running` but `availabilityState != Normal` | `state != Running` |

## Container Apps and AKS

```bash
az containerapp list -g <rg> --query "[].{name:name, running:properties.runningStatus, provisioning:properties.provisioningState}" -o table
az aks list -g <rg> --query "[].{name:name, power:powerState.code, provisioning:provisioningState}" -o table
```

For AKS pod-level health see `investigate-incident/references/kubernetes-commands.md`.

## Application Insights: failure rate and exception spikes

```bash
az monitor app-insights query --app <name> -g <rg> --offset 30m \
  --analytics-query "requests | summarize total=count(), failed=countif(success == false) by cloud_RoleName | extend failRate=round(100.0 * failed / total, 2) | order by failRate desc"

az monitor app-insights query --app <name> -g <rg> --offset 30m \
  --analytics-query "exceptions | summarize count() by cloud_RoleName, outerMessage | top 5 by count_"
```

## Service Bus dead-letters

```bash
# Queues
az servicebus queue list -g <rg> --namespace-name <ns> \
  --query "[].{name:name, active:countDetails.activeMessageCount, deadLetter:countDetails.deadLetterMessageCount}" -o table

# Topic subscriptions
for t in $(az servicebus topic list -g <rg> --namespace-name <ns> --query "[].name" -o tsv); do
  az servicebus topic subscription list -g <rg> --namespace-name <ns> --topic-name "$t" \
    --query "[].{topic:'$t', sub:name, active:countDetails.activeMessageCount, deadLetter:countDetails.deadLetterMessageCount}" -o table
done
```

A high `active` count with zero dead-letters can still mean a stuck consumer (or a session ID mismatch on session-enabled entities). Sample twice a few minutes apart.

## Databases

```bash
az sql db list -g <rg> -s <server> --query "[].{name:name, status:status, tier:currentSku.tier}" -o table
az cosmosdb list -g <rg> --query "[].{name:name, state:provisioningState}" -o table
az postgres flexible-server list -g <rg> --query "[].{name:name, state:state}" -o table
```

Healthy: SQL `Online`, Cosmos `Succeeded`, PostgreSQL `Ready`. Also check SQL `cpu_percent` and Cosmos 429s:

```bash
az monitor metrics list --resource <sql-db-resource-id> --metric cpu_percent --interval PT5M --offset 30m --aggregation Maximum -o table
az monitor metrics list --resource <cosmos-resource-id> --metric TotalRequests --filter "StatusCode eq '429'" --interval PT5M --offset 30m -o table
```

## Gateways

```bash
az network application-gateway show-backend-health -n <gw> -g <rg> \
  --query "backendAddressPools[].backendHttpSettingsCollection[].servers[].{address:address, health:health}" -o table
```

## Key Vault expiry (names and dates only)

```bash
az keyvault certificate list --vault-name <vault> --query "[].{name:name, expires:attributes.expires}" -o table
az keyvault secret list --vault-name <vault> --query "[?attributes.expires].{name:name, expires:attributes.expires}" -o table
```
