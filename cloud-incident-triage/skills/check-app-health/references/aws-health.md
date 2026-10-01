# AWS Health Sweep

Read-only. Add `--region <region>` to every call and repeat per region in scope. Time window (GNU date; on macOS use `date -v-30M`):

```bash
START_ISO=$(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ); END_ISO=$(date -u +%Y-%m-%dT%H:%M:%SZ)
```

## Alarms (start here: fastest signal)

```bash
aws cloudwatch describe-alarms --state-value ALARM \
  --query 'MetricAlarms[].{name:AlarmName,metric:MetricName,reason:StateReason}' --output table
```

## Load balancers and API Gateway

```bash
# Unhealthy targets per target group
for tg in $(aws elbv2 describe-target-groups --query 'TargetGroups[].TargetGroupArn' --output text); do
  echo "== $tg"; aws elbv2 describe-target-health --target-group-arn "$tg" \
    --query 'TargetHealthDescriptions[].{target:Target.Id,state:TargetHealth.State}' --output table
done

# 5xx in the window: LB-generated vs app-generated
for m in HTTPCode_ELB_5XX_Count HTTPCode_Target_5XX_Count RequestCount; do
  aws cloudwatch get-metric-statistics --namespace AWS/ApplicationELB --metric-name $m \
    --dimensions Name=LoadBalancer,Value=app/<lb-name>/<lb-id> \
    --start-time "$START_ISO" --end-time "$END_ISO" --period 1800 --statistics Sum \
    --query "Datapoints[0].Sum" --output text | sed "s/^/$m: /"
done
```

Failure rate = (ELB 5xx + Target 5xx) / RequestCount. For API Gateway use `AWS/ApiGateway` with `5XXError` and `Count`, dimension `ApiName`.

## Compute

```bash
# ECS: running vs desired for every service in a cluster
aws ecs describe-services --cluster <cluster> \
  --services $(aws ecs list-services --cluster <cluster> --query 'serviceArns' --output text) \
  --query 'services[].{name:serviceName,desired:desiredCount,running:runningCount}' --output table

# Lambda: errors and throttles in the window
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Errors \
  --dimensions Name=FunctionName,Value=<fn> --start-time "$START_ISO" --end-time "$END_ISO" \
  --period 1800 --statistics Sum
aws cloudwatch get-metric-statistics --namespace AWS/Lambda --metric-name Throttles \
  --dimensions Name=FunctionName,Value=<fn> --start-time "$START_ISO" --end-time "$END_ISO" \
  --period 1800 --statistics Sum

# EC2 status checks
aws ec2 describe-instance-status --include-all-instances \
  --query 'InstanceStatuses[?SystemStatus.Status!=`ok` || InstanceStatus.Status!=`ok`].{id:InstanceId,state:InstanceState.Name,system:SystemStatus.Status,instance:InstanceStatus.Status}' \
  --output table

# EKS: cluster status, then pods (see investigate-incident/references/kubernetes-commands.md)
aws eks describe-cluster --name <cluster> --query 'cluster.status'
```

| Resource | Healthy | Down |
|---|---|---|
| ECS service | `running == desired` | `running == 0` with `desired > 0` |
| Target group | all `healthy` | no `healthy` targets |
| EC2 | both checks `ok` | `impaired` |
| EKS | `ACTIVE` | anything else |

## Data stores

```bash
aws rds describe-db-instances \
  --query 'DBInstances[].{id:DBInstanceIdentifier,status:DBInstanceStatus}' --output table
aws rds describe-pending-maintenance-actions --output table

aws dynamodb describe-table --table-name <table> --query 'Table.TableStatus'
```

Healthy: RDS `available`, DynamoDB `ACTIVE`. Also check `CPUUtilization` (Warning > 80%) and DynamoDB `ThrottledRequests` (Warning > 0).

## Queues

```bash
# For each DLQ: messages waiting
aws sqs get-queue-attributes --queue-url <dlq-url> --attribute-names ApproximateNumberOfMessages \
  --query 'Attributes.ApproximateNumberOfMessages' --output text

# For each main queue: age of oldest message (seconds)
aws cloudwatch get-metric-statistics --namespace AWS/SQS --metric-name ApproximateAgeOfOldestMessage \
  --dimensions Name=QueueName,Value=<queue> --start-time "$START_ISO" --end-time "$END_ISO" \
  --period 300 --statistics Maximum
```

## Certificates

```bash
aws acm list-certificates --query 'CertificateSummaryList[].{domain:DomainName,notAfter:NotAfter,status:Status}' --output table
```
