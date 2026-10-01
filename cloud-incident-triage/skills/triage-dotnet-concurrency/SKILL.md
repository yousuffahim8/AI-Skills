---
name: triage-dotnet-concurrency
description: 'Use when a .NET service (ASP.NET Core, Azure Functions, AWS Lambda, worker/queue consumer, WebJob) shows multi-threading symptoms: hangs, deadlocks, thread pool starvation, requests that queue then time out under load, intermittent wrong or mixed-up data between concurrent requests or tenants, "A second operation was started on this context instance", "Collection was modified", InvalidOperationException only under load, or bugs that never reproduce locally.'
argument-hint: 'Symptom, exception text, or "triage <service>"'
---

# Triage .NET Concurrency Issues

Find and prove multi-threading defects in .NET services. Concurrency bugs are **probabilistic**: they need real parallelism to appear, so they hide in local runs and low-traffic test environments, then surface in production under load.

**Core principle:** a concurrency root cause is not confirmed until a deterministic test reproduces it. Logs show the symptom; a test that forces the interleaving proves the mechanism.

This skill complements `investigate-incident`. Use that skill's Safety Rules and report format.

## Step 1: Classify the symptom

| Symptom | Likely class | Go to |
|---|---|---|
| Latency climbs under load, CPU low, thread count keeps growing, eventually timeouts | **Thread pool starvation** (sync-over-async, blocking I/O) | Step 2A |
| Process stops responding entirely, CPU near zero, never recovers | **Deadlock** (lock ordering, sync-over-async with a SynchronizationContext) | Step 2A |
| High CPU, contention counters high, throughput flat | **Lock contention / hot lock** | Step 2A |
| Wrong data, one tenant/user sees another's values, "not found" for rows that exist, FK/PK violations with IDs that do not belong | **Shared mutable state / context bleed** | Step 2B |
| `InvalidOperationException: A second operation was started on this context instance` | **DbContext shared across threads** | Step 2B |
| `Collection was modified`, `IndexOutOfRangeException` or corrupted `Dictionary` (sometimes 100% CPU in `FindEntry`) | **Non-thread-safe collection in shared state** | Step 2B |
| Duplicates, lost updates, 409/412 conflicts, double-processed messages | **Race on check-then-act** (TOCTOU, missing optimistic concurrency) | Step 2B |
| Unhandled exception crashes the process with no useful request context | **`async void`** or fire-and-forget task | Step 2B |
| Outbound HTTP throughput capped no matter how many workers | **Connection limit** (.NET Framework `DefaultConnectionLimit = 2`), socket exhaustion | Step 2B |
| SQL error 1205 (deadlock victim) or 1222 (lock request timeout) | **Database-level concurrency**, often jobs colliding with live traffic | Step 2B, then DB blocking analysis |

## Step 2A: Capture runtime evidence (hangs, starvation, contention)

Capture **before** restarting. A restart destroys the evidence. Commands and per-host instructions (App Service, AKS/EKS, ECS, VMs) are in [runtime-diagnostics.md](./references/runtime-diagnostics.md).

1. **Live counters** (`dotnet-counters`): `threadpool-queue-length`, `threadpool-thread-count`, `monitor-lock-contention-count`, `time-in-gc`, `cpu-usage`
2. **Two dumps 30 to 60 seconds apart** (`dotnet-dump collect`). Comparing them separates "stuck" from "slow".
3. **Analyze** with `dotnet-dump analyze`: `threadpool`, `parallelstacks`, `syncblk`, `dumpasync`, `clrstack -all`

Reading the result:
- Many threads parked in `Task.Wait`, `.Result`, `GetAwaiter().GetResult()`, or `ManualResetEventSlim.Wait` under request handlers: **sync-over-async starvation**
- Thread count rising roughly one per second or slower with a growing queue: the pool's slow injection rate is the bottleneck, confirming starvation
- `syncblk` shows a lock owned by thread A while A waits on a lock owned by B: **deadlock**, with the exact lock objects
- Identical stacks in both dumps: stuck. Different stacks: slow but progressing.

## Step 2B: Hunt the pattern in code (wrong data, races, exceptions)

1. Get the exception, stack trace, and the **concurrency signature** from telemetry: do failures span many distinct tenants/users across multiple instances at the same time? That is the condition shared-state bugs need.
2. Search the code for the anti-patterns in [code-patterns.md](./references/code-patterns.md). Each entry has a grep pattern, the broken shape, and the fix.
3. Prioritize **anything that changed recently** in: ambient context (`AsyncLocal`, `HttpContext` access, `CallContext` replacements), DI lifetimes, static fields, caching, and framework/runtime upgrades. Upgrades that replace a context-storage mechanism are a classic source of new bleeds.
4. Trace the shared object: who writes it, who reads it, and whether two concurrent flows can hold the **same reference**.

## Step 3: Prove it with a deterministic test

Write a unit test that forces the interleaving, using the recipe in [repro-test.md](./references/repro-test.md): fan out N concurrent flows, synchronize them at the critical point with a barrier, and assert each flow sees only its own data.

- **Test fails on current code** = mechanism proven
- **Same test passes with the fix applied** = fix validated
- Then run the full suite to check for collateral damage

If you cannot write a failing test, say so in the report and label the root cause **Hypothesis**.

## Step 4: Confirm in the environment (when the test alone is not enough)

Add temporary logging at the failure site that prints the **ambient** value next to the **entity's own** value (for example, ambient tenant ID vs the record's tenant ID). A mismatch is a direct observation of the bleed.

When reproducing in a test environment, expect a low hit rate. Raise parallelism, use many distinct tenants, run longer, and pin to a single instance before concluding "does not reproduce".

## Step 5: Report

Use the `investigate-incident` report structure. Add these lines under **Findings**:

- **Concurrency class:** {from Step 1}
- **Shared object:** {type, field, lifetime, file:line}
- **Interleaving:** {flow A does X, flow B does Y, result Z}
- **Proof:** {test name and result, before and after fix}
- **Silent impact:** {paths that fail open with no exception, e.g. queries scoped only by the corrupted value. These need a data audit, not just log monitoring.}

## Common Mistakes

| Mistake | Fix |
|---|---|
| Restarting before capturing dumps | Capture two dumps first; restart is the mitigation, not the diagnosis |
| "Can't reproduce locally, so it's not our code" | Local runs lack parallelism. Force it with the repro test |
| Raising `ThreadPool.SetMinThreads` as the fix | That buys time. Remove the blocking call |
| Wrapping everything in `lock` | Find the shared object and remove the sharing (copy, scope, or immutable) |
| Trusting error logs to measure impact | Bleeds can return wrong data silently. Audit data written during the window |
| Declaring a fix done because errors stopped for an hour | Load-dependent bugs need a long, high-traffic window to confirm |
