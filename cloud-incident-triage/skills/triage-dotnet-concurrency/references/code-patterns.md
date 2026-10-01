# .NET Concurrency Anti-Patterns

Each entry: what to grep for (ripgrep syntax, `--type cs`), the broken shape, and the fix. A grep hit is a lead, not a verdict. Confirm the object is actually shared between concurrent flows.

---

## 1. Mutable reference stored in `AsyncLocal<T>` (context bleed)

**Grep:** `AsyncLocal<`

`AsyncLocal` flows the **reference** to child async flows. If the value is a mutable object (a `Dictionary`, a context class with setters), mutating it in place is visible to every sibling flow that inherited the same reference. Concurrent requests or queue messages overwrite each other's tenant ID, user, correlation ID. Last writer wins, and a value can change **mid-flow** (read #1 as tenant A, read #2 as tenant B).

```csharp
// BROKEN: one shared dictionary, mutated in place
private static readonly AsyncLocal<Dictionary<string, object>> Data = new();
public static void Set(string key, object value)
{
    Data.Value ??= new Dictionary<string, object>();
    Data.Value[key] = value;                      // visible to sibling flows
}

// FIX: copy-on-write, so each flow's writes stay private to it and its children
public static void Set(string key, object value)
{
    var copy = Data.Value is null ? new Dictionary<string, object>() : new Dictionary<string, object>(Data.Value);
    copy[key] = value;
    Data.Value = copy;                            // replace, never mutate
}
```

Better still: store an **immutable** value (`ImmutableDictionary`, a `record`) so mutation is impossible.

Watch for this after framework upgrades that replace `CallContext.LogicalSetData` (copy-on-write semantics) with a hand-rolled `AsyncLocal` store.

## 2. Sync-over-async (thread pool starvation, deadlocks)

**Grep:** `\.Result\b|\.Wait\(\)|GetAwaiter\(\)\.GetResult\(\)|Task\.WaitAll\(`

Each blocked call holds a pool thread while it waits for another pool thread to complete the task. Under load the pool runs out and grows slowly. On .NET Framework ASP.NET, WinForms, or WPF (which have a `SynchronizationContext`), this can deadlock outright.

```csharp
var data = client.GetStringAsync(url).Result;     // BROKEN
var data = await client.GetStringAsync(url);      // FIX: async all the way up
```

If a sync boundary truly cannot be removed (legacy interface), isolate it and add `ConfigureAwait(false)` throughout the async library code it calls.

## 3. `async void` and async lambdas passed to `Action`

**Grep:** `async void|Parallel\.ForEach\(.*async|\.ForEach\(async`

Exceptions from `async void` cannot be caught by the caller and crash the process. `Parallel.ForEach` and `List.ForEach` take `Action`, so an async lambda becomes `async void`: the loop "finishes" instantly while work continues unobserved.

```csharp
Parallel.ForEach(items, async i => await ProcessAsync(i));                  // BROKEN
await Parallel.ForEachAsync(items, new ParallelOptions { MaxDegreeOfParallelism = 8 },
    async (i, ct) => await ProcessAsync(i, ct));                              // FIX (.NET 6+)
```

Only event handlers should be `async void`, and they should catch everything.

## 4. Fire-and-forget tasks

**Grep:** `_ = Task\.Run|_ = \w+Async\(|^\s*Task\.Run\(`

Unobserved tasks lose exceptions and outlive the request scope (disposed `DbContext`, `HttpContext` accessed after the request ended). Use a background queue (`Channel<T>` + `BackgroundService`) or `IHostedService`, and create a fresh DI scope inside it.

## 5. Shared `DbContext` (EF Core)

**Grep:** `AddSingleton<.*DbContext|static .*DbContext|Task\.WhenAll\(` (then check whether the tasks share one context)

`DbContext` is not thread-safe. Symptom: `A second operation was started on this context instance before a previous operation completed`.

```csharp
await Task.WhenAll(ids.Select(id => _db.Orders.FindAsync(id).AsTask()));   // BROKEN: one context, parallel queries

// FIX: one context per concurrent unit of work
await Task.WhenAll(ids.Select(async id =>
{
    await using var db = await _dbFactory.CreateDbContextAsync();
    return await db.Orders.FindAsync(id);
}));
```

## 6. Captive dependency (scoped or transient captured by a singleton)

**Grep:** `AddSingleton<` then inspect constructor parameters of each singleton

A singleton that takes a scoped service in its constructor keeps the **first** request's instance forever and shares it across all requests. Enable scope validation to catch it at startup:

```csharp
builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });
```

Inside singletons and hosted services, resolve scoped services via `IServiceScopeFactory.CreateScope()` per unit of work.

## 7. Non-thread-safe collections in shared state

**Grep:** `static (readonly )?(Dictionary|List|HashSet|Queue)<|private (readonly )?(Dictionary|List|HashSet)<` in classes registered as singletons

Concurrent writes to `Dictionary<TKey,TValue>` can corrupt its internals, sometimes causing an **infinite loop at 100% CPU** in `FindEntry` or `TryInsert`, not just an exception.

**Fix:** `ConcurrentDictionary`, `ImmutableDictionary` with `ImmutableInterlocked`, or a lock around every access (reads too). Note `ConcurrentDictionary.GetOrAdd(key, factory)` may run the factory more than once; wrap the value in `Lazy<T>` if the factory has side effects.

## 8. Static mutable fields

**Grep:** `rg --type cs '\bstatic\b' | rg -v 'readonly|const |class |void |async |Task|\(|=>'`

Any non-readonly static is shared by every thread. Also check `static readonly` fields that hold mutable objects.

## 9. `lock` around `await`, and semaphores not released

**Grep:** `SemaphoreSlim|\block\s*\(`

`await` inside `lock` does not compile, which leads people to `Monitor.Enter`/`Exit` across an await (the continuation may run on another thread: `SynchronizationLockException`). Use `SemaphoreSlim`, always released in `finally`:

```csharp
await _gate.WaitAsync(ct);
try { await DoWorkAsync(ct); }
finally { _gate.Release(); }
```

Lock ordering: if two code paths take locks A and B in opposite orders, they can deadlock. `syncblk` in a dump shows it.

## 10. Check-then-act races (TOCTOU)

**Grep:** `if \(!.*(Exists|Any|Contains)(Async)?\(` followed by an insert/add

Two concurrent flows both see "not exists" and both insert: duplicates or PK violations. **Fix:** enforce at the store (unique constraint + handle the violation, conditional writes, ETags / `If-Match`, rowversion concurrency tokens, DynamoDB condition expressions), not in application memory.

## 11. Outbound connection limits

**Grep:** `ServicePointManager|new HttpClient\(`

- **.NET Framework:** `ServicePointManager.DefaultConnectionLimit` defaults to **2 per host** for non-ASP.NET processes, capping parallel outbound calls no matter how many workers you add. Raise it at startup.
- **All versions:** `new HttpClient()` per call exhausts sockets under load. Use `IHttpClientFactory` or one long-lived `SocketsHttpHandler` with `PooledConnectionLifetime`.

## 12. Timer and scheduled job overlap

**Grep:** `System\.Threading\.Timer|System\.Timers\.Timer|PeriodicTimer|Cron|TimerTrigger`

`System.Threading.Timer` and `System.Timers.Timer` fire again even if the previous callback is still running. Use `PeriodicTimer` in a loop (never overlaps), or a non-reentrant guard. Also check whether several maintenance jobs are scheduled at the **same instant** against overlapping tables: they compete for locks with each other and with live traffic (SQL 1205 deadlocks, 1222 lock timeouts).

## 13. Closures capturing loop variables

**Grep:** `for \(var \w+ = ` then read each loop body for lambdas or `Task.Run`

`foreach` variables are per-iteration since C# 5, but a `for` loop index is shared: every lambda may see the final value. Copy to a local inside the loop.
