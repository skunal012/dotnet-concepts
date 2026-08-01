# EF Core's mental model

> **The one idea:** a `DbContext` is a unit of work wrapped around a change tracker. You load objects into it, mutate them as plain C#, and `SaveChanges` works out what actually changed and writes the minimum SQL to make the database agree — in one transaction.

▶ **Widget:** [`widgets/04-ef-cores-mental-model.html`](../widgets/04-ef-cores-mental-model.html) — move entities through the tracker and watch which SQL each state generates. Open it in any browser; no build step.

---

## Why this concept exists

Objects are a graph of references. Tables are sets of rows joined by keys. Something has to translate, and before EF you were that something.

ADO.NET gave you a connection, a command and a reader. You wrote the SQL, mapped columns to properties by ordinal or by name, and — this is the part that actually hurt — you tracked what had changed yourself. Every update was a hand-written `UPDATE` listing the columns you remembered to include. Forget one and the change silently disappeared. Include them all and you overwrote a concurrent edit you never saw. The bug was never in the SQL; it was in the bookkeeping around it.

EF's answer is to stop writing the `UPDATE`. Describe the mapping once, load objects, change them like ordinary objects, and let something else compute the difference.

That "something else" is the change tracker, and it is the whole chapter. You have met the `DbContext` three times already — as the middleware that could not hold one (ch. 01 §7), as the scoped service `IOptionsSnapshot` behaves like (ch. 02 §6), and as the captive dependency in every one of ch. 03's failure modes. It was the example. Now it is the subject, and its scoped lifetime stops being a rule to remember: a unit of work is *defined* by having a beginning and an end.

---

## 1. The change tracker is an identity map with receipts

When a query materialises an entity, the context does two things: it files the instance under its primary key, and it keeps a **snapshot** of the values it arrived with.

The identity map is the visible half:

```csharp
var a = await db.Racks.FirstAsync(r => r.Id == 12);
var b = await db.Racks.FirstAsync(r => r.Id == 12);   // a second round trip to the database

ReferenceEquals(a, b);   // true
```

Two queries, two round trips — but one object. The second query's rows are matched against the identity map by key, and the instance already tracked wins. This is why a context is a *unit of work* and not a cache: within it, "rack 12" is a single thing you can pass around and mutate, and everyone holding it sees the same mutation.

The snapshot is the invisible half, and it is what makes the rest work:

```csharp
rack.Height = 4;                     // plain property set. No EF call.
db.Entry(rack).State;                // Modified
```

You never assign the state. The tracker infers it by comparing the live object against the snapshot it took. `EntityState` is therefore a *derived* fact, not an instruction:

| State | Means | Becomes |
|---|---|---|
| `Added` | in the tracker, not in the database | `INSERT` |
| `Unchanged` | live values match the snapshot | nothing |
| `Modified` | at least one property differs from the snapshot | `UPDATE`, changed columns only |
| `Deleted` | marked for removal | `DELETE` |
| `Detached` | not tracked at all | nothing — the context has never heard of it |

Two consequences worth having in advance:

- **`db.Update(entity)` is usually wrong.** It exists for detached entities coming back from somewhere else — a web request, a message — where there is no snapshot to diff against, so EF has to assume everything changed and marks every property `Modified`. On an entity the context already tracks it is strictly worse than doing nothing, because it turns a two-column `UPDATE` into a full-row one and reintroduces the lost-update problem the tracker was built to avoid.
- **Detached is the default outside a context.** An entity serialised to JSON and posted back is a bag of values with no snapshot and no identity. Reattaching it is a decision, not a formality — you are asserting what the original values were.

---

## 2. `SaveChanges` is a diff, and it is already a transaction

One call does five things in order:

1. **Detect changes** — walk tracked entities, compare each against its snapshot.
2. **Build commands** — one `INSERT` / `UPDATE` / `DELETE` per changed entity.
3. **Order them** — by foreign-key dependency, so a parent inserts before its children. Your call order is irrelevant; the graph decides.
4. **Execute in a transaction** — all of it, or none of it.
5. **Accept changes** — everything surviving becomes `Unchanged`, snapshots are replaced, database-generated keys are read back into the objects.

Step 4 is the answer to *how do you handle transactions in EF Core*, and the answer is mostly **you already did**. A single `SaveChangesAsync()` is atomic without you writing anything. Wrapping one call in `BeginTransaction` adds nothing.

You need an explicit transaction in exactly three situations:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
// 1. two or more SaveChanges calls that must succeed or fail together
// 2. EF work that must be atomic with raw SQL or a Dapper query on the same connection
// 3. you need an isolation level other than the provider's default
await tx.CommitAsync(ct);
```

Two details that bite in production:

**Retrying execution strategies and manual transactions are mutually exclusive by default.** Turn on `EnableRetryOnFailure` — which you should, against any managed database — and EF refuses to let you open a transaction manually, because a retry would replay only part of it. The fix is to hand the whole unit to the strategy so the *transaction* is what gets retried:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    await db.SaveChangesAsync(ct);
    await otherWork(ct);
    await tx.CommitAsync(ct);
});
```

**The snapshot is also your concurrency check.** Add a `[Timestamp]` / `RowVersion` property and the generated `UPDATE` becomes `WHERE Id = @id AND RowVersion = @original`. Zero rows affected means someone else got there first, and EF turns that into `DbUpdateConcurrencyException`. Without a concurrency token, last write wins silently — which is the ADO.NET bug from the top of the chapter, still available to you.

---

## 3. `IQueryable` is an expression tree, not a collection

`db.Racks.Where(...)` returns nothing and touches no database. It builds an expression tree. The provider translates that tree to SQL only when something forces enumeration — `ToListAsync`, `FirstAsync`, `CountAsync`, `foreach`.

So this composes into **one** statement with a `WHERE`, an `ORDER BY` and a `LIMIT`:

```csharp
var q = db.Racks.Where(r => r.WarehouseId == id);
if (onlyTall) q = q.Where(r => r.Height > 2);
var page = await q.OrderBy(r => r.Code).Take(50).ToListAsync(ct);
```

And the single most expensive mistake in EF Core is crossing the boundary early:

```csharp
var racks = db.Racks.ToList().Where(r => r.Height > 2);   // SELECT * FROM racks — all of them
```

`ToList()` ends translation. Everything after it is LINQ-to-Objects running in your process over rows you already paid to transfer. The type is the tell: the moment an expression stops being `IQueryable<T>` and becomes `IEnumerable<T>`, you have told EF you are finished composing.

Since EF Core 3.0 the provider **throws** rather than quietly doing this for you when a `Where` cannot be translated — a deliberately breaking change, because silent client evaluation was shipping fine in development and melting under production data volumes.

**N+1** is the same mistake wearing a loop. One query for the parents, then a lazy-loaded navigation touched per row:

```csharp
foreach (var rack in await db.Racks.ToListAsync(ct))
    total += rack.Containers.Count;        // one query per rack
```

`Include` fixes it by joining; for several collection includes prefer `AsSplitQuery()`, because a single query with two collection joins multiplies rows cartesian-style. Better still, project — §4.

**The provider leaks through the interface.** Chapter 03 §6 claimed a repository does not buy you database independence; this is where that becomes concrete. On PostgreSQL, `Where(r => r.Code == code)` is case-sensitive where SQL Server's default collation is not, `EF.Functions.ILike` exists and has no SQL Server equivalent, and identifiers fold to lower case unless quoted. The C# reads identical across providers. The SQL, the plan and the result set do not.

---

## 4. Tracking is a cost you are allowed to decline

Every tracked entity costs a snapshot in memory, and every `SaveChanges` walks all of them comparing property by property. For a request that loads three entities and saves one, this is free. For a request that loads forty thousand rows to compute a number, you are paying for bookkeeping on a change set that will always be empty.

```csharp
var racks = await db.Racks.AsNoTracking().ToListAsync(ct);   // no snapshots, no identity map
```

Three tools, in increasing order of how much they save:

- **`AsNoTracking()`** — read-only queries. No snapshot, no identity map, so duplicate rows materialise as duplicate objects. Use `AsNoTrackingWithIdentityResolution()` when you need the graph deduplicated but still do not intend to write.
- **Projection** — `Select` into a DTO or anonymous type. Never tracked *and* only fetches the columns named, which is usually the larger win. A projection that reads three columns of a fifty-column table changes the query, not just the tracker.
- **`ExecuteUpdateAsync` / `ExecuteDeleteAsync`** — one SQL statement, no entities loaded at all. The catch is exactly what makes them fast: the tracker does not know they happened, so anything already tracked is now stale.

**In GROS:** the optimizer loads large rack and container sets to score placements and never writes a row on that path. Tracking them would mean tens of thousands of snapshots held for the life of the scope and re-scanned on every `DetectChanges` — pure cost against a change set guaranteed to be empty. The read paths are `AsNoTracking` or projections; the write path, which is small and does have invariants, keeps the tracker. That split is the useful shape: the tracker earns its keep where you mutate, and nowhere else.

---

## 5. The model is the source of truth; a migration is a diff of two snapshots

"Code-first versus database-first" is usually asked as a preference question. It is not one — they are two directions of the same mapping, and which you use is decided by which artifact you are allowed to change.

- **Code-first** — the C# model is authoritative. `dotnet ef migrations add` generates the schema change.
- **Database-first** — the schema is authoritative, usually because something else owns it. `dotnet ef dbcontext scaffold` generates the model, once, and you re-scaffold when the schema moves.

The detail that makes migrations stop being mysterious is **what the diff is taken against**. It is not the database. Every migration ships alongside `ModelSnapshot.cs`, a generated description of the model as of the last migration, and `migrations add` diffs *your current model against that file*:

```
current model  ──diff──▶  Up()   ──▶  new snapshot
                          Down()
```

Everything awkward about migrations follows from that one fact:

- Hand-edit a migration and the snapshot no longer matches it, so the *next* migration diffs against a lie and generates nonsense. Change the model and regenerate instead.
- Two developers adding migrations on separate branches both rewrite the snapshot file. The merge conflict is real, and resolving it by picking one side leaves the snapshot describing a model nobody has. Re-generate the second migration after merging.
- `migrations remove` exists precisely because deleting the migration file by hand leaves the snapshot ahead of it.

At runtime, `__EFMigrationsHistory` records which migrations have been applied — that is how `database update` knows where to resume, and it is the reason a migration is identified by name rather than by comparing to the live schema.

---

## 6. `dotnet ef` has to build your app, and that explains its whole design

The tooling needs your model, and the model only exists as the result of running `OnModelCreating` on a real `DbContext`. So `dotnet ef` compiles your project and instantiates the context at **design time**, in your machine's process, before it can generate anything.

Once you know that, the CLI's odd edges are all consequences:

- It needs a way to construct the context outside the app. It tries your host builder first — which is why `AddDbContext` in `Program.cs` is enough for most projects — and falls back to `IDesignTimeDbContextFactory<T>` when the host is not constructible (a class library, or a `Program.cs` doing real work on startup).
- `--project` and `--startup-project` are separate flags because the context and the host that configures it frequently live in different assemblies.
- Design-time connection strings come from your user secrets and environment, not the server's. Adding a migration talks to *your* database, or to none at all.
- A project that does not compile has no migrations. The error is a build error wearing a tooling error's clothes.

The four commands that matter:

```bash
dotnet ef migrations add AddRackHeight      # diff model against snapshot, emit Up/Down
dotnet ef migrations remove                 # undo the last add, snapshot included
dotnet ef database update                   # apply pending migrations to a database
dotnet ef migrations script --idempotent    # emit SQL guarded by history checks  ← the deploy artifact
```

That last one is the one to remember, and §8 is why.

---

## 7. Dapper: what the tracker costs, and when to stop paying

Dapper is a mapper, not an ORM. It takes SQL you wrote and a type, executes the command and fills the type from the columns. There is no model, no translation, no identity map and no change tracker — which means the EF-versus-Dapper question is not really about two libraries. It is the question *do I want a change tracker on this code path*, and you already have the framework to answer it.

**Keep EF** where there is a write model with invariants: you load a graph, enforce rules against objects, and commit one unit of work. That is the tracker's entire job, and hand-rolling it is how the bugs in "Why this concept exists" come back.

**Reach for Dapper** where the tracker is dead weight and the SQL is the point: reporting and analytics, window functions and CTEs and recursive queries, a hot read path where materialising entities is measurable, or a bulk operation.

Two honest qualifications, because the comparison is usually oversold:

- Most of the read-side gap closes with `AsNoTracking` and projections (§4). Reaching for Dapper because "EF is slow" without having tried a projection is skipping the cheap fix.
- The cost is real but it is *your* cost, not the database's. Dapper does not produce a better query plan; it produces the plan for the SQL you wrote, which may be better or worse than the one EF generated.

Using both in one application is normal and not an admission of failure. They can share a connection and a transaction, so a Dapper read inside an EF transaction is a supported thing rather than a hack. The line worth defending in an interview: EF owns writes because writes have invariants; Dapper owns the reads that are really reports.

---

## 8. The detail most people miss: migrating on startup is a deployment decision

`db.Database.Migrate()` in `Program.cs` is the most natural line in the world to write. It works on your machine immediately, it works in CI, and it removes a manual step. It is also the point where schema changes stop being reviewed, and it fails in ways that are specific to production.

**It races across replicas.** Scale to three instances and all three boot at once, each calling `Migrate()`, each finding the same pending migration. Recent EF versions take a lock on the history table to make this survivable, and the outcome is usually that one applies while the others wait — but Microsoft's own guidance has never changed: applying migrations at startup from multiple instances is not a supported deployment strategy. "Usually" is carrying the entire system.

**There is no rollback.** A migration that fails halfway leaves the schema in a state no snapshot describes, and the app crash-loops on boot — so the thing that would let you diagnose it is the thing that will not start. `Down()` is generated but almost never tested, and it cannot restore dropped data regardless.

**It grants DDL rights forever.** The application's runtime account now needs permission to alter the schema, permanently, so that it can use it during the first ninety seconds after deployment.

**It inverts the deployment order.** The new schema arrives when the first *new* instance boots, while old instances are still serving traffic against it. Any destructive change — drop a column, rename one — breaks the running version during the rollout window. This is what expand/contract is for: add the new column, deploy code that writes both, backfill, then drop the old one in a later release. Auto-migration does not prevent this, but it does hide the moment where you would have thought about it.

And `EnsureCreated()` is not a lighter-weight `Migrate()`. It creates the schema directly from the model and writes **no** history rows, so a database created that way can never accept a migration afterwards. It belongs in throwaway test fixtures and nowhere else.

**What to do instead:** generate `dotnet ef migrations script --idempotent` as a build artifact and apply it as an explicit step — an init container, a release stage, a job that runs to completion before the new version starts. It is reviewable in a pull request, it is the same script in staging and production, and it fails *before* anything is serving traffic rather than during. Keep `Migrate()` for local development and integration tests, where a race is impossible and a broken database is a `docker compose down` away.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| An `UPDATE` writes every column, not the one you changed | `db.Update(entity)` on a tracked entity — no snapshot diff, so everything is `Modified` |
| A change to a loaded object never reaches the database | The entity is detached, or `SaveChanges` was never called on *that* context |
| `The instance of entity type 'X' cannot be tracked because another instance with the same key is already tracked` | Two objects, one key. Usually a detached entity reattached while the identity map already holds it |
| One query in development, hundreds in production logs | N+1 — a navigation touched inside a loop; needs `Include` or a projection |
| A query returns far more rows than the result needs | `ToList()` before the `Where` — translation ended, the rest ran in memory |
| `The LINQ expression could not be translated` | A C# method with no SQL equivalent inside a translated clause. Since 3.0 this throws instead of silently degrading |
| Row counts explode when two collections are included | Cartesian product from multiple collection joins — `AsSplitQuery()` |
| Memory and CPU climb with result-set size on a read-only endpoint | Tracked entities: a snapshot each, re-scanned on every `DetectChanges` |
| `DbUpdateConcurrencyException` | The concurrency token in the `WHERE` clause did not match — someone else wrote first. Working as designed |
| `The configured execution strategy does not support user-initiated transactions` | Retry-on-failure plus a manual transaction; wrap the unit in `CreateExecutionStrategy().ExecuteAsync` |
| The next migration generates changes you already applied | The model snapshot no longer matches the migrations — usually a hand-edited or hand-deleted migration |
| App crash-loops after deploy with a half-applied schema | `Migrate()` on startup failed partway, or raced another replica |
| A database created for tests will not accept migrations | `EnsureCreated()` writes no `__EFMigrationsHistory` rows |
| Data written by `ExecuteUpdate` is not visible on tracked entities | The statement bypassed the tracker; the loaded objects are stale |

---

## Check yourself

Answer out loud, without looking:

1. You load an entity, change one property, and call `SaveChanges`. Name every piece of state EF needed in order to produce that `UPDATE`, and say where each one came from.
2. Why is wrapping a single `SaveChangesAsync()` in `BeginTransaction` pointless, and what are the situations where it stops being pointless?
3. A migration is a diff. A diff between *what* and *what* — and what breaks if those two things disagree?
4. Why does `dotnet ef migrations add` need your project to compile, and why does it read *your* connection string rather than production's?
5. `Migrate()` on startup works perfectly for a year, then the service scales to three replicas. Name three separate things that are now wrong, only one of which is the race.

---

## Questions this chapter answers

Eight of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch04):

| # | As originally asked | Answered by section |
|---|---|---|
| 28 | What is EF Core and how do you use it? | §1, §3 — a change tracker plus a query translator; the rest is those two |
| 29 | How do you handle migrations in EF Core? | §5 — a diff against the model snapshot, not against the database |
| 56 | How do you work with databases using EF Core? | §3–4 — compose an `IQueryable`, decide whether you want tracking |
| 57 | What is the purpose of the `DbContext`? | §1 — it owns the identity map, the snapshots and the pending change set |
| 58 | Code-First and Database-First approaches | §5 — two directions of one mapping, decided by which artifact you own |
| 59 | How do you handle database transactions? | §2 — `SaveChanges` is already one; the three cases where you need your own |
| 60 | Dapper as an alternative to EF | §7 — the question is whether that path wants a change tracker |
| 72 | Explain the `dotnet ef` CLI tool | §6 — it builds and instantiates your context at design time, and everything follows |

Chapter 05 takes the thread this one kept pulling on — `await`, and what a thread is actually doing while a query runs: 30, 36–40, 95–98.

## Next

→ [`05-async-threads-and-memory.md`](05-async-threads-and-memory.md) — every database call in this chapter was `await`ed, and §4 was a memory argument in disguise. The next chapter is what those two words are really doing.
