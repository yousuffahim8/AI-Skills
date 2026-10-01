# SRE Inventory

Copy to `sre/inventory.md` in your workspace and fill it in. The SRE agent reads it to find the right account, region, and resource names, and adds resources it discovers.

## Environments

| Environment | Provider | Account / Subscription | Region(s) | Resource group / tag filter |
|---|---|---|---|---|
| prod | AWS | `123456789012` (profile `prod`) | us-east-1, us-west-2 | `env=prod, app=<app>` |
| prod | Azure | `<subscription-name>` | eastus2 | `rg-<app>-prod` |
| nonprod | Azure | `<subscription-name>` | eastus2 | `rg-<app>-dev` |

## Services

| Service | Owning repo | Compute | Data stores | Messaging | Telemetry |
|---|---|---|---|---|---|
| orders-api | `<org>/orders-api` | ECS service `orders-api` | RDS `orders-db` | SQS `orders-queue` (+ DLQ) | CloudWatch log group `/ecs/orders-api` |
| billing-worker | `<org>/billing` | Azure Functions `func-billing-prod` | Azure SQL `sql-billing/billing` | Service Bus `sb-billing`, topic `invoices` | App Insights `appi-billing-prod` |

## Deploy pipelines

| Service | Pipeline | Where to see run history |
|---|---|---|
| orders-api | GitHub Actions `deploy.yml` | `gh run list --workflow deploy.yml -R <org>/orders-api` |
| billing-worker | Azure DevOps pipeline `<id>` | `az pipelines runs list --pipeline-id <id>` |

## Feature flags and config

| Service | Store |
|---|---|
| orders-api | AWS AppConfig app `<name>` |
| billing-worker | Azure App Configuration `appcs-billing-prod` |

## Known behaviors

Facts that prevent wrong conclusions, e.g. "Service Bus topic `invoices` is session-enabled: stuck messages produce no errors", or "nightly cleanup jobs run at 03:00 UTC and collide with backfill".
