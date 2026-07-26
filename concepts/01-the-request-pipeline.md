# The request pipeline

> **The one idea:** ASP.NET Core is a *pipe*. A request travels inward through a chain of components, reaches your code, and the response travels back out through the same components in reverse. Everything else in this chapter is a consequence of that sentence.

---

## Why this concept exists

Every web framework has to answer the same question: between the moment a byte arrives on a socket and the moment your controller method runs, what happens?

Old ASP.NET answered with a fixed set of lifecycle events you hooked into (`Application_BeginRequest`, and friends). You didn't compose the pipeline; you subscribed to a pipeline someone else designed.

ASP.NET Core threw that out and replaced it with something much simpler: **a chain of functions, each of which can call the next one.** You build the chain yourself, in order, in `Program.cs`. That's it. That's the whole architecture.

Once you internalize that, roughly nine of the "100 questions" stop being separate facts.

---

## 1. Kestrel is not middleware

Kestrel is the **web server** — it owns the socket, speaks HTTP, and does the unglamorous work of parsing bytes into a request. It builds an `HttpContext` object (request, response, connection info, user identity) and hands it to the front of your pipeline.

So the shape is:

```
network → Kestrel → [ your middleware pipeline ] → your endpoint
```

Kestrel is cross-platform and genuinely fast, which is why .NET benchmarks well. In production it usually sits behind a reverse proxy (nginx, IIS, YARP) that handles TLS termination, load balancing, and serving static assets.

**Your setup:** in GROS, nginx *is* that proxy — it terminates TLS and routes per app. So there are two pipelines stacked: nginx's, then Kestrel's. This matters more than it sounds; see "Behind a proxy" below.

---

## 2. Middleware is a nested onion, not a queue

This is the part almost everyone gets wrong on the first pass.

A middleware component is just this shape:

```csharp
app.Use(async (context, next) =>
{
    // A: runs on the way IN
    await next();
    // B: runs on the way OUT
});
```

Code before `await next()` runs as the request travels inward. Code after it runs as the response travels back outward — **in reverse registration order**.

So if you register three components, execution is not `1 → 2 → 3`. It's:

```
1-in → 2-in → 3-in → [endpoint] → 3-out → 2-out → 1-out
```

They're nested, like layers of an onion. The first one you register is the outermost layer — it sees the request first and the response last.

**Why this matters:** it's the reason exception handling works. `UseExceptionHandler` registered first means every later component runs *inside* its `try` block. Register it fifth and it can only catch exceptions from components six onward. It's not a convention — it's the call stack.

### `Use` vs `Run` vs `Map`

- **`Use`** — can call `next()` and pass control inward. The normal case.
- **`Run`** — *terminal*. Never calls `next()`. The pipeline stops here.
- **`Map`** — branches the pipeline on a path prefix. `app.Map("/admin", ...)` builds a separate sub-pipeline.

---

## 3. Short-circuiting is a feature, not an edge case

A middleware that returns **without calling `next()`** ends the request there. The response starts unwinding immediately, and nothing deeper ever runs.

This is deliberate and used constantly:

| Middleware | Short-circuits when | Result |
|---|---|---|
| Static files | the file exists in `wwwroot` | serves it, skips routing/auth entirely |
| HTTPS redirection | request arrived over http | 301, no further work |
| Authorization | policy fails | 401/403, controller never runs |
| Response caching | a valid cached response exists | served from cache |
| Rate limiting | quota exceeded | 429 |

**The gotcha worth remembering:** because static files short-circuit *before* authentication and authorization, anything in `wwwroot` is public. If you drop a sensitive PDF there and assume `[Authorize]` protects it — it doesn't. `[Authorize]` lives on endpoints, and static files never reach endpoint routing.

---

## 4. Routing is two middlewares, and this explains the ordering rule

Here's the insight that makes the canonical order stop being a list to memorize.

Routing is split into **two separate stages**:

1. **`UseRouting`** — *selects* the endpoint. It matches the URL against registered routes, then attaches the chosen endpoint (and all its metadata — including `[Authorize]` attributes) onto the `HttpContext`. It does **not** execute anything.
2. **`MapControllers` / `UseEndpoints`** — *executes* the selected endpoint.

Everything registered **between** those two can inspect which endpoint was chosen.

And that's exactly why authorization sits there. `UseAuthorization` works by reading the `[Authorize]` metadata off the selected endpoint. If no endpoint has been selected yet, there's no metadata — so it finds nothing to enforce and **lets the request through**.

Put `UseAuthorization` before `UseRouting` and you don't get an error. You get a silently unprotected API. That's the failure mode: not a crash, a security hole.

---

## 5. The canonical order, with reasons

Never memorize this list. Derive it from the four reasons on the right.

```csharp
app.UseExceptionHandler("/error");   // outermost — can only catch what's inside it
app.UseHsts();                       // header, before real work
app.UseHttpsRedirection();           // bounce http early, don't waste work
app.UseStaticFiles();                // short-circuit assets before auth/routing
app.UseRouting();                    // SELECT endpoint → attaches metadata
app.UseCors();                       // after routing: per-endpoint CORS policy
app.UseAuthentication();             // who are you?  → builds ClaimsPrincipal
app.UseAuthorization();              // are you allowed? → reads endpoint metadata
app.MapControllers();                // EXECUTE endpoint
```

The four load-bearing reasons:

1. **Exception handling is outermost** because it's a `try` block, and a `try` only catches what it encloses.
2. **Cheap short-circuits go early** so you don't run auth and routing to serve a `.css` file.
3. **Routing before authorization** because authorization reads endpoint metadata that routing attaches.
4. **Authentication before authorization** because you can't evaluate permissions before you know who's asking. *Authn = identity. Authz = permission.*

---

## 6. Behind a reverse proxy — the detail most people miss

When nginx sits in front of Kestrel, the app sees the *proxy's* connection, not the client's. So `Request.Scheme` says `http` (nginx already terminated TLS) and `RemoteIpAddress` is nginx's IP.

Consequences: HTTPS redirection can loop, generated absolute URLs come out as `http://`, and logged client IPs are useless.

The fix is `UseForwardedHeaders`, registered **very early** — it reads `X-Forwarded-For` / `X-Forwarded-Proto` and rewrites the context before anything else looks at it.

```csharp
app.UseForwardedHeaders(new ForwardedHeadersOptions {
    ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto
});
```

This is directly relevant to your nginx-gateway topology, and it's a great thing to be able to explain — it shows you've actually deployed behind a proxy rather than just read about one.

---

## 7. Writing your own

Two forms. Inline for something small:

```csharp
app.Use(async (ctx, next) =>
{
    var sw = Stopwatch.StartNew();
    await next();
    logger.LogInformation("{Path} took {Ms}ms", ctx.Request.Path, sw.ElapsedMilliseconds);
});
```

Note that the timing works *because* of the onion — `next()` returns only after everything deeper has finished.

Or a class with `InvokeAsync`, registered via `app.UseMiddleware<T>()`. Important detail: middleware classes are instantiated **once** (effectively singleton), so inject scoped services as a parameter on `InvokeAsync`, not through the constructor. Constructor-injecting a `DbContext` into middleware is a classic captive-dependency bug.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| `[Authorize]` silently ignored | `UseAuthorization` before `UseRouting` |
| Exceptions escape your handler | handler registered too late |
| Endless redirect loop behind proxy | missing `UseForwardedHeaders` |
| Static file served without auth | expected — static files short-circuit first |
| `Cannot set headers, response already started` | writing headers after `next()` — the response already flushed |
| Scoped service disposed in middleware | injected in constructor instead of `InvokeAsync` |

---

## Check yourself

Answer out loud, without looking:

1. Why must the exception handler be registered first — what's the mechanical reason?
2. What actually breaks if `UseAuthorization` comes before `UseRouting`, and why is it dangerous rather than just broken?
3. A `.css` file is served but never authenticated. Bug or design?
4. You add timing middleware and it reports ~0ms. What did you get wrong?
5. Why does `UseRouting` exist separately from `MapControllers`?

---

## Questions this chapter answers

Nine of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch01):

| # | As originally asked | Answered by section |
|---|---|---|
| 16 | Describe the MVC pattern | §1–2 — the controller is the endpoint at the onion's centre |
| 17 | How do you set up a Web API? | §5 — setup *is* assembling the pipeline |
| 18 | What are middleware components? | §2 — the onion, `Use`/`Run`/`Map` |
| 19 | How are static files served? | §3 — the canonical short-circuit |
| 24 | How does routing work? | §4 — the two-stage select/execute split |
| 25 | *(inferred: Razor Pages)* | §4 — an alternative endpoint type, same pipeline |
| 33 | Explain the role of Kestrel | §1 — what feeds the pipeline |
| 77 | Role and configuration of middleware | §5 — the canonical order, derived not memorised |
| 83 | HTTPS redirection and HSTS | §3, §5 — middleware whose position explains their behaviour |

Chapter 06 will pick up the questions this one only borders on: 27, 48, 50, 80 (the auth chain itself).

## Next

→ `02-configuration-as-layers.md` — the other "ordered chain" concept, and it works the same way.
