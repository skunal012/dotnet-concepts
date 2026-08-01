# Testing and TDD

> **The one idea:** a test is worth what it *isolates*. Change one behaviour and exactly one test should fail — and whether that is possible is a property of your design long before it is a property of your tests.

---

## Why this concept exists

The reason to write tests is not to prove code correct. It is to make a change to a large system without having to hold the whole system in your head, and to find out in seconds rather than in production whether you broke something you had forgotten existed.

That framing decides everything else. A suite that takes forty minutes does not give you seconds. A suite where one behaviour change reddens ninety tests does not tell you what you broke — it tells you that you touched something. A suite that has never failed tells you nothing at all (§9). None of those are testing-framework problems, which is why the framework is the least interesting decision in this chapter.

The connection to everything before it is direct. Chapter 03 argued that `new` is a decision made at the wrong time and that constructor injection creates a seam. **A test is what that seam is for.** If a class is hard to test, it is almost never because testing is hard — it is because the class has a dependency it never declared, and the test is the first thing that has tried to supply one.

---

## 1. A test is the first consumer of your design

Mechanically a test is unremarkable: arrange some state, act on it, assert something about the result.

```csharp
[Test]
public void Rejects_a_container_taller_than_the_rack()
{
    var rack = new Rack(height: 2);
    var result = rack.CanHold(new Container(height: 3));
    Assert.That(result.Allowed, Is.False);
}
```

What makes it valuable is not the assertion. It is that writing it forced you to answer three questions about the code: how do I construct this thing, how do I invoke the behaviour, and how do I observe the outcome. Those are exactly the questions a *caller* asks, and a class that answers them awkwardly is awkward for every caller — the test is simply the first one to say so out loud.

That gives you a diagnostic worth trusting. When a test is hard to write, read the difficulty rather than working around it:

| The test needs… | What the design is telling you |
|---|---|
| A database, to test a calculation | The rule is tangled with its storage |
| Six mocks to construct one class | The class has six responsibilities |
| To set a static before running | There is hidden shared state — and it will break under parallel tests |
| To wait, or to run at a specific time | Time is an undeclared dependency; inject a clock |
| Internals made `public` to assert on | You are testing implementation, not behaviour |

The last two are worth pausing on. Chapter 03 §1 used `IClock` as its example of a registration, and this is why that example exists: `DateTime.Now` inside a method is a dependency on the machine's clock that never appears in the constructor, so a test cannot control it, and behaviour that changes at midnight or on a leap day cannot be reproduced. Injecting it makes the dependency visible, which is the same move DI makes for everything else.

And **test behaviour, not implementation.** A test that asserts on private state or on the exact sequence of internal calls fails every time you refactor, even when nothing observable changed. That test does not protect the behaviour; it prevents the refactoring. Assert on what a caller could see.

---

## 2. Unit and integration differ by what you hold still

The distinction is usually taught as a matter of size, which is why it stays fuzzy. It is not about size. It is about **what you are deliberately holding still in order to learn something about the rest**.

A **unit test** holds everything still except one behaviour. No database, no clock, no network, no filesystem. It is fast — microseconds — and deterministic, so it can run on every save. In exchange, it can only tell you that a piece works *in the world you constructed for it*.

An **integration test** deliberately stops holding things still. It runs real components together against a real database, a real pipeline, real configuration. It is slower and less precise about *where* something broke, and it is the only kind of test that can tell you the pieces fit.

That last point is the one worth internalising, because it decides what you write:

> **Unit tests verify the parts. Integration tests verify the wiring. A bug in the wiring is invisible to every unit test you could possibly write.**

This handbook has already produced three bugs of exactly that shape. `UseAuthorization` missing from the pipeline, so `[Authorize]` is decoration (ch. 06 §2). A middleware registered in the wrong order (ch. 01). A captive dependency that only surfaces on the second request (ch. 03 §3). Every one of those is a *composition* failure — each part is correct in isolation, and a hundred percent unit coverage says nothing about any of them.

So the useful shape of a suite is not a ratio anyone can hand you. It is: many fast unit tests for logic that has branches worth enumerating, and a deliberately small number of integration tests over the paths that would be catastrophic to get wrong — authentication, authorization, persistence, the two or three flows the business actually runs on.

---

## 3. The framework is a footnote — and here is the honest difference

*xUnit, NUnit or MSTest* is a question with a boring true answer: they are interchangeable, all three are maintained, all three run in CI, and no project has ever failed because of this choice. Pick the one the team already uses.

What actually differs is **test lifecycle**, and that is worth knowing because it is where surprises come from:

| | Instance per test | Setup | Parallelism |
|---|---|---|---|
| **xUnit** | New instance per test method | Constructor / `IDisposable`; `IClassFixture<T>` to share | Test collections in parallel **by default** |
| **NUnit** | One instance per fixture, reused | `[SetUp]` / `[TearDown]`, `[OneTimeSetUp]` | Opt-in via `[Parallelizable]` |
| **MSTest** | New instance per test method | `[TestInitialize]` / `[TestCleanup]` | Opt-in |

xUnit's "new instance per test" is a deliberate stance — it makes shared state between tests structurally difficult rather than merely discouraged. NUnit's reused fixture is more convenient and puts the discipline on you: a field mutated by one test is visible to the next, and with `[Parallelizable]` on, visible *concurrently*.

The parameterised-test syntax differs and means the same thing — xUnit's `[Theory]` with `[InlineData]`, NUnit's `[TestCase]`, MSTest's `[DataRow]`. Whichever you use, the point is the same: one test method, several cases, one name per behaviour rather than five near-identical methods.

**In GROS** the stack is NUnit with Moq, which is a perfectly ordinary production choice and worth saying plainly in an interview: the framework was chosen once, it works, and the interesting decisions were all about what to isolate.

---

## 4. Test doubles: the seam is an interface you already declared

A **test double** is any stand-in for a real dependency. The vocabulary is over-elaborated in most writing about it; two distinctions carry their weight:

- A **fake** has a working implementation that is unsuitable for production — an in-memory repository, a dictionary-backed cache. It behaves.
- A **mock** is configured per test to return values and to record calls, so you can assert that something *was called*. It verifies.

```csharp
var repo = new Mock<IRackRepository>();
repo.Setup(r => r.GetAsync(12, It.IsAny<CancellationToken>()))
    .ReturnsAsync(new Rack(height: 2));

var sut = new PlacementService(repo.Object, new FixedClock(...));
```

Chapter 03 is doing the work here. `PlacementService` takes `IRackRepository` because it declared what it needs, and the test supplies a different implementation. That is the *only* reason this is possible, and it is why "is this testable" and "are its dependencies declared" are the same question.

Two rules keep doubles from becoming the problem they were meant to solve:

**Mock what you own.** Your own interfaces are contracts you defined and can keep honest. Mocking a third-party client — or worse, `DbContext` — means encoding your *assumptions* about someone else's behaviour, and the test then passes forever regardless of whether those assumptions are true. If you need a boundary you do not own to be substitutable, wrap it in an interface you do own and test the wrapper against the real thing.

**Prefer a fake to a mock when you can.** A test asserting `repo.Verify(r => r.GetAsync(12, ...), Times.Once)` is asserting on *how* the code works, which brings back §1's refactoring problem. A fake lets you assert on the outcome instead. Reach for `Verify` when the call genuinely is the observable behaviour — an email was sent, an event was published, a payment was taken.

And the failure state worth naming: a test that sets up five mocks and then asserts on those mocks is a test of your mock configuration. It will stay green through a total rewrite of the logic underneath it.

---

## 5. Integration testing ASP.NET Core: the wiring is the point

`WebApplicationFactory<T>` starts your **actual application** — real `Program.cs`, real DI container, real middleware pipeline, real endpoint routing — over an in-memory transport with no sockets and no port.

```csharp
public class RackApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    public RackApiTests(WebApplicationFactory<Program> factory) =>
        _client = factory.WithWebHostBuilder(b => b.ConfigureServices(s =>
        {
            s.RemoveAll<IRackRepository>();
            s.AddSingleton<IRackRepository, InMemoryRackRepository>();
        })).CreateClient();

    [Fact]
    public async Task Unauthenticated_request_is_rejected()
    {
        var response = await _client.GetAsync("/api/racks/12");
        Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
    }
}
```

That last test is the entire argument for this section. It is the only kind of test that catches chapter 06's silent failure — `[Authorize]` present, `UseAuthorization()` absent, endpoint wide open. No unit test can catch it, because there is no unit involved: the bug lives in the composition.

Two notes that matter in practice. The service replacement above works because chapter 03's rule still holds — **last registration wins**, so `RemoveAll` plus a re-registration substitutes a dependency without touching `Program.cs`. And this is precisely why `Program.cs` should stay a composition root: the more logic it contains, the less of your app an integration test is actually exercising.

Configuration is a layer here too (ch. 02) — a test host can add its own provider on top of the stack, which is how you point the test at a different database or turn a feature flag on without editing `appsettings.json`.

---

## 6. Testing EF Core, and why the in-memory provider lies

The EF Core in-memory provider looks like the obvious answer and is a trap, for one reason: **it is not a relational database**. It is a dictionary that speaks `DbSet`.

What it does not do, all of which your production database does:

- **No referential integrity, no unique constraints, no check constraints.** A test inserting a duplicate or an orphan passes. Production rejects it.
- **No SQL translation.** LINQ runs with LINQ-to-Objects semantics, so a query that *cannot be translated* — chapter 04 §3's throw — passes cleanly in the test and fails on the first real request.
- **Different semantics for almost everything else** — case sensitivity, collation, `null` ordering, concurrency tokens, transactions.

So it tests that your code compiles and that your mapper roughly works, and it silently green-lights the entire class of bug chapter 04 was about. Microsoft's own guidance says the same.

The two honest options:

- **SQLite in-memory** — a real relational engine, fast, enforces constraints, and catches most translation problems. Not your production dialect, so provider-specific behaviour still slips through.
- **The real database in a container** — Testcontainers, or a Compose service in CI. Slower to start, and it is the only thing that tells you your PostgreSQL query actually works on PostgreSQL. For anything with meaningful queries, this is the one worth paying for.

Which brings back chapter 04's argument from the other side: the reason a repository interface is a useful seam is not database independence — `IQueryable` leaks the provider straight through it — but that it gives you *somewhere to stand* when you want a fake for a test that is genuinely about something else.

---

## 7. TDD is design pressure, and here is where it does not fit

Red-green-refactor, stated properly:

1. **Red** — write a failing test for a behaviour that does not exist. Run it. *Watch it fail.* (§9 is about why this step is not ceremony.)
2. **Green** — the least code that passes. Ugly is fine.
3. **Refactor** — clean it up with the test holding the behaviour still.

The value is almost entirely in step 1, and not for the reason usually given. A test written before the code is **a consumer of an API that does not exist yet**, so you are designing the call site before the implementation — from the outside in. Code written that way comes out with declared dependencies and observable outcomes, because a version without those properties would have been impossible to write the test for. TDD produces the seams chapter 03 argues for without anyone having to mandate them.

It also changes what "done" means. The test is written when the behaviour is specified, not after, so there is no backlog of untested code and no negotiation about whether to go back for it.

**And it does not always fit.** Three cases where test-first is the wrong tool, stated plainly because pretending otherwise is how the practice loses credibility:

- **You do not know the shape yet.** Exploratory work, a spike against an unfamiliar API, a UI you are still designing. Writing tests against a design you are about to throw away is waste. Explore, then delete the spike and rebuild it test-first.
- **The feedback loop is not the bottleneck.** For a pure function you can verify by reading, the test is still worth having — but writing it first buys nothing.
- **Correctness is empirical rather than specified.** This is the honest one, and it is the shape of the optimizer work: you do not know the right answer in advance. You tune heuristics, measure results against real data, and decide the output is good. Test-first is meaningless there, because the assertion does not exist until after the experiment. What you do instead is **pin the result once you have it** — characterisation tests over known inputs, so the next tuning pass cannot silently regress a case you already got right. That is not TDD, it is a regression harness, and confusing the two is how people end up believing TDD failed them.

The general rule: use TDD where the behaviour is *specified* and you are designing an interface. Use tests-after where the behaviour is *discovered*. Both end with tests; only one of them starts there.

---

## 8. Logging is the same seam, left running

Testing and logging are the same instinct pointed at different environments: make behaviour inspectable by something other than a human watching it happen. A test inspects behaviour on your machine. A log inspects it in production, where you cannot attach a debugger and cannot reproduce the input.

`ILogger<T>` is injected, which by now should read as a statement about testability rather than convenience — it is a declared dependency, so it can be substituted, and a component's diagnostic output is as replaceable as its repository.

**Log structured, not stringly:**

```csharp
_logger.LogInformation("Rack {RackId} rejected container {ContainerId}: {Reason}",
                       rackId, containerId, reason);          // yes

_logger.LogInformation($"Rack {rackId} rejected container {containerId}");   // no
```

They render the same in a console and are completely different downstream. The first preserves `RackId` as a *field* — queryable, aggregatable, filterable. The second has already collapsed into a string, and the information is gone before it leaves the process. Interpolation also formats the string even when the level is disabled, so you pay for a message nobody will read.

**Levels are a contract, not a mood.** `Information` is what you can afford to emit for every request in production. `Debug` and `Trace` are for when something is wrong and you turn them up — and the level is configuration (ch. 02), so turning them up is a settings change rather than a deploy. `Warning` means something recovered. `Error` means an operation failed. `Critical` means the process is in trouble. A codebase where everything is `Information` has no levels; it has one level with extra syntax.

**Asserting on logs is usually a smell**, for §4's reason: it is testing how something works. The exception is when the log *is* the observable behaviour — an audit trail, a compliance record, a security event. Then it is output, and it deserves a test like any other output.

**Boundary:** this section is deliberately about logging within one process. The moment there is more than one, the interesting question stops being "what did this service record" and becomes "what happened to *this request* across five services" — correlation IDs, trace context, aggregation. That is a distributed-systems problem, and it is chapter 09's (question 35).

---

## 9. The detail most people miss: a test you have never seen fail is not a test

Every test makes a claim: *if this behaviour breaks, I will go red.* Nothing verifies that claim except watching it happen. Until you have, you have a method that runs and passes, and passing is also what a broken test does.

This is what the "red" step in red-green-refactor is actually for. Not ritual — **evidence that the test is wired to the thing it claims to be testing.** Writing the test first gets you that evidence for free, because the only way to reach green is through a failure you observed.

The tests that were never seen failing go wrong in ways that all look fine in the report:

- **An assertion that cannot fail.** `Assert.NotNull(result)` on a method that cannot return null. `Assert.True(true)` left behind after a debugging session. Green forever, meaning nothing.
- **A test that asserts on its own mocks** (§4) — survives a rewrite of the logic underneath it.
- **A test that never runs.** `[Ignore]`/`[Skip]` added during a bad week, a fixture that silently fails to discover, a filter in CI that excludes a whole project. The suite is green because those tests are absent.
- **A build that does not fail on failures.** A test step whose exit code is swallowed, or a pipeline where the test job is `continue-on-error`. Everything is reported and nothing is enforced.
- **Flaky tests, which are worse than no tests.** A test that fails one run in twenty trains everyone to re-run the pipeline, and once "just re-run it" is the habit, a *real* failure gets re-run too. Flakiness is almost always shared state, real time, or ordering assumptions under parallel execution — §1's table and §3's lifecycle differences. Fix it or delete it; quarantining it indefinitely is deleting it with extra steps.

The habit that catches most of this costs nothing: **when you write a test after the code, break the code on purpose and watch it go red.** Comment out the line, invert the condition, return the wrong constant. If it stays green, you have learned something important immediately rather than during an incident. (Mutation testing — Stryker.NET — automates exactly this, and is worth a run against a suite you are unsure of.)

And the corollary for coverage numbers: coverage measures which lines *executed*, not which behaviours are *protected*. A suite that runs every line and asserts almost nothing reports the same number as one that would catch a regression. Coverage is a decent way to find code nobody tested, and a terrible way to argue that code is tested.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| One behaviour change reddens dozens of tests | Tests assert on implementation, not behaviour — or every one of them constructs the whole world |
| A test needs six mocks to build one class | The class has six responsibilities; the test is reporting a design problem |
| Tests pass alone and fail together, or fail in a different order | Shared state — a static, a reused fixture field, one database. Sharpened by parallel execution |
| Tests fail at midnight, month end, or on a leap day | Real time is an undeclared dependency; inject a clock |
| Everything is green and production is broken | A composition bug — no unit test can see the wiring, §2 |
| An EF query works in tests and throws in production | The in-memory provider does not translate to SQL, §6 |
| A duplicate or orphan row is accepted in tests | The in-memory provider enforces no constraints, §6 |
| A test stayed green through a rewrite of the code it covers | It asserts on mocks, §4 |
| CI is green with failing tests | Skipped tests, undiscovered fixtures, or a swallowed exit code, §9 |
| "Just re-run it" is the team's reflex | Flakiness has trained everyone to ignore red, §9 |
| High coverage, low confidence | Coverage counts executed lines, not protected behaviours |
| A structured log has no queryable fields | String interpolation collapsed them before they left the process, §8 |

---

## Check yourself

Answer out loud, without looking:

1. A class needs six mocks to instantiate in a test. What is the test telling you, and why is adding a seventh the wrong response?
2. Give a bug that a hundred percent unit coverage cannot possibly catch, and say why an integration test can.
3. The EF Core in-memory provider passes a query that throws in production. Explain the mechanism, not just the fact.
4. What is the "red" step in red-green-refactor actually verifying — and what do you do instead when you wrote the code first?
5. Name a case where test-first is the wrong approach, and say what you do in its place without giving up regression safety.

---

## Questions this chapter answers

Five of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch07):

| # | As originally asked | Answered by section |
|---|---|---|
| 41 | How do you write unit tests? | §1, §4 — a test is the first caller; doubles work because dependencies are declared |
| 42 | xUnit, NUnit, and MSTest | §3 — interchangeable; lifecycle and parallelism are the only real differences |
| 43 | Integration tests vs unit tests | §2, §5–6 — what you hold still, and why wiring bugs are invisible to units |
| 44 | Logging API for troubleshooting | §8 — the same seam, left running in production |
| 45 | What is the TDD approach? | §7 — design pressure from the outside in, and where it does not fit |

Chapter 08 takes the suite from §9 and asks what runs it, and what happens after it goes green: 13, 46, 47, 49, 51, 52, 54, 55.

## Next

→ [`08-build-and-ship.md`](08-build-and-ship.md) — §9 argued that a suite nothing enforces is decoration. The next chapter is the machinery that enforces it, and everything that happens between a green build and a running process.
