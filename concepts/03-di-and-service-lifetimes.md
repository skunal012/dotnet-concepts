# DI and service lifetimes

> **The one idea:** you register a *recipe* and a *lifetime*, and the container builds the whole object graph on demand — the lifetime decides which cache the instance is stored in and when it is disposed. Almost every hard DI bug is a lifetime mismatch.

---

## Why this concept exists

`new` is a decision made at the wrong time. The moment a class writes `new SqlRackRepository()`, it has chosen a concrete type, a construction order, and a disposal story — at compile time, on behalf of every caller that will ever exist. There is no seam to substitute anything in a test, because nothing was ever asked for.

Old ASP.NET had no answer to this. Controllers needed parameterless constructors, so dependencies were either `new`ed inline or pulled from ambient statics — `HttpContext.Current`, a static `ServiceLocator`, a singleton property. Third-party containers (Ninject, Autofac, StructureMap) got bolted on, each with its own conventions, and the framework's own services stayed outside them.

.NET Core made the container part of the host. That is a bigger change than "DI is now available", and it is the part worth holding onto:

**The framework is built out of the same container you register into.** `ILogger<T>`, `IOptions<T>`, `IHttpClientFactory`, `IHostApplicationLifetime`, your `DbContext` — all of them are entries in the same list. This is why constructor injection just works in a controller you never wired up, and why `HttpContext.Current` no longer exists. Ambient statics are precisely what the container replaced.

This is the third ordered-list concept in a row. Chapter 01's list decided who sees the request first; chapter 02's decided whose value wins; this one decides which implementation you get — and adds a second axis the other two did not have: **how long it lives**.

---

## 1. The mechanism: a list of recipes, resolved recursively

`IServiceCollection` is a `List<ServiceDescriptor>`. A descriptor is three things: the service type you ask for, how to make it, and the lifetime.

```csharp
builder.Services.AddScoped<IRackOptimizer, RackOptimizer>();
// ≈ new ServiceDescriptor(typeof(IRackOptimizer), typeof(RackOptimizer), ServiceLifetime.Scoped)
```

`builder.Build()` freezes that list into an `IServiceProvider`. Resolution is then a graph walk, and it is not clever: find the descriptor for `T`, look at the implementation's constructor, resolve each parameter the same way, recurse to the leaves, construct back up.

That is the entire mechanism. "Inversion of control" only means you stopped calling constructors, so something else has to — and the something else reads a list.

Two consequences fall straight out, no memorising required:

- **The container picks the constructor whose parameters it can all satisfy.** So a second "convenience" constructor is not free — it is a resolution hazard, and an ambiguous pair throws at resolve time.
- **Last registration wins.** `GetService<T>()` returns the *last* matching descriptor, because it is a list and lookup takes the last entry — the same rule as chapter 02's provider stack, for the same reason. Register an interface twice and inject `IEnumerable<T>` to get **all** of them, in registration order. This is why libraries register their defaults with `TryAddSingleton` rather than `AddSingleton`: `TryAdd*` is "only if nothing is registered yet", which lets your override survive.

Four registration forms, each answering a different question:

```csharp
services.AddSingleton<IClock, SystemClock>();               // type: the container constructs it
services.AddSingleton<IClock>(sp => new SystemClock(tz));   // factory: construction needs runtime data
services.AddSingleton<IClock>(new SystemClock(tz));         // instance: you built it, you own it (§4)
services.AddKeyedSingleton<IPackStrategy, FirstFit>("ff");  // .NET 8: one interface, several names
```

---

## 2. Lifetime is a question about *which cache*

The usual framing — "how long does the object live" — is vague enough to be useless at the point where it matters. The precise question is: **which container holds the cached instance?**

| Lifetime | Cached in | New instance when |
|---|---|---|
| Transient | nowhere | every single resolve |
| Scoped | the scope it was resolved from | once per scope |
| Singleton | the root provider | once per process |

ASP.NET Core opens a scope when a request arrives and disposes it when the response completes. That is a **convention of the hosting layer**, not part of the definition. So say "one per scope" and never "one per request" — the day you write a background worker there is no request, and the difference stops being pedantry. §7.

Choosing then becomes mechanical rather than a matter of taste:

- Holds per-operation state — a change tracker, the current user, a transaction? **Scoped.**
- Stateless and cheap to build? **Transient.** The default for your own services.
- Holds process-wide state or an expensive resource — a cache, a connection pool, a compiled model? **Singleton**, and now thread safety is your problem (§5).

---

## 3. The captive dependency — the rule that generates the rest

> A service may only depend on something that lives **at least as long** as it does.

The reason is in §1: dependencies are resolved **once, at construction**. A singleton is constructed once, so whatever it received, it holds for the life of the process. Injecting a shorter-lived service into a longer-lived one does not shorten the consumer — it *promotes the dependency*, silently.

Singleton depending on scoped `DbContext`, concretely:

1. **It becomes shared across requests.** `DbContext` is not thread-safe, so two concurrent requests produce `A second operation was started on this context before a previous operation completed`.
2. **It is disposed while still referenced.** The first scope ends, disposal runs, and the singleton keeps using a dead object — `ObjectDisposedException`, at whatever time the second request happens to arrive.

The container does catch this — **in Development only**. `ValidateScopes` and `ValidateOnBuild` are enabled by default when the environment is Development, which produces `Cannot consume scoped service 'X' from singleton 'Y'` at startup. Run locally with `ASPNETCORE_ENVIRONMENT=Production`, or turn validation off because it is "noisy", and the identical code does the wrong thing quietly. That is the whole argument for leaving it on.

You have now met this bug three times in three chapters:

| Where | What it looked like |
|---|---|
| Ch 01 §7 | `DbContext` constructor-injected into middleware — middleware is instantiated once |
| Ch 02 §6 | `IOptionsSnapshot<T>` injected into a singleton — scoped, so it never updates |
| Here | Anything scoped injected into a singleton |

One rule, three costumes. The fix is never "make the dependency a singleton too" — that just moves the thread-safety problem into a class that was written assuming it had one. The fix is to **resolve it later, from a scope you open yourself**:

```csharp
public sealed class OptimizationQueueWorker(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in _queue.ReadAllAsync(ct))
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<GrosDbContext>();
            await Run(job, db, ct);
        }   // scope disposed → DbContext disposed → change tracker gone
    }
}
```

One job, one scope, one change tracker. And note *why* the constructor could not have taken the `DbContext` directly: `AddHostedService<T>` registers `T` as a **singleton**. The worker is long-lived by construction, so it must open scopes rather than receive them.

---

## 4. Disposal follows ownership

The container disposes what **it** created, at the end of the scope that created it. Two rules follow, and both are quietly load-bearing:

**Register an instance and you keep the disposal.** `AddSingleton(new SomeClient())` hands the container an object it did not construct, so it will not dispose it either. If that client holds a socket, closing it is your job. Prefer the factory overload — `AddSingleton<IClient>(sp => new SomeClient(...))` — where the container constructs and therefore owns.

**Transient does not mean untracked.** A transient `IDisposable` is still registered for disposal with the scope that resolved it, so it survives until that scope ends. Two ways that turns into a leak:

- Resolved from the **root** provider, or injected into a singleton: the root provider is the scope, and it ends at process shutdown. The instances accumulate for the entire uptime of the app. Memory grows steadily under a flat request rate — the signature symptom.
- Resolved many times inside one long-running scope: same accumulation, bounded by the scope instead of the process.

So: a disposable created many times per operation should not be a container-managed transient. Give it a factory and a `using` block. `IDbContextFactory<T>` exists for exactly this reason, and chapter 04 comes back to it.

---

## 5. Singleton means shared, which means concurrent

A singleton is touched by every request simultaneously. Every mutable field in it is shared mutable state across threads — that is not a risk to weigh, it is the definition of what you registered.

**In GROS:** the optimizer keeps a metric cache — precomputed rack and container metrics that are identical for every run and expensive to build — registered as a singleton. That is the correct lifetime: recomputing per request would be pure waste. And it is a `ConcurrentDictionary` for exactly one reason: **the registration line made it shared**, so a plain `Dictionary` would corrupt under concurrent writes, and locking the whole thing would serialise the requests I made it a cache to speed up.

The data structure is a *consequence of the lifetime*, not an independent decision. That is the test worth carrying into an interview: if you register something as a singleton, every field it owns must be immutable or thread-safe, and you should be able to say which.

---

## 6. Where the "design patterns" questions actually go

Most pattern questions on a .NET list are DI questions in a costume. The honest answers:

- **Factory** — the container is one. Write your own only when the choice depends on runtime data: inject `Func<T>`, a keyed lookup, or a small factory interface. A factory that just calls `new` on a fixed type is a registration you have not made yet.
- **Strategy** — inject `IEnumerable<IStrategy>` and select, or use keyed services when the selector is a name. In the optimizer, "which packing strategy" is a runtime input, so the strategies are registered together and picked per job.
- **Singleton (the GoF one, a static `Instance`)** — replaced by `AddSingleton`, and the replacement is strictly better: substitutable in tests, disposed by the host, and *visible in the constructor*. A static instance is a dependency you cannot see, which is the thing this whole chapter exists to eliminate.
- **Decorator** — register the inner implementation, then register the interface with a factory that resolves the inner and wraps it. Retry, caching and logging layers are decorators; this is how you add one without touching the class it wraps.
- **Repository + Unit of Work** — `DbContext` is already both. `DbSet<T>` is the repository, `SaveChangesAsync()` is the unit-of-work commit, and its **scoped lifetime** is what makes "one unit of work per request" true rather than aspirational. So a repository on top of EF Core buys you three specific things — a narrower surface than the full `DbSet`, a seam for tests, and somewhere to put query logic that would otherwise be duplicated. It does **not** buy you independence from the database: `IQueryable<T>` leaks provider behaviour straight through the interface. If the methods only forward to `DbSet`, delete the layer. Chapter 04 goes further.
- **CQRS** — separating the write model from the read model because they genuinely have different shapes, and paying for it with two models and eventual consistency between them. A dispatcher that resolves `IRequestHandler<TCommand, TResult>` from the container is one implementation detail of one half. Worth being blunt about in an interview: MediatR is not CQRS. It is a mediator, and a mediator over a single database is CQRS in the same sense that a folder named `Services` is architecture.
- **Domain-driven design** — DI is the mechanism that makes dependency inversion practical at a boundary: domain code declares the interfaces it needs, infrastructure implements them, and the composition root is the only place that knows both. Aggregate roots decide transaction boundaries; in .NET that boundary lands on the scoped `DbContext` and one `SaveChangesAsync`.
- **SOLID** — **D** is the container itself, literally. **I** — small interfaces are cheap to register and cheap to fake. **O** — you extend by registering a different implementation instead of editing a class. **S** and **L** are still yours; no container will save you from a 900-line service.

---

## 7. The detail most people miss: a scope is a unit of work, not a request

Everyone learns "scoped = per request". It is true in exactly one context — an HTTP request handled by ASP.NET Core — and it fails in the two places where the failures are expensive.

**No request at all.** Background services, queue consumers, `IHostedService`, timers. There is no ambient scope, so a scoped service resolved from the root provider is created once and lives forever, which is the captive dependency of §3 arriving through the back door. One scope per unit of work, opened by you (§3).

**More than one thread inside one request.** This is the sharper one. `Task.WhenAll` over work that shares the request's scoped `DbContext` produces `A second operation was started on this context before a previous operation completed`. Scoped means *one instance per scope* — it says nothing about threads. The instant you fan out, "scoped" stops meaning "safe" and starts meaning "shared by everything in this scope, including the eight tasks you just started".

**In GROS:** an optimization run fans out across candidate placements, and that is precisely where the request-scoped mental model breaks. Parallel branches need their own context — a scope per branch, or `IDbContextFactory<T>` — while genuinely shared, read-only state (the metric cache) stays a singleton. Those two decisions look unrelated until you see them as the same question: *what is the unit of work, and what is shared across all of them?*

A third one, shorter, that belongs in the same family: injecting `IServiceProvider` and resolving from it inside a domain service is the service locator pattern coming back in through the container. It hides dependencies from the constructor, defeats scope validation, and makes the class untestable without a container. Acceptable in a composition-root factory; a smell anywhere else.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| `Cannot consume scoped service 'X' from singleton 'Y'` at startup | Captive dependency. Inject `IServiceScopeFactory` — do not "fix" it by widening the dependency's lifetime |
| Startup error in Development, silent misbehaviour in Production | Scope validation is Development-only by default |
| `A second operation was started on this context` | One `DbContext` shared across parallel tasks, or held by a singleton |
| `ObjectDisposedException` on a service that worked yesterday | A longer-lived object is holding something disposed at the end of an earlier scope |
| `Unable to resolve service for type 'X'` on first request, not at startup | Never registered. Resolution is lazy, so wiring mistakes surface on use unless `ValidateOnBuild` catches them |
| Memory grows steadily under constant load | Transient `IDisposable` resolved from the root provider or from a singleton — tracked until shutdown |
| The wrong implementation is injected | Registered twice; the last descriptor wins. Use `TryAdd*` for defaults |
| Corrupt or racy data in a cache under load | A singleton with a non-thread-safe field |
| Configuration changes never reach a service | `IOptions`/`IOptionsSnapshot` in a singleton — ch. 02 §6, same rule |
| A dependency is never injected / the wrong constructor runs | Multiple constructors; the container chose the one it could satisfy |
| An object registered with `AddSingleton(new X())` never releases its handle | The container did not create it, so it does not dispose it |

---

## Check yourself

Answer out loud, without looking:

1. Why is "scoped means one per request" a description rather than a definition, and name the two situations where the difference costs you.
2. A singleton needs a `DbContext`. Making the `DbContext` a singleton removes the startup error — name the two separate things that are now broken.
3. You register `IPackStrategy` four times. What does `GetService<IPackStrategy>()` return, and how do you get all four?
4. Why can a *transient* leak, and what is special about resolving one from the root provider?
5. A singleton cache uses `ConcurrentDictionary`. Why is that a consequence of the registration rather than a performance choice?

---

## Questions this chapter answers

Seven of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch03):

| # | As originally asked | Answered by section |
|---|---|---|
| 21 | What is Dependency Injection in .NET Core? | §1 — a list of recipes, resolved recursively; the framework uses the same list |
| 22 | How do you implement custom services and use DI? | §1–2 — the four registration forms, and choosing a lifetime mechanically |
| 61 | Popular design patterns in .NET Core | §6 — most of them are supplied by the container or dissolved by it |
| 62 | Repository and Unit of Work patterns | §6 — `DbContext` is already both; what a repository on top actually buys |
| 63 | Explain CQRS | §6 — two models and their cost, versus a dispatcher resolving handlers |
| 64 | Importance of domain-driven design | §6 — DI as the mechanism of dependency inversion at a boundary |
| 65 | How does .NET Core support SOLID? | §6 — D *is* the container; the others follow from interface registration |

Chapter 04 picks up the scoped service you will actually spend your life with: 28, 29, 56–60, 72.

## Next

→ `04-ef-cores-mental-model.md` — the `DbContext` has been the example in every section here. Next it becomes the subject: why it is scoped, what the change tracker is doing, and why `SaveChanges` is the unit-of-work commit this chapter kept referring to.
