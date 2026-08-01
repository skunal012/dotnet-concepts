# Async, threads and memory

> **The one idea:** nearly every .NET performance question is a cost paid at the wrong time — a thread held while nothing is happening, memory held after you are finished with it, or work done at run time that could have been settled at compile time.

▶ **Widget:** [`widgets/05-async-threads-and-memory.html`](../widgets/05-async-threads-and-memory.html) — send load at a fixed thread pool, blocking and then awaiting, and watch the queue. Open it in any browser; no build step.

---

## Why this concept exists

A server's throughput is bounded by a resource nobody names in the requirements: threads. A thread costs about a megabyte of stack plus scheduler attention, so you get thousands, not millions — and the moment one blocks, you have taken a unit of your capacity out of circulation.

The old model was a thread per request, and it was fine while requests were CPU work. It stopped being fine when requests became *waiting*: a database round trip, an HTTP call to another service, a file read. During that wait the thread holds a megabyte of stack and a slot in the pool to accomplish precisely nothing, while the CPU sits idle and the queue behind it grows.

Memory has the same shape. The garbage collector will reclaim anything nothing refers to, so "using less memory" is never about freeing — it is about *stopping referring*. And compilation has the same shape again: reflection resolves a type on every call, forever, to answer a question that was already answerable when you built the thing.

Three different mechanisms, one principle: **stop paying for something during the time you are not using it.** This chapter is that sentence three times.

Chapter 04 has been setting it up. Every database call there was `await`ed, and §4's argument about tracking was a memory argument in a costume — snapshots you keep for a change set you know will be empty.

---

# Movement one — threads you don't block

## 1. `await` gives the thread back; it does not make anything faster

This is the correction that has to land before anything else. Making a method `async` does not speed up a single request. It usually makes it very slightly slower, because there is a state machine where there was a straight line.

What it changes is **occupancy**. The compiler rewrites the method into a state machine; at an `await`, if the awaited operation has not already completed, the method *returns to its caller* and registers a continuation. The thread goes back to the pool and picks up other work. When the I/O completes, the continuation is scheduled onto some pool thread — not necessarily the one you started on — and the method resumes at the line after the `await`.

```csharp
var rack = await db.Racks.FindAsync(id, ct);   // thread released here
return Ok(rack);                               // resumed, possibly on a different thread
```

So the honest one-line answer to *what is the purpose of asynchronous programming* is: **the same number of threads serves far more concurrent requests, because none of them are held during a wait.** One request is not faster. A thousand are, because the thousandth is not stuck in a queue behind nine hundred threads doing nothing.

The corollary matters as much: **async only helps when you are waiting on something external.** Wrapping CPU-bound work in `Task.Run` on a server does not release anything — it moves work from one pool thread to another and adds a hop. There is no I/O to wait for, so there is no thread to give back.

---

## 2. Thread pool starvation is the failure this exists to prevent

The pool starts with a minimum number of threads (historically tied to core count) and **grows slowly** past it — on the order of one or two a second, because thread injection is a hill-climbing heuristic, not an allocation. That rate is the whole story. Block threads faster than the pool can add them and the work queue grows without bound.

The fingerprint is unmistakable once you have seen it: **latency climbing steeply while CPU sits low.** Nothing is working hard; everything is waiting for a thread that is itself waiting.

The way you get there is sync-over-async:

```csharp
var rack = db.Racks.FindAsync(id).Result;              // blocks a pool thread
var rack = db.Racks.FindAsync(id).GetAwaiter().GetResult();   // same thing, longer
someTask.Wait();                                        // same thing again
```

Each of these holds a pool thread for the entire duration of an operation that was specifically designed not to need one. One blocked thread per in-flight request is exactly the model async removed, reintroduced by hand — and now with a state machine on top.

Three details that decide whether you hit this:

- **ASP.NET Core has no `SynchronizationContext`.** In old ASP.NET and WinForms, `.Result` on an async method deadlocked outright, which was unpleasant but at least loud. Here it does not deadlock — it starves. The app degrades under load instead of failing in development, which is strictly harder to catch.
- **`ConfigureAwait(false)` is a no-op in ASP.NET Core**, for the same reason: there is no context to capture. It still belongs in library code, which cannot know whether some caller has a context.
- **`async void` is only for event handlers.** It has no task to observe, so an exception inside it cannot be caught by the caller and takes the process down. If it is not literally an event handler, it should be `async Task`.

And `CancellationToken` belongs on every async method that can be reached from a request. When a client disconnects, the token is signalled, and everything downstream — EF queries, HTTP calls, your own loops — can stop. It is the same principle applied to abandoned work: stop paying for a result nobody is waiting for.

---

## 3. Concurrency is not parallelism, and `Task.WhenAll` is where they meet

Concurrency is *dealing with* many things at once; parallelism is *doing* many things at once. `await` buys you the first. `Task.WhenAll` over genuinely independent I/O buys you overlap:

```csharp
var racks = FetchRacksAsync(ct);
var rules = FetchRulesAsync(ct);
await Task.WhenAll(racks, rules);       // two round trips, one wall-clock wait
```

That is a real win when the two calls are independent and go somewhere else. It is a trap the moment they share state — and you have met this exact bug in the two previous chapters. Fanning out over one scoped `DbContext` gives you `A second operation was started on this context before a previous operation completed`, because scoped means *one instance per scope*, not *one thread at a time*. Third appearance, same rule: a scope is a unit of work, and parallel branches need their own.

For genuinely CPU-bound work, `Parallel.ForEachAsync` or `Task.Run` are the right tools — on a background service or a desktop app. On a request thread they usually just move the cost sideways.

---

# Movement two — memory you don't hold

## 4. A leak is a root you forgot, not memory that failed to free

The collector reclaims every object that is unreachable from a **root** — a static field, a local on some thread's stack, a CPU register, or a GC handle. Reachability is the entire model. Generations are an optimisation on top of it: most objects die young, so gen0 is collected often and cheaply, survivors are promoted to gen1 and then gen2, and gen2 collections are rare and expensive. Objects over about 85 KB go straight to the large object heap, which is not compacted by default.

Once that is the model, "how do you find a memory leak in .NET" stops being a technique question. **Managed memory is never leaked in the C sense — it is retained.** So there is always a reference, and the job is to find who is holding it. The usual suspects, in the order they actually occur:

- **Static collections.** A `static Dictionary` that only ever gets added to is a root that grows forever.
- **Event handlers.** A long-lived publisher holds a reference to every subscriber that never unsubscribed. The subscriber cannot be collected even after everything else has forgotten it.
- **Caches with no eviction policy** — §5.
- **Captured closures on long-lived delegates.** The lambda holds its display class, which holds everything it captured, for as long as something holds the delegate.
- **Undisposed timers, subscriptions and `CancellationTokenRegistration`s.**
- **A singleton holding something it should have resolved per-operation** — chapter 03 §4, arriving here as a memory symptom rather than a lifetime error.

The diagnosis path is mechanical: watch **gen2 heap size across collections**. If it climbs and never comes back down, something is being promoted and retained. `dotnet-counters` shows you that in production; `dotnet-gcdump` or `dotnet-dump` gives you the heap, and the question you ask of it is always the same — *what is the path from a root to this object?*

**Server GC is the default for ASP.NET Core**, and it trades memory for throughput: a heap and a background collector thread per core, less frequent pauses, a higher memory floor. That default is correct on a machine you own and often wrong in a small container — §10.

---

## 5. A cache is deliberate leaking, so eviction *is* the design

This is why "how do you cache data" belongs in a memory chapter and not a data-access one. A cache is memory you have decided not to release yet. The storage is trivial; the only interesting question is **when do you let go**, and a cache with no answer to that is a leak with better public relations.

`IMemoryCache` is in-process: fast, no serialisation, and **per instance**. Behind a load balancer with three replicas you have three caches with three different answers, and a user's next request may not see what the last one wrote. That is fine for reference data that changes daily and unacceptable for anything a user just edited.

`IDistributedCache` (Redis, SQL Server) gives one answer for all instances, and charges a network hop plus serialisation for it. The trade is coherence for latency, and the right choice depends entirely on whether staleness is visible to a user.

Response and output caching sit further out still — chapter 01's pipeline, short-circuiting before your endpoint runs at all. The cheapest request is the one that never reaches your code.

Whatever the tier, three decisions are the actual design:

- **Expiration** — absolute (this is stale after N minutes, full stop) versus sliding (stale N minutes after last use). Sliding on a popular key never expires, which is occasionally what you want and usually a surprise.
- **A size limit.** `MemoryCacheOptions.SizeLimit` with a size per entry, or you have built an unbounded dictionary with an eviction API you never call.
- **Its lifetime**, which is a chapter 03 question. A cache is a singleton, singletons are shared across threads, and every field it owns must therefore be immutable or thread-safe.

**In GROS:** the optimizer's metric cache is exactly this shape — precomputed rack and container metrics, identical for every run and expensive to build, held as a singleton `ConcurrentDictionary`. It is bounded by the data set rather than by requests, which is why it can afford to have no expiry at all. That is the honest version of "when is a cache without eviction fine": when the key space is closed and you know its size.

---

## 6. Allocation is a cost you can often just decline

Every allocation is future GC work. The fastest collection is the one with nothing to trace, so the highest-leverage optimisation is usually not allocating in the first place.

- **`Span<T>` and `Memory<T>`** — a view over memory you already have. Slicing a string or a buffer without copying turns a parse loop that allocated per token into one that allocates nothing.
- **`ArrayPool<T>.Shared`** — rent and return large buffers instead of allocating them per operation, which keeps them off the large object heap.
- **`struct` over `class`** for small, short-lived values — stack or inline in the containing object, no collection needed. `readonly struct` avoids defensive copies; `ref struct` guarantees it never escapes to the heap.
- **`ValueTask`** for hot paths that usually complete synchronously — a cache hit that returns `Task.FromResult` allocates a task object per call for no reason. The caveat is real: a `ValueTask` may only be awaited once.
- **`StringBuilder`** for concatenation in loops, because every `+=` on a string allocates a new one.

**Local functions** belong here, and this is the whole reason they are a memory question. A lambda that captures a variable allocates a *display class* on the heap to hold it, plus a delegate. A local function that is only ever called directly — never converted to a delegate, never passed anywhere — needs neither: the compiler can put its captures in a struct passed by reference, and allocate nothing at all. Convert that same local function to a delegate and you are back to a lambda's cost. That is the practical difference, and it is why a local function is the better default for a private helper that closes over the enclosing method's state.

Further out, and answering *what features improve performance* properly: **tiered JIT** compiles methods quickly first and re-compiles hot ones with full optimisation, so startup and steady state are no longer a single trade-off. **ReadyToRun** ships a pre-JIT'd image to cut startup. **Native AOT** removes the JIT and most of the startup cost entirely, at the price of no runtime code generation and a hard trimming constraint — which leads directly into the next movement.

---

# Movement three — work moved to compile time

## 7. Reflection is a bill you pay per call; a source generator pays it once

Reflection answers questions about types at run time: what properties does this have, what constructor should I call, what attribute is on this member. It is enormously useful and it costs on every invocation — lookups, allocations, and no possibility of inlining. Worse for modern deployment, it is *invisible to the linker*: a trimmer or an AOT compiler cannot see that `typeof(T).GetProperty(name)` needs that property kept, so trimming either breaks the app or has to be disabled.

**Source generators** move the same work into the build. A generator runs as part of compilation, inspects the syntax and semantic model, and adds new C# to the compilation. It can only add — it never rewrites your code — which is what keeps the model comprehensible.

What that buys, concretely: `System.Text.Json` serialisers that emit a typed reader and writer instead of walking properties reflectively, `LoggerMessage` that emits the formatting code, `[GeneratedRegex]` that emits a matcher instead of interpreting a pattern, gRPC service and client stubs generated from `.proto` files. Startup reflection disappears, per-call cost disappears, the linker can see everything, and the generated code is real C# you can step into.

**Dynamic compilation is the same trade in reverse** — Roslyn scripting, `Expression.Compile()`, `DynamicMethod`, `dynamic`. You generate and compile code at run time, paying JIT cost per compilation, holding memory that is awkward to release, and giving up trimming and AOT entirely. It earns its place when the shape genuinely is not known until run time: a user-authored rules engine, a query compiler, a dynamic proxy. EF Core's own compiled queries are this done well.

The rule that generates both answers: **settle it at compile time when the shape is knowable then; defer to run time only when it genuinely is not.**

---

## 8. Nullable reference types move a failure class — they do not remove it

`<Nullable>enable</Nullable>` is the purest example of this movement, and also the one most often misread. It is **entirely a compile-time feature**. `string` and `string?` are the same type at run time; the annotations are metadata, and the compiler performs a flow analysis over the code it can see. Nothing is checked when the program runs.

Three consequences follow, and all three bite:

- **A non-nullable reference can absolutely be null at run time** if the value came from somewhere the compiler could not analyse: JSON deserialisation, EF Core materialisation, reflection, or a library compiled without annotations. The annotation is a *claim you are making*, not a guarantee the runtime enforces.
- **They are warnings, not errors,** unless you make them errors. A solution can be `<Nullable>enable</Nullable>` and still ship a thousand suppressed warnings, at which point you have the ceremony without the safety.
- **`!` is you overriding the analysis.** Sometimes correct, always worth a comment, and a file full of them means the annotations are wrong rather than the compiler.

So the discipline that makes it worth enabling is: annotate honestly, keep the warning count at zero, and **still validate at boundaries** — model binding, deserialisation, anything crossing into your code from outside. You have moved `NullReferenceException` from a production incident to a build warning for all the code the compiler can see, which is most of it. What crosses a boundary is still run time's problem, and always was.

---

## 9. None of this is arguable without counters

*How do you monitor performance* and *what profiling tools do you use* are the same question, and the useful answer is not a tool list — it is knowing **which number proves which of the three movements is your problem.**

| Movement | The counter that proves it | What "bad" looks like |
|---|---|---|
| Threads | `ThreadPool Queue Length`, `ThreadPool Thread Count` | Queue climbing while CPU is low — starvation, §2 |
| Memory | `GC Heap Size`, `Gen 2 GC Count`, `% Time in GC` | Gen2 heap rising across collections — retention, §4 |
| Allocation | `Allocation Rate` | High rate with a flat heap — churn, not a leak; §6 |
| Compile time | Startup duration, first-request latency | Slow cold start — reflection or JIT, §7 |

The tooling, in the order you reach for it:

- **`dotnet-counters`** — live counters against a running process, no instrumentation, no restart. This is the first thing to run, because it tells you which of the four rows above you are in.
- **`dotnet-trace`** — an EventPipe trace when you need to know *where*, not just *how much*.
- **`dotnet-dump` / `dotnet-gcdump`** — a heap you can interrogate. `gcdump` is small enough to pull from production; a full dump usually is not.
- **Visual Studio's profilers / PerfView / JetBrains dotMemory & dotTrace** — for the analysis session afterwards, on a machine that is not serving traffic.
- **OpenTelemetry, Application Insights** — continuous, aggregated, and the only one of these that tells you something was wrong at 3am on Tuesday.

The habit worth building: **measure before you optimise, and measure the counter that matches your hypothesis.** Almost every "async made no difference" story is someone fixing threads when the counter said memory.

---

## 10. The detail most people miss: the GC manages its heap, not your container

Modern .NET is container-aware. Since .NET Core 3.0 the runtime reads cgroup limits: `Environment.ProcessorCount` honours a CPU limit, and the GC sets a heap hard limit at roughly 75% of the container's memory limit. So far, so good — and this is exactly where the trap is, because it makes people assume the two numbers are the same number.

**The GC only manages the managed heap.** Thread stacks, native allocations, loaded assemblies, the JIT's own memory and any native library you P/Invoke into all live outside it. The container limit counts *all* of it. So a process can be killed for exceeding its memory limit while every GC counter you are watching looks healthy — because the part that grew was never the GC's to report.

**And an OOM kill is a `SIGKILL`.** There is no exception, no `finally`, no `IHostApplicationLifetime.ApplicationStopping`, no log line. The process simply ceases. Your telemetry shows a clean stream of successful requests and then nothing, which reads exactly like a network problem. The evidence lives in the orchestrator — `kubectl describe pod` reporting `OOMKilled`, or a restart count climbing — and teams routinely spend days looking for a bug in an application that was executed by the kernel.

**Server GC in a small container is usually the wrong default.** It allocates a heap and a background thread per core for throughput, which is the right trade on a box you own and the wrong one in a two-core sidecar with 512 MB, where it raises the memory floor to buy throughput you are not able to use anyway. `<ServerGarbageCollection>false</ServerGarbageCollection>` is often a large, free reduction in a container's memory footprint.

The general shape, which outlives the specific knobs: **the runtime optimises for the machine it believes it is on.** When you put it somewhere with a limit, you have to tell it — or verify that what it inferred matches what the orchestrator will actually enforce. It is chapter 02's lesson in another costume: the environment is a layer, and the defaults were chosen by someone who could not see yours.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| Latency climbs steeply under load while CPU stays low | Thread pool starvation — something is blocking pool threads, §2 |
| An endpoint works in testing and collapses at concurrency | `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` on a request path |
| An exception from a handler crashes the process instead of being caught | `async void` — nothing to observe the task |
| `A second operation was started on this context` | `Task.WhenAll` over work sharing one scoped `DbContext`; ch. 03 §7, ch. 04 |
| Work continues after the client has gone | No `CancellationToken` threaded through |
| Gen2 heap grows across collections and never returns | Retention — a root is holding it. Statics, events, an unbounded cache, §4 |
| Memory high, allocation rate high, heap flat | Churn, not a leak. Allocation reduction, not root hunting, §6 |
| Different replicas return different cached values | `IMemoryCache` is per instance; you needed `IDistributedCache`, §5 |
| A cache grows until the process dies | No size limit and no eviction policy — the leak with good PR |
| Trimming or AOT breaks the app at run time | Reflection the linker cannot see; a source generator is the fix, §7 |
| Slow cold start, fast once warm | JIT and startup reflection — ReadyToRun, AOT, or generators, §6–7 |
| A non-nullable property is null in production | The value crossed a boundary the compiler could not analyse, §8 |
| Pod restarts with no exception, no log, no stack trace | `OOMKilled`. Check the orchestrator, not the app, §10 |
| Container memory far above what GC counters report | Native memory, stacks and assemblies are outside the managed heap, §10 |

---

## Check yourself

Answer out loud, without looking:

1. `await` does not make a single request faster. Say precisely what it does make faster, and what has to be true for it to help at all.
2. Why does `.Result` deadlock in old ASP.NET but merely degrade in ASP.NET Core — and which of those two is harder to catch?
3. "There is a memory leak" in a .NET service. Why is that sentence technically wrong, and what does the correct version tell you to go looking for?
4. A cache with no eviction policy and a cache with a closed, known key space are the same code. Why is one a bug and the other fine?
5. A pod is being `OOMKilled` while `GC Heap Size` looks flat and healthy. Explain how both facts are true at once.

---

## Questions this chapter answers

Ten of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch05):

| # | As originally asked | Answered by section |
|---|---|---|
| 30 | Describe how to cache data | §5 — a cache is deliberate retention, so eviction is the design |
| 36 | What features improve application performance? | §6–7 — allocation you can decline, and work moved to build time |
| 37 | How can you monitor performance? | §9 — which counter proves which of the three movements |
| 38 | Discuss performance profiling tools | §9 — the order you reach for them, and what each answers |
| 39 | Purpose of asynchronous programming | §1–3 — occupancy, not speed |
| 40 | How do you identify and resolve memory leaks? | §4 — reachability makes it a search for a root |
| 95 | Source Generators | §7 — reflection's per-call bill, paid once at compile time |
| 96 | Dynamic compilations | §7 — the same trade in reverse, and when it earns its place |
| 97 | Local functions | §6 — no display class unless you convert one to a delegate |
| 98 | Nullable reference types in C# 8 | §8 — a failure class moved to the compiler, for code it can see |

Chapter 06 turns to the request again, and to the two questions the pipeline deliberately deferred: 27, 48, 50, 80–82, 84.

## Next

→ [`06-authentication-and-authorization.md`](06-authentication-and-authorization.md) — chapter 01 put `UseAuthentication` and `UseAuthorization` in the pipeline and did not say what they do. This is that, plus why the order between them is not arbitrary.
