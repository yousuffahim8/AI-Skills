# Deterministic Repro Test Recipe

Concurrency bugs reproduce randomly under load. A good repro test removes the randomness: it **forces** the bad interleaving every run.

## The recipe

1. **Establish shared state in the parent**, the way the real host does (middleware, a queue listener, a base class). Child flows inherit it.
2. **Fan out N flows** on dedicated threads (`TaskCreationOptions.LongRunning`), so a blocking barrier cannot starve the thread pool.
3. **Each flow writes its own identity** (tenant ID, user, correlation ID).
4. **Barrier:** every flow waits until all writes are done. This guarantees the overlap that production only hits sometimes.
5. **Each flow reads back** and records what it saw.
6. **Assert each flow saw only its own value.**

Run the test against current code (must fail), then against the fix (must pass), then run the full suite.

## Worked example: `AsyncLocal` context bleed (xUnit, .NET 6+)

The broken store mutates a dictionary shared by reference. The fixed store uses copy-on-write. Verified: the broken test fails on every run with all flows reporting the same tenant; the fixed test passes.

```csharp
public static class BrokenAmbientContext
{
    private static readonly AsyncLocal<Dictionary<string, object>> Data = new();

    public static void Set(string key, object value)
    {
        Data.Value ??= new Dictionary<string, object>();
        Data.Value[key] = value;                       // mutates the shared instance
    }

    public static object? Get(string key) =>
        Data.Value is { } d && d.TryGetValue(key, out var v) ? v : null;
}

public static class FixedAmbientContext
{
    private static readonly AsyncLocal<Dictionary<string, object>> Data = new();

    public static void Set(string key, object value)
    {
        var copy = Data.Value is null
            ? new Dictionary<string, object>()
            : new Dictionary<string, object>(Data.Value);
        copy[key] = value;
        Data.Value = copy;                             // replaces, so the write stays in this flow
    }

    public static object? Get(string key) =>
        Data.Value is { } d && d.TryGetValue(key, out var v) ? v : null;
}

public class AmbientContextTests
{
    private const int Flows = 8;

    private static async Task<int[]> RunConcurrentFlows(Action<string, object> set, Func<string, object?> get)
    {
        set("RequestStarted", true);                   // 1. parent creates the shared state

        using var barrier = new Barrier(Flows);
        var seen = new int[Flows];

        var tasks = Enumerable.Range(0, Flows).Select(i => Task.Factory.StartNew(() =>
        {
            set("TenantId", i);                        // 3. each flow writes its own identity
            barrier.SignalAndWait();                   // 4. all writes finish before any read
            seen[i] = (int)get("TenantId")!;           // 5. read back
        }, TaskCreationOptions.LongRunning));          // 2. dedicated threads

        await Task.WhenAll(tasks);
        return seen;
    }

    [Fact]
    public async Task Broken_BleedsAcrossConcurrentFlows()
    {
        var seen = await RunConcurrentFlows(BrokenAmbientContext.Set, BrokenAmbientContext.Get);
        Assert.Equal(Enumerable.Range(0, Flows), seen);   // FAILS: e.g. [7, 7, 7, 7, 7, 7, 7, 7]
    }

    [Fact]
    public async Task Fixed_IsIsolatedAcrossConcurrentFlows()
    {
        var seen = await RunConcurrentFlows(FixedAmbientContext.Set, FixedAmbientContext.Get);
        Assert.Equal(Enumerable.Range(0, Flows), seen);   // PASSES
    }
}
```

In a real investigation, replace the two toy classes with the **production** context class and call it exactly as the production entry point does.

## Adapting the recipe to other classes

| Class | What each flow does before the barrier | What to assert |
|---|---|---|
| Captive scoped dependency | Resolve the service from a new scope and set a per-request value | Each flow reads its own value |
| Shared `Dictionary` / `List` | Write many keys in a tight loop (no barrier needed; use 10k iterations) | No exception, final count is exact |
| Check-then-act | Run the "if not exists, insert" path for the **same** key | Exactly one insert succeeds |
| Shared `DbContext` | Start a query on the shared context | Expect `InvalidOperationException`; with the fix, all succeed |
| Thread pool starvation | Call the sync-over-async path from 50+ concurrent `Task.Run` | Completes within a time limit (use `Task.WhenAny` with `Task.Delay` as a timeout) |

## Gotchas

- **Sync vs async writes matter for `AsyncLocal`.** A write made synchronously, before any `await`, mutates the caller's own context and stays visible to it. Only an async fork (`Task.Run`, an `async` method that has awaited) creates isolation. Reproduce with the same sync/async shape as the real code.
- **Never use `Thread.Sleep` to "line up" threads.** It is timing-dependent and flaky. Use `Barrier`, `CountdownEvent`, or `TaskCompletionSource`.
- **Keep the repro test in the suite** after the fix, as a regression guard.
