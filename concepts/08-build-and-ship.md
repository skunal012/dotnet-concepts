# Build and ship

> **The one idea:** `dotnet publish` produces a self-sufficient folder. Everything after that — Docker, Kubernetes, App Service, Lambda — is a different answer to two questions: *where does that folder run*, and *who makes more copies of it*.

---

## Why this concept exists

Deploying old ASP.NET meant putting files somewhere IIS was already configured to look, on a machine where the right .NET Framework version was already installed, with the right modules registered and the right app pool identity. The application was not a thing you could move; it was a *configuration of a server* that happened to include your code.

.NET Core inverted that. The output of a build is a directory containing your assemblies, your dependencies, your static assets and — if you ask — the runtime itself. It has no opinion about the machine. Copy it somewhere with a compatible runtime and run the executable, and you have a working application.

That is the whole concept, and the reason it matters is that **every deployment technology in this chapter is a wrapper around that folder.** A container is the folder plus a filesystem to stand on. Kubernetes is a thing that keeps N copies of that container alive. App Service is someone else running the folder for you. Lambda is the folder invoked per request rather than left running. Learn the folder and the rest are variations, which is considerably more durable than learning four vendors' portals.

Chapter 07 ended by arguing that a test suite nothing enforces is decoration. The pipeline in §8 is what enforces it, and it produces exactly one of these folders.

---

## 1. What `publish` actually produces

`dotnet build` compiles. `dotnet publish` compiles *and* gathers everything needed to run into one output directory — your assemblies, NuGet dependencies copied out of the package cache, `appsettings.json`, `wwwroot`, and a `.deps.json` and `.runtimeconfig.json` telling the host what to load.

```bash
dotnet publish -c Release -o ./out
```

Since .NET 8, `publish` defaults to Release, which quietly removed a long-standing way to ship a debug build by accident.

The one real decision is **who supplies the runtime**:

| | Framework-dependent | Self-contained |
|---|---|---|
| Runtime | Must already be on the machine | Shipped inside the folder |
| Size | Small — a few MB | Large — 60 MB+ |
| Needs a RID | No | Yes — `linux-x64`, `win-x64`, … |
| Patched by | The host's runtime updates | You, by rebuilding |

Framework-dependent is the sensible default when you control the base image or the host, because a runtime security patch arrives by rebasing rather than by rebuilding your application. Self-contained earns its place when you cannot rely on what is installed — a desktop tool, an appliance, a locked-down server.

Layered on top are the switches chapter 05 §6 was building towards, all of them trading flexibility for startup or size:

- **`PublishReadyToRun`** — pre-JITs to native code alongside the IL. Faster startup, bigger folder.
- **`PublishSingleFile`** — bundles the folder into one executable. A packaging convenience, not a performance feature.
- **`PublishTrimmed`** — removes unreferenced code. Self-contained only, and it will break anything using reflection the linker cannot see — chapter 05 §7's argument arriving as a build error.
- **`PublishAot`** — compiles ahead of time, no JIT at all. The fastest start and smallest footprint, at the cost of no runtime code generation and hard trimming constraints.

And what is **not** in the folder matters just as much: the configuration for the environment it will run in. `appsettings.Production.json` may ship, but connection strings and secrets arrive at run time from the provider stack (ch. 02). That is the property that makes the next seven sections possible — **the artifact is the same everywhere, and only its inputs change.**

---

## 2. A container is the folder plus the floor it stands on

A container image is your published folder plus the minimum operating system underneath it, frozen together and addressed by digest. The value is not isolation — it is that "works on my machine" becomes a statement about an artifact rather than about a machine.

The standard build is two-stage, and both stages are doing something specific:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["Gros.Api/Gros.Api.csproj", "Gros.Api/"]
RUN dotnet restore "Gros.Api/Gros.Api.csproj"     # ← cached unless the csproj changes
COPY . .
RUN dotnet publish "Gros.Api/Gros.Api.csproj" -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final   # runtime only — no SDK, no source
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
ENTRYPOINT ["dotnet", "Gros.Api.dll"]
```

Three details in there are the difference between a good image and a bad one:

**The copy order is a caching decision.** Copying the `.csproj` and restoring *before* copying the source means the restore layer is reused on every build where dependencies did not change — which is almost all of them. Copy everything first and you re-download the world on every commit. Add a `.dockerignore` for `bin`, `obj`, `.git` and `node_modules`, or you invalidate that cache with files you never needed and ship a much larger context.

**The final stage is the runtime image, not the SDK.** The SDK image is roughly ten times the size and contains a compiler, a package manager and your source code. Shipping it to production is both waste and a larger attack surface.

**`USER $APP_UID` runs as non-root.** The .NET 8 images define that variable, and the default listening port moved from 80 to 8080 in the same release specifically so a non-root user could bind it. A container running as root is one container-escape bug from being a host compromise.

The property that makes containers worth the ceremony is **immutability**: the image that passed your tests is bit-for-bit the image in production, referenced by digest. That is chapter 02's core argument — one artifact, many environments — finally delivered by the packaging rather than promised by discipline.

---

## 3. Configuration is what turns one image into many deployments

If the artifact is identical everywhere, something has to differ, and chapter 02 already established what: the provider stack, resolved at run time.

In a container that means environment variables, which is why `__` is the nesting separator (ch. 02 §3) — `ConnectionStrings__Gros` works in a Dockerfile, a Compose file, a Kubernetes manifest and an App Service setting, none of which can express a colon.

For secrets specifically, the progression is worth naming because each step removes a copy of the secret:

1. **Environment variables** — visible to anything that can read the process environment, and to `docker inspect`. Fine for non-secrets.
2. **The orchestrator's secret store** — Kubernetes Secrets, Container Apps secrets. Mounted at run time, not baked into the image.
3. **A vault** — Key Vault, Secrets Manager, mounted as a configuration provider so the code still just reads `IConfiguration`.
4. **No secret at all** — managed identity. The platform gives the process a token for a resource it is authorised to reach, and there is nothing to leak.

That last one is the honest answer to *how do you integrate Azure services*. A queue, a blob store, a SQL database or Key Vault all arrive the same way: a connection detail from configuration, a client registered in DI (ch. 03), and credentials that are ideally not credentials at all:

```csharp
builder.Services.AddSingleton(_ => new BlobServiceClient(
    new Uri(builder.Configuration["Storage:Uri"]!), new DefaultAzureCredential()));
```

`DefaultAzureCredential` resolves to your developer login locally and to the managed identity in production — the same code, different inputs, which is the chapter's thesis applied to authentication. Note the lifetime: these clients are expensive, thread-safe and pool connections, so they are singletons, and chapter 03 §5's obligation applies.

---

## 4. Scaling is a question about state, not about machines

**Vertical** scaling is a bigger machine. Simple, bounded, and it does nothing for availability — one machine is still one thing that can die.

**Horizontal** scaling is more copies of the folder. Effectively unbounded, and it gives you redundancy for free. It is the right answer almost always, and it has exactly one requirement: **the copies must not need to know about each other.**

That requirement is the whole section, because "stateless" is not a property you get by intending it. Here is what actually breaks the first time replica count goes from one to three, and every item is something this handbook has already met:

| What breaks | Why | Fix |
|---|---|---|
| Cached data is inconsistent between requests | `IMemoryCache` is per process (ch. 05 §5) | `IDistributedCache` for anything a user can observe |
| Sessions vanish intermittently | In-memory session state, same reason | Distributed session, or no server session at all |
| A scheduled job runs three times | Every replica has the same timer | A leader election, or a scheduler outside the app |
| Real-time messages reach some clients | SignalR connections are per instance | A backplane — ch. 09 |
| Deploys race on the database | Every replica calls `Migrate()` (ch. 04 §8) | An idempotent script as a deploy step |
| Users are logged out at random | **Data protection keys are per instance — §9** | A shared key ring |

Sticky sessions are the tempting shortcut, and they are worth being clear about: pinning a user to one replica does not make the application stateless, it hides that it is not. You lose even load distribution, you lose the ability to drain a node cleanly, and every affected user's session dies with the instance. It is a workaround for an application that has not been made horizontally scalable, and it should be labelled as such rather than treated as an architecture.

Autoscaling is then just a rule over that: a metric, a threshold, a cooldown. The thing worth knowing is that scaling out takes time — pulling an image, starting a process, warming a JIT (ch. 05 §6) — so a rule tuned to a slow-moving metric will always be reacting to a load that has already arrived.

---

## 5. Where the folder runs: four answers, one axis

Every hosting option is a trade on the same axis — **how much of the floor do you want to own**. Reading down, you give up control and get back operational burden:

| | You provide | Platform provides | Scales by | Give up |
|---|---|---|---|---|
| **VM** | OS, runtime, process supervision | Hardware | You do it | Nothing; you own everything |
| **PaaS** (App Service) | The folder, or an image | OS, runtime, TLS, restarts, autoscale | A slider or rule | Fine control over the host |
| **Containers** (Container Apps, ECS) | An image | Everything under it, plus scale-to-zero | Rules on requests or metrics | Access to the node |
| **Orchestrator** (Kubernetes) | An image + declared desired state | Scheduling, healing, rollout | A replica count you declare | A large amount of your time |
| **Serverless** (Functions, Lambda) | A function | Everything, including the process lifetime | Per invocation, automatically | Long-running state, and startup control |

The honest guidance, which is not what a vendor will tell you: **take the highest row that meets your requirements.** A single API with a database is well served by PaaS or Container Apps, and the team that reaches for Kubernetes at that scale usually spends the following year on infrastructure rather than on the product. Kubernetes earns its keep when you have many services, real multi-tenancy, or a platform team whose job it is.

**Calibration:** my production experience here is container-side — Docker images behind nginx, which is the "containers" row with the reverse proxy from chapter 01 §6 in front. What follows on Kubernetes and Lambda is accurate as far as I have used and read them, but I would not claim to have run a large cluster.

---

## 6. Kubernetes is a control loop, not a deployment tool

The single idea worth taking from Kubernetes, and the one that makes everything else make sense: **you declare desired state, and a controller continuously works to make actual state match.** You do not tell it to start three pods. You tell it that three pods should exist, forever, and it keeps checking.

That reframing explains the parts:

- A **Deployment** declares "N replicas of this image". Kill a pod and one reappears — not because something reacted to your action, but because the loop noticed a mismatch.
- A **Service** gives that changing set of pods one stable address, because individual pods are cattle with changing IPs.
- A **rolling update** is the same loop with a constraint: replace pods gradually, keeping a minimum available. Which is exactly why chapter 04 §8's warning matters — during a rollout, old and new code are both live against one database, so a destructive migration breaks the version still serving traffic.

**Probes are where the reframing pays off**, and where the most damaging mistake lives:

- **Liveness** — "is this process broken?" Failing it means *restart the container*.
- **Readiness** — "should this receive traffic right now?" Failing it means *remove it from the load balancer*, without restarting.
- **Startup** — "has it finished booting?" Suppresses the other two until it passes.

```csharp
builder.Services.AddHealthChecks().AddNpgSql(conn, tags: ["ready"]);

app.MapHealthChecks("/health/live",  new() { Predicate = _ => false });          // process only
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

The mistake is checking dependencies in the **liveness** probe. Put the database in it and a thirty-second database blip fails liveness on every replica simultaneously, so Kubernetes restarts your entire fleet — turning a recoverable dependency hiccup into a full outage, and adding a thundering herd of reconnecting pods to a database that was already struggling. Liveness answers *is this process wedged*. Dependencies belong in readiness, where the response is to stop sending traffic and wait.

Resource requests and limits are the other half, and chapter 05 §10 is the reason: the limit is what the kernel enforces, the GC only manages its own heap, and exceeding the limit is a `SIGKILL` with no exception and no log line.

---

## 7. Serverless dissolves the folder into invocations

Serverless keeps the code and discards the *process*. There is no long-running host; the platform instantiates your function, runs it, and may destroy it. You are billed per invocation and duration rather than per hour.

What follows from having no durable process:

- **Cold starts are the defining cost.** The first invocation on a new instance pays for provisioning, loading the runtime and JIT-compiling. This is precisely why §1's ReadyToRun and AOT switches exist, and where they pay for themselves most clearly.
- **Anything expensive must be initialised outside the handler.** A static or constructor-scoped `HttpClient`, database connection or DI container is reused while the instance stays warm; constructing it per invocation pays the cost every time. This is chapter 03's lifetime question with the process boundary removed.
- **There is no ambient state between invocations**, and no guarantee the next call reaches the same instance. In-memory anything is a cache with an unpredictable, unannounced eviction policy.
- **Execution time is capped**, so long jobs must be decomposed or moved to a durable workflow.

The fit is genuinely good for event-driven, spiky, or infrequent work — a queue consumer, a webhook receiver, a nightly job — where scale-to-zero means paying nothing to be available. It fits badly where cold starts are user-visible on a latency-sensitive path, or where the work is long-running and stateful.

**Calibration:** I have not shipped .NET on Lambda. The mechanism above is the general model and holds across Azure Functions and Lambda; I would defer to someone who has run it in anger on the specifics of the .NET custom runtime and its cold-start numbers.

---

## 8. CI/CD builds the folder once and promotes it

The pipeline's job is not "run some steps". It is to **produce exactly one artifact and move that same artifact through environments** — which is chapter 02's founding argument, now as infrastructure rather than as configuration.

```
commit → restore → build → test → publish artifact
                                        ↓
                          staging ──gate──▶ production
                       (same image, different configuration)
```

The rule that makes it worth doing: **build once, deploy many.** If production is built by a separate job from staging, you have tested one artifact and shipped a different one, and every argument in chapter 02 about `web.config` transforms applies again with more YAML. Promote the digest.

Which makes chapter 07 §9 enforceable rather than aspirational: the test step must fail the build. A pipeline with `continueOnError: true` on its test job reports failures and enforces nothing, and is worse than no pipeline because it manufactures confidence.

The stages that earn their place:

- **Build and test** — one job, one artifact, tests gating it. Integration tests (ch. 07 §5) with a real database in a service container.
- **Publish** — the image, pushed to a registry, tagged with the commit SHA. Not `latest`, which is a name that does not identify anything.
- **The database migration script** — generated as an artifact (`--idempotent`, ch. 04 §8) and applied as an explicit step *before* the new version starts, not by the application on boot.
- **Deploy** — the digest, plus environment-specific configuration.
- **A gate before production** — approval, a smoke test, or a soak in staging.

Azure DevOps and GitHub Actions differ in syntax and in almost nothing conceptually: triggers, jobs, steps, artifacts, environments with approvals, and secrets injected at run time. Whichever you use, the reviewable question is the same — *is the thing being deployed to production the identical bytes that passed the tests?*

And deployment strategy is a rollout choice on top: **rolling** replaces gradually with both versions live (see §6's migration warning), **blue-green** stands up a parallel environment and switches traffic, giving a fast rollback at double the infrastructure, and **canary** sends a fraction of traffic to the new version first. All three assume your database change is backward-compatible with the version still running, which is the constraint people discover last.

---

## 9. The detail most people miss: data protection keys are per instance

Everything ASP.NET Core encrypts for its own use — authentication cookies, antiforgery tokens, `TempData`, anything through `IDataProtector` — is protected with a key from the **data protection key ring**. By default that ring is generated on first use and written to the local filesystem.

In a single long-lived VM this is invisible, and it works for years. Then you containerise, or scale to three replicas, and the assumption underneath it quietly stops holding:

- **Each replica generates its own key ring.** A cookie issued by replica A cannot be decrypted by replica B, so a user round-robined across instances is logged out apparently at random. Antiforgery validation fails on a form whose page was rendered by a different pod (ch. 06 §3).
- **A container filesystem is ephemeral.** The ring is regenerated on every restart and every deploy, so *every* user is signed out on each release — which teams often accept as normal rather than recognising as a bug.
- **The symptoms do not name the cause.** "Users get logged out sometimes", "the login page says the antiforgery token is invalid", "it worked before we scaled". Nothing in the logs says *key*, and the natural suspects — session, load balancer, cookie expiry — are all wrong.

The fix is to put the ring somewhere shared and durable, and to say explicitly which applications share it:

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(blobUri, credential)   // or Redis, or a mounted volume
    .SetApplicationName("gros");                           // same name = same ring
```

`SetApplicationName` matters because the application name is part of the key derivation. Two replicas of the same app get it right automatically; two *different* apps that must share cookies — an MVC front end and an API on the same domain — will not, unless you tell them.

The general lesson outlives the specific API, and it is the one this chapter is really about: **the folder is stateless, but the framework quietly kept some state anyway.** Every default in .NET was chosen for some deployment topology, and the local filesystem was a reasonable choice when there was one long-lived server. Horizontal scaling (§4) invalidates that assumption without warning you, and this is the case where it is invisible until it is a support ticket you cannot reproduce.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| Users logged out at random after scaling out | Per-instance data protection keys — §9 |
| Everyone signed out on every deploy | The key ring lives on an ephemeral container filesystem, §9 |
| Antiforgery token invalid on a form that just rendered | Page and post handled by different replicas with different keys |
| `dotnet publish` output will not run on the target | Framework-dependent build, runtime missing — or wrong RID for self-contained |
| Docker builds re-download every package on every commit | Source copied before `restore`; the layer cache is invalidated, §2 |
| The production image is over a gigabyte | Shipping the SDK image rather than the runtime image |
| Reflection-based code breaks only in the published build | `PublishTrimmed` removed what the linker could not see — ch. 05 §7 |
| The whole fleet restarts when the database blips | A dependency check in the **liveness** probe, §6 |
| Pods restart with no exception or log | Memory limit exceeded — `SIGKILL`, ch. 05 §10 |
| A scheduled job runs once per replica | Every instance has its own timer, §4 |
| Cached values differ between requests | `IMemoryCache` per process — ch. 05 §5 |
| A deploy half-applies migrations | Replicas racing `Migrate()` on startup — ch. 04 §8 |
| Old version errors during a rolling deploy | A destructive schema change while both versions are live, §6, §8 |
| Staging passed and production failed identically-configured | Production was rebuilt rather than promoted, §8 |
| CI is green but tests are failing | The test step does not fail the build — ch. 07 §9 |

---

## Check yourself

Answer out loud, without looking:

1. What is in a `dotnet publish` folder that is not in a `dotnet build` folder, and what is deliberately *not* in either?
2. Why does copying the `.csproj` and restoring before copying the source change how long your builds take?
3. Replica count goes from one to three and users start getting logged out at random. Give the cause and say why nothing in the logs mentions it.
4. A liveness probe and a readiness probe both check the database. Describe the outage.
5. Your pipeline builds separately for staging and production. Name what you have lost, and which earlier chapter made the same argument about configuration.

---

## Questions this chapter answers

Eight of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch08):

| # | As originally asked | Answered by section |
|---|---|---|
| 13 | Process of publishing a .NET Core application | §1 — the folder, and who supplies the runtime |
| 46 | What is Docker and how can it be used? | §2 — the folder plus a floor, made immutable |
| 47 | Deploy to the cloud (e.g. Azure) | §5 — how much of the floor you want to own |
| 49 | Strategies for scaling | §4 — horizontal scaling is a statelessness requirement |
| 51 | Integrate Azure services | §3 — configuration plus a DI registration, ideally with no secret |
| 52 | Azure DevOps for CI/CD | §8 — build once, promote the same artifact |
| 54 | Container orchestration (Kubernetes) | §6 — a control loop reconciling declared state |
| 55 | Serverless (AWS Lambda) | §7 — the folder without a durable process |

Chapter 09 picks up the two things §4 deferred: what happens when the copies must talk to each other, and how you follow one request across them — 31, 32, 35, 53, 89.

## Next

→ [`09-distributed-and-real-time.md`](09-distributed-and-real-time.md) — §4 listed SignalR backplanes and a scheduled job running three times as things that break when there is more than one copy. Those are not deployment problems; they are what happens when processes have to agree.
