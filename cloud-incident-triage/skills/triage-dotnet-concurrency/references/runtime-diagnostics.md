# Runtime Diagnostics for .NET Concurrency

## Safety

- Counters and `dotnet-stack` are low impact.
- **A dump pauses the process** for seconds to minutes (proportional to heap size) and can contain secrets and customer data in memory. In production, get the user's approval first, prefer one instance out of rotation, and store dumps somewhere access-controlled. Delete them when done.

## Telemetry clue before touching the process

Compare **request duration** with the sum of its **dependency durations** (Application Insights `requests` vs `dependencies`, or X-Ray / OpenTelemetry traces). If requests take seconds while their dependencies take milliseconds, the time is spent waiting for a thread to run the continuation: a strong sign of thread pool starvation.

```kql
requests
| join kind=leftouter (dependencies | summarize depMs=sum(duration) by operation_Id) on operation_Id
| summarize avgReqMs=avg(duration), avgDepMs=avg(depMs) by bin(timestamp, 5m), cloud_RoleInstance
| extend unexplainedMs = avgReqMs - avgDepMs
| order by timestamp asc
```

## Tools

```bash
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-stack
```

No SDK in the container? Download single-file builds instead, e.g. `https://aka.ms/dotnet-dump/linux-x64` and `https://aka.ms/dotnet-counters/linux-x64`, then `chmod +x`.

## 1. Live counters

```bash
dotnet-counters ps
dotnet-counters monitor -p <pid> --counters System.Runtime
```

| Counter (.NET 8 and earlier) | .NET 9+ name | Starvation / contention signal |
|---|---|---|
| `threadpool-queue-length` | `dotnet.thread_pool.queue.length` | Sustained > 0 and growing |
| `threadpool-thread-count` | `dotnet.thread_pool.thread.count` | Climbing steadily, well above core count |
| `monitor-lock-contention-count` | `dotnet.monitor.lock_contentions` | High and rising: hot lock |
| `time-in-gc` | `dotnet.gc.pause.time` | High: memory pressure, not threading |
| `cpu-usage` | `dotnet.process.cpu.time` | Low CPU + long queue = blocking; high CPU + contention = hot lock |

## 2. Quick stacks without a dump

```bash
dotnet-stack report -p <pid>
```

## 3. Dumps (two, 30 to 60 seconds apart)

```bash
dotnet-dump collect -p <pid> -o /tmp/dump1.dmp
sleep 45
dotnet-dump collect -p <pid> -o /tmp/dump2.dmp
```

Crash dumps automatically on unhandled exceptions (set on the app, then restart once):

```bash
DOTNET_DbgEnableMiniDump=1
DOTNET_DbgMiniDumpType=4          # 4 = full
DOTNET_DbgMiniDumpName=/tmp/crash-%p.dmp
```

## 4. Analyze

```bash
dotnet-dump analyze /tmp/dump1.dmp
```

| Command | What it answers |
|---|---|
| `threadpool` | Worker counts, queued work items, whether the pool is saturated |
| `parallelstacks` (`pstacks`) | Threads grouped by identical stack: one big group blocked in `Wait`/`.Result` is the smoking gun |
| `syncblk` | Which thread owns which `lock`, and how many are waiting: deadlocks and hot locks |
| `dumpasync` | Async state machines and what each is awaiting: stuck awaits that have no thread |
| `clrthreads` | All managed threads and their state |
| `clrstack -all` | Full managed stacks (verbose; use after `pstacks` narrows it down) |

## Getting to the process by host

| Host | How to run the tools |
|---|---|
| **Azure App Service** | Portal: Diagnose and solve problems > Collect a memory dump. Or `az webapp ssh -n <app> -g <rg>` (Linux) and run the single-file tools |
| **AKS / EKS** | `kubectl exec -it <pod> -n <ns> -- /bin/sh`, or `kubectl debug -it <pod> --image=mcr.microsoft.com/dotnet/sdk:8.0 --target=<container>` for a tools container that shares the process namespace. Copy out with `kubectl cp <ns>/<pod>:/tmp/dump1.dmp ./dump1.dmp` |
| **AWS ECS** | `aws ecs execute-command --cluster <c> --task <task> --container <name> --interactive --command "/bin/sh"` (needs `enableExecuteCommand`). Copy dumps out via S3 or a mounted volume |
| **VMs / EC2** | SSH or RDP; Windows can also use `procdump -ma <pid>` |
| **Azure Functions (Consumption) / AWS Lambda** | Dumps are impractical. Rely on telemetry, the code-pattern hunt, and a local repro test |

Notes: the tools must run as the same user as the target process and share its `/tmp` (the diagnostics IPC socket lives there).
