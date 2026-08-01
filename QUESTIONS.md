# The question map

All 100 source questions, mapped to the chapter that dissolves them.

**How the mapping was done:** each question was reduced to the *underlying thing you'd need to understand* to answer any variation of it — not the topic word in the question. That's why Q33 (Kestrel) and Q83 (HSTS) land in the same chapter: both are answered by knowing what happens to a request between the socket and your controller. And it's why Q30 (caching) splits away from Q56 (EF Core) even though both mention data: one is about memory and instance scope, the other about change tracking.

A question can only live in one chapter. Where two chapters both touch it, it's assigned to the one whose mental model *generates* the answer, and the other chapter links to it.

**Legend:** ✅ chapter written · ⬜ planned. Numbers link back to each chapter's "questions this chapter answers" section once the chapter exists.

---

## Coverage at a glance

| Chapter | Questions | Count |
|---|---|---|
| [01 · The request pipeline](#ch01) ✅ | 16, 17, 18, 19, 24, 25, 33, 77, 83 | 9 |
| [02 · Configuration as layers](#ch02) ✅ | 9, 20, 23, 70, 75, 76, 78, 79 | 8 |
| [03 · DI and service lifetimes](#ch03) ✅ | 21, 22, 61, 62, 63, 64, 65 | 7 |
| [04 · EF Core's mental model](#ch04) ✅ | 28, 29, 56, 57, 58, 59, 60, 72 | 8 |
| [05 · Async, threads and memory](#ch05) ✅ | 30, 36, 37, 38, 39, 40, 95, 96, 97, 98 | 10 |
| [06 · Authentication and authorization](#ch06) ✅ | 27, 48, 50, 80, 81, 82, 84 | 7 |
| [07 · Testing and TDD](#ch07) ⬜ | 41, 42, 43, 44, 45 | 5 |
| [08 · Build and ship](#ch08) ⬜ | 13, 46, 47, 49, 51, 52, 54, 55 | 8 |
| [09 · Distributed and real-time](#ch09) ⬜ | 31, 32, 35, 53, 89 | 5 |
| [10 · Platform and history](#ch10) ⬜ | 1, 2, 3, 7, 14, 85, 86, 87, 88, 93, 94, 99, 100 | 13 |
| [11 · Interop and migration](#ch11) ⬜ | 26, 34, 66, 67, 68, 69, 90, 91, 92 | 9 |
| [12 · Tooling reference](#ch12) ⬜ | 4, 5, 6, 8, 10, 11, 12, 15, 71, 73, 74 | 11 |

100 questions, 12 chapters, no question in two places.

---

<a id="ch01"></a>
## 01 · The request pipeline ✅

**The concept:** a request travels inward through a chain of middleware; the response unwinds back outward in reverse. → [`concepts/01-the-request-pipeline.md`](concepts/01-the-request-pipeline.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 16 | Describe the MVC pattern in .NET Core | MVC's controller *is* the endpoint at the centre of the onion — the pattern only makes sense once you know what delivers a request to it |
| 17 | How do you set up a Web API in a .NET Core project? | "Setting up" a Web API is assembling a pipeline: register services, add middleware, map endpoints |
| 18 | What are middleware components in .NET Core? | The chapter's core mechanism, directly |
| 19 | Explain how static files are served | Static files = the canonical short-circuit; the interesting part is what they *skip* |
| 24 | How does routing work in a .NET Core MVC application? | The two-stage split (select vs execute) is the chapter's key insight |
| 25 | *(not visible in source — inferred as Razor Pages)* | Razor Pages is an alternative endpoint type at the centre of the same pipeline |
| 33 | Explain the role of a Kestrel server | Kestrel is what feeds the pipeline — the "before the onion" piece |
| 77 | Describe the role and configuration of middleware | Duplicate of 18 phrased as configuration; the ordering rules are the answer |
| 83 | Discuss HTTPS redirection and HSTS | Both are just middleware whose position in the chain explains their behaviour |

---

<a id="ch02"></a>
## 02 · Configuration as layers ✅

**The concept:** configuration is an ordered stack of providers where later sources override earlier ones — the same "ordered chain" idea as the pipeline, applied to settings. → [`concepts/02-configuration-as-layers.md`](concepts/02-configuration-as-layers.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 9 | What is the purpose of the `global.json` file? | Pinning — one more layer in "which value wins", at SDK level |
| 20 | Discuss the use and configuration of `appsettings.json` | The base layer of the stack |
| 23 | What are environment variables and how do they work? | The override layer that makes one image run in any environment |
| 70 | How do you manage user secrets in development? | A dev-only provider slotted into the same stack |
| 75 | How do you manage configurations for multiple environments? | *Is* the layering question, asked directly |
| 76 | What is the IOptions pattern? | How the resolved stack becomes typed, injectable objects |
| 78 | Explain how Configuration Builders are used | The mechanism that assembles the stack |
| 79 | How do you read command-line arguments? | The topmost override layer |

---

<a id="ch03"></a>
## 03 · DI and service lifetimes ✅

**The concept:** you declare *what* you need and *how long it lives*; the container decides *when it's built*. Most DI bugs are lifetime mismatches, and most "patterns" questions are DI questions in disguise. → [`concepts/03-di-and-service-lifetimes.md`](concepts/03-di-and-service-lifetimes.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 21 | What is Dependency Injection in .NET Core? | The core mechanism |
| 22 | How do you implement custom services and use DI? | Registration + lifetime choice, directly |
| 61 | What are popular design patterns in .NET Core? | The honest answer: most of them exist because of, or are replaced by, DI |
| 62 | Repository and Unit of Work patterns | Only answerable well when you know `DbContext` is already both — a scoped-lifetime insight |
| 63 | Explain CQRS | A dispatch pattern built on top of the container |
| 64 | Importance of domain-driven design | Service boundaries and composition — where DDD meets the container |
| 65 | How does .NET Core support SOLID? | The D in SOLID *is* this chapter; the rest follow from interface-based registration |

---

<a id="ch04"></a>
## 04 · EF Core's mental model ✅

**The concept:** the `DbContext` is a unit of work with a change tracker — you mutate tracked objects, and `SaveChanges` diffs and translates. Everything (migrations, transactions, performance) follows from that. → [`concepts/04-ef-cores-mental-model.md`](concepts/04-ef-cores-mental-model.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 28 | What is EF Core and how do you use it? | The core mechanism |
| 29 | How do you handle migrations in EF Core? | Schema-as-diff: the model is the source of truth, migrations are its version history |
| 56 | How do you work with databases using EF Core? | Duplicate of 28 phrased as workflow |
| 57 | What is the purpose of the DbContext? | *Is* the mental model, asked directly |
| 58 | Code-First and Database-First approaches | Two directions of the same model-vs-schema mapping |
| 59 | How do you handle database transactions? | `SaveChanges` is already a transaction — the question is when to widen it |
| 60 | Dapper as an alternative to EF | Only answerable by knowing what the change tracker costs and when to skip it |
| 72 | Explain the `dotnet ef` CLI tool | The tooling face of migrations; lives with the concept, not in the tooling grab-bag |

---

<a id="ch05"></a>
## 05 · Async, threads and memory ✅

**The concept:** nearly every performance question is a cost paid at the wrong time — a thread held while nothing is happening, memory held after you are done with it, or work done at run time that was knowable at compile time. → [`concepts/05-async-threads-and-memory.md`](concepts/05-async-threads-and-memory.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 30 | Describe how to cache data | Cache choice = instance scope + memory pressure question, not a data-access question |
| 36 | What features improve application performance? | GC, tiered JIT, `Span<T>` — the runtime story |
| 37 | How can you monitor performance? | Observing the same runtime: counters, traces, APM |
| 38 | Discuss performance profiling tools | Tooling over the same mental model |
| 39 | Purpose of asynchronous programming | The core mechanism |
| 40 | How do you identify and resolve memory leaks? | The GC model tells you exactly where to look: roots that never let go |
| 95 | Source Generators | Compile-time codegen exists to remove runtime reflection cost — a performance story |
| 96 | Dynamic compilations | The opposite trade: runtime codegen and what it costs |
| 97 | Local functions | Capture semantics and allocation — small, but it's a memory question |
| 98 | Nullable reference types in C# 8 | Moving a runtime failure class to compile time |

---

<a id="ch06"></a>
## 06 · Authentication and authorization ✅

**The concept:** authn establishes *identity* (a `ClaimsPrincipal`), authz evaluates *permission* against it — two separate middlewares, in that order, for a reason chapter 01 already explained. → [`concepts/06-authentication-and-authorization.md`](concepts/06-authentication-and-authorization.md)

| # | Question | Why it's this chapter |
|---|---|---|
| 27 | How do you ensure the security of an application? | An umbrella question; the spine of the answer is the auth chain |
| 48 | Security considerations when deploying | Deployment-facing phrasing of the same layered answer |
| 50 | How do you configure HTTPS and SSL? | TLS termination and where identity meets transport — pairs with the proxy section of ch. 01 |
| 80 | How does .NET Core handle authentication and authorization? | The core question, directly |
| 81 | What is the Identity system? | The membership store — when you own accounts vs delegate to OIDC |
| 82 | How to implement JWT authentication | The concrete mechanism of the authn half |
| 84 | How can you secure API endpoints? | Endpoint metadata + policies — the authz half applied |

---

<a id="ch07"></a>
## 07 · Testing and TDD ⬜

**The concept:** a test's value is what it *isolates* — units isolate one behaviour, integration tests isolate the wiring. Framework choice is a footnote.

| # | Question | Why it's this chapter |
|---|---|---|
| 41 | How do you write unit tests? | The core mechanism |
| 42 | xUnit, NUnit, and MSTest | The footnote — interchangeable, and the chapter says why that's the honest answer |
| 43 | Integration tests vs unit tests | *Is* the isolation question, asked directly |
| 44 | Logging API for troubleshooting | Observability as testing's production twin: both are about making behaviour inspectable |
| 45 | What is the TDD approach? | Red-green-refactor as a design pressure, not a ritual |

---

<a id="ch08"></a>
## 08 · Build and ship ⬜

**The concept:** `publish` produces a self-sufficient folder; everything after that — containers, orchestrators, serverless — is different answers to "where does that folder run and who scales it".

| # | Question | Why it's this chapter |
|---|---|---|
| 13 | Process of publishing a .NET Core application | The core mechanism |
| 46 | What is Docker and how can it be used? | The folder, containerised |
| 47 | Deploy to the cloud (e.g. Azure) | The folder, hosted PaaS |
| 49 | Strategies for scaling | Horizontal vs vertical = "how many copies of the folder" |
| 51 | Integrate Azure services | The folder's runtime dependencies, injected |
| 52 | Azure DevOps for CI/CD | The pipeline that produces the folder |
| 54 | Container orchestration (Kubernetes) | Who schedules and heals the copies |
| 55 | Serverless (AWS Lambda) | The folder dissolved into per-invocation execution |

---

<a id="ch09"></a>
## 09 · Distributed and real-time ⬜

**The concept:** once there's more than one process, everything is a contract plus a transport — REST, gRPC, SignalR, and queues are four answers to "who initiates, and what survives a failure".

| # | Question | Why it's this chapter |
|---|---|---|
| 31 | What is SignalR and its use cases? | Server-initiated transport |
| 32 | Build real-time applications | Same question phrased as a goal |
| 35 | How do you perform logging? | In one process it's diagnostics; across services it's correlation — it earns its place here |
| 53 | Best practices for microservices architecture | The contract question at system scale |
| 89 | Support for gRPC | Typed, binary, service-to-service transport |

---

<a id="ch10"></a>
## 10 · Platform and history ⬜

**The concept:** one timeline — Framework → Core → unified .NET — explains every "what is / what's the future of / how does X relate" question. Reference chapter, deliberately thin.

| # | Question | Why it's this chapter |
|---|---|---|
| 1 | What is .NET Core vs .NET Framework? | The timeline's starting point |
| 2 | Cross-platform capabilities | *Why* Core was built |
| 3 | Main components of the architecture | Runtime / BCL / SDK — the platform's anatomy |
| 7 | What is the runtime and SDK? | Same anatomy, narrower |
| 14 | What is .NET Standard? | A compatibility artifact of the timeline — meaningful only historically |
| 85 | Future with .NET 5 and later | The unification event |
| 86 | How .NET 5 changed .NET Core | Same event, other direction |
| 87 | .NET Core in IoT | Platform reach |
| 88 | Desktop development with MAUI | Platform reach |
| 93 | New features in the latest version | The cadence (LTS/STS) more than any one feature |
| 94 | Machine learning with ML.NET | Platform reach |
| 99 | Open source community contribution | How the platform is developed |
| 100 | Support options for developers | The LTS policy in practice |

---

<a id="ch11"></a>
## 11 · Interop and migration ⬜

**The concept:** old and new .NET meet at *boundaries* — processes, contracts, or shared netstandard libraries — never by linking runtimes. Every migration question is "where do you draw the boundary".

| # | Question | Why it's this chapter |
|---|---|---|
| 26 | Tag Helpers in ASP.NET Core | Razor's server-rendered world — the boundary with the pre-SPA model |
| 34 | What is Blazor? | C# crossing the browser boundary |
| 66 | Integrate with legacy .NET Framework apps | The core question |
| 67 | Consume COM objects | The Windows-only boundary |
| 68 | Migrating from Framework to Core | Boundary-drawing as a strategy |
| 69 | Challenges of interoperability | The same boundaries, enumerated as risks |
| 90 | Integrate Angular/React/Vue with a Web API | The frontend/backend contract boundary |
| 91 | SPA templates | Hosting choice at that same boundary |
| 92 | Server-side rendering with JS frameworks | Where rendering happens relative to the boundary |

---

<a id="ch12"></a>
## 12 · Tooling reference ⬜

**The concept:** none — and that's the point. These are lookup facts about the `dotnet` CLI and project system. The chapter is a reference table, not an essay, because pretending there's a deep model here would be padding.

| # | Question | Why it's this chapter |
|---|---|---|
| 4 | The .NET Core CLI and its functions | CLI surface |
| 5 | Create a new project using the CLI | `dotnet new` |
| 6 | Purpose of a `.csproj` file | Project system |
| 8 | Manage different SDK versions | Side-by-side installs (`global.json` itself is in ch. 02) |
| 10 | Directory structure of a typical project | Convention reference |
| 11 | Add and manage NuGet packages | Package workflow |
| 12 | Role of the NuGet package manager | Same, one level up |
| 15 | Create a class library | `dotnet new classlib` |
| 71 | The watch command | Inner-loop tooling |
| 73 | Scaffolding | Codegen tooling |
| 74 | Run and debug from the CLI | Inner-loop tooling |

---

## Judgement calls worth recording

These are the assignments that could have gone another way, and why they didn't:

- **Q30 (caching) → ch. 05, not ch. 04.** The hard part of caching isn't storing data, it's *instance scope* (in-memory vs distributed = one process vs many) and memory pressure. That's the async/memory model's territory.
- **Q35 (logging) → ch. 09, not ch. 07.** In a single process logging is simple. It becomes a concept when a request crosses services and you need correlation — a distributed-systems problem.
- **Q72 (`dotnet ef`) → ch. 04, not ch. 12.** The tool is meaningless without the migrations model; splitting them would force ch. 12 to re-explain it.
- **Q9 (`global.json`) → ch. 02; Q8 (SDK versions) → ch. 12.** Deliberately split: Q9 is about *pinning as layering* (the config concept), Q8 is the mechanical fact that SDKs install side-by-side.
- **Q26 (Tag Helpers) and Q34 (Blazor) → ch. 11.** Neither is "advanced .NET Core" as the source list claims — both are about which side of a rendering boundary your UI lives on, which is the interop chapter's actual subject.
- **Q25 was not visible in the source screenshots** (image with Q16–24 ends, next begins at Q26). Filled as Razor Pages — the standard question in that slot — and flagged in ch. 01. Replace if the real one differs.
