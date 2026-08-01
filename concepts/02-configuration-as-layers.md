# Configuration as layers

> **The one idea:** configuration is not a file. It is a single flat dictionary of string keys, assembled by stacking ordered providers, where the last provider to supply a key wins.

▶ **Widget:** [`widgets/02-configuration-as-layers.html`](../widgets/02-configuration-as-layers.html) — resolve a key down through the provider stack and see which layer claims it. Open it in any browser; no build step.

---

## Why this concept exists

Old ASP.NET had `web.config`: one XML file, and a build-time XDT transform (`Web.Release.config`) that rewrote it per environment. That design has a fatal property — **the artifact differs per environment**. You build a Release package for staging, then build again for production, and the thing you tested is not the thing you shipped.

It also assumed the file lived on disk next to the app, which stops being true the moment your settings come from a container's environment, a Key Vault, or a Kubernetes secret.

.NET Core inverted it. Instead of one file transformed at build time, you get **many sources merged at run time**, in an order you control. One binary, one image, one published folder — pointed at different values.

This is the same shape as chapter 01: an ordered chain where position determines behaviour. There the ordering decided *who sees the request first*. Here it decides *whose value wins*.

---

## 1. There is no hierarchy — everything is flattened

`appsettings.json` looks nested. It isn't, by the time anything reads it. Every JSON path is flattened into a single string key with `:` separators.

```json
{
  "Optimizer": {
    "MaxParallelism": 4,
    "Solver": { "TimeLimitSeconds": 30 }
  }
}
```

becomes exactly two entries:

```
Optimizer:MaxParallelism           = "4"
Optimizer:Solver:TimeLimitSeconds  = "30"
```

Two things follow immediately, and both matter later:

- **Every value is a string.** `4` in JSON is `"4"` in the dictionary. Typing happens at bind time, not read time.
- **Arrays are flattened by index.** `[ "FirstFit", "BestFit", "Genetic" ]` under `Solver:Strategies` becomes `Solver:Strategies:0`, `:1`, `:2`. There is no array in the dictionary — only keys that happen to end in digits.

Keys are compared **case-insensitively**. `GetSection("optimizer")` finds `Optimizer`.

And `GetConnectionString("Gros")` is not a special feature — it is literally a lookup of the key `ConnectionStrings:Gros`.

---

## 2. The stack, and the order it is built in

`WebApplication.CreateBuilder(args)` has already assembled a stack for you. Later entries override earlier ones:

```csharp
var builder = WebApplication.CreateBuilder(args);
// providers, in override order (last wins):
//   1. appsettings.json
//   2. appsettings.{Environment}.json
//   3. user secrets                    ← Development only
//   4. environment variables
//   5. command-line arguments
```

Reading is the mechanism in one line: `configuration["Key"]` walks the provider list **in reverse** and returns the first hit. Nothing is merged in advance; there is no winner computed at startup. The precedence *is* the reverse iteration.

`builder.Configuration` is itself an `IConfigurationBuilder`, so you extend the stack by appending to it — and appending means winning:

```csharp
builder.Configuration.AddJsonFile("optimizer.overrides.json", optional: true);
// registered after env vars → this file now beats them
```

That is the whole answer to "how do configuration builders work". They are not a separate system; they are the list, and you are pushing onto it.

### The bootstrap problem, and how it is solved

Step 2 needs to know the environment name in order to pick the file. But the environment name is itself configuration.

So the host builds a **small config stack first** — command-line arguments plus environment variables prefixed `DOTNET_` and `ASPNETCORE_` — reads `ASPNETCORE_ENVIRONMENT` out of it, and only then builds the application stack whose second layer it now knows the name of.

This is why `ASPNETCORE_ENVIRONMENT` can only come from an environment variable or the command line. Setting it inside `appsettings.json` cannot work: the file that would tell you which file to load is the one you are trying to choose.

---

## 3. Environment variables, and why `__` not `:`

`:` is not a legal character in an environment variable name on Linux — you cannot `export Optimizer:MaxParallelism=8`. So the environment variable provider maps a **double underscore** to `:`:

```bash
Optimizer__Solver__TimeLimitSeconds=120
ConnectionStrings__Gros="Host=db;Database=gros"
```

This is the single most useful fact in the chapter, because it is what makes one container image deployable anywhere. The image ships with `appsettings.json` as sane defaults; the compose file or Kubernetes manifest supplies the per-environment leaves as env vars, which sit above the files in the stack.

**In GROS:** the optimizer image is built once. The database connection string, the gRPC service token, the solver time limit, and the auto-migrate-on-startup flag are all environment variables per deployment. Nothing environment-specific is baked into the artifact, which is the property `web.config` transforms could never give you.

A single `_` means nothing special — it is just a character in the key. `Optimizer_MaxParallelism` sets a key with an underscore in its name and silently fails to override anything.

---

## 4. Command-line arguments are the top of the stack

Registered last, so they beat everything — which is exactly what you want from an operator's escape hatch.

```bash
dotnet run --Optimizer:Solver:TimeLimitSeconds=5 --environment Staging
```

Three accepted forms: `--Key=value`, `--Key value`, and `/Key=value`. Note that `:` works fine here — the shell restriction was on variable *names*, not on argument text.

Short aliases are not magic either; they are a dictionary you supply:

```csharp
builder.Configuration.AddCommandLine(args, new Dictionary<string, string>
{
    ["-t"] = "Optimizer:Solver:TimeLimitSeconds"
});
```

And the reason `args` is passed to `CreateBuilder(args)` at all is this provider. Drop the argument and command-line configuration silently stops existing.

---

## 5. User secrets are not encrypted — they are just outside the repo

`dotnet user-secrets` gets described as secure storage. It is not.

```bash
dotnet user-secrets init          # writes a <UserSecretsId> GUID into the .csproj
dotnet user-secrets set "ConnectionStrings:Gros" "Host=localhost;Password=dev"
```

That value lands in plaintext at `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json` (`%APPDATA%\Microsoft\UserSecrets\...` on Windows). No encryption, no key management. The entire security property is **it is not inside your working tree**, so it cannot be committed by accident.

Which explains its two hard limits, both derivable:

- It is registered **only when the environment is Development**, because it is keyed by a GUID in your project file and read from your home directory — neither of which exists meaningfully on a server.
- It is per-developer, not shared. That is a feature, not a gap.

Production secrets are the same problem solved by a different provider slotted into the same stack: environment variables for simple cases, Azure Key Vault (`AddAzureKeyVault`) when you need rotation and audit. The stack does not care where a layer's values come from — that is the point of the provider abstraction.

---

## 6. `IOptions<T>` — where the stack meets DI

The dictionary is stringly-typed, which is fine for the framework and miserable for your code. `IOptions<T>` binds a section to a class once, then injects it.

```csharp
builder.Services.AddOptions<OptimizerOptions>()
    .Bind(builder.Configuration.GetSection("Optimizer"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

The binder matches **public settable properties** to keys, case-insensitively, converting the strings as it goes. A key with no matching property is ignored. A property with no matching key keeps its default.

That second rule is the trap: misspell the section name and you get a fully-populated object of default values, no exception, no log line. `MaxParallelism` is `0`, the optimizer does nothing, and everything reports healthy. `ValidateOnStart()` converts that into a startup crash, which is the correct behaviour for a misconfigured deployment — fail loudly at boot, not quietly at 3am.

Three interfaces, and the difference between them is a lifetime question:

| Interface | Lifetime | Reflects later changes? | Use when |
|---|---|---|---|
| `IOptions<T>` | singleton | **no** — computed once, on first resolve | value is fixed for the process |
| `IOptionsSnapshot<T>` | scoped | yes — recomputed per request | per-request consistency, may change between requests |
| `IOptionsMonitor<T>` | singleton | yes — `CurrentValue` plus `OnChange` callbacks | a singleton that must observe changes |

The rule that generates that table: a singleton resolves its dependencies once, so any reload-aware option consumed by a singleton has to be `IOptionsMonitor<T>`. Injecting `IOptionsSnapshot<T>` into a singleton is the captive-dependency bug from chapter 01 wearing a different hat.

---

## 7. `global.json` — the same idea, one clock earlier

Everything above resolves at run time. `global.json` is the same layering logic applied at **build** time, to the SDK itself.

```json
{
  "sdk": { "version": "8.0.400", "rollForward": "latestFeature" }
}
```

The `dotnet` host walks **up** from the current directory looking for the nearest `global.json`, and uses the first one it finds — directory proximity is the precedence rule, exactly as provider order is at run time. Nothing found means "use the newest installed SDK".

`rollForward` is the interesting field, because it decides how strict the pin is: `latestPatch` (default), `latestFeature`, `latestMajor`, or `disable` for an exact match. `disable` on a build agent that installed a newer SDK is a hard failure, which is sometimes what you want and usually is not.

One distinction worth being precise about in an interview: `global.json` pins the **SDK** — the build tooling. It says nothing about which runtime your app targets; `<TargetFramework>` does that. An SDK can build for older frameworks, so pinning the SDK is about reproducible builds, not about runtime behaviour.

---

## 8. The detail most people miss: layers merge per key, not per section

Because the stack is a flat dictionary and lookups happen per key, **you cannot override a section**. You override leaves.

Base file:

```json
{ "Solver": { "Strategies": [ "FirstFit", "BestFit", "Genetic" ] } }
```

Environment file, trying to cut it down to two:

```json
{ "Solver": { "Strategies": [ "FirstFit", "BestFit" ] } }
```

What the app actually sees is **three strategies**. The override wrote `Strategies:0` and `Strategies:1`. Nothing wrote `Strategies:2`, so the base value survives — and `Genetic` runs in production because no provider ever removed a key it did not know existed.

Same mechanism, same surprise, for objects: an environment file supplying two properties of a five-property section leaves the other three at their base values. That is usually what you want. With arrays it almost never is.

The workarounds all follow from understanding the cause: keep environment-specific arrays as a single delimited string and split it yourself, override the array in one place only, or set the trailing indices to `null` explicitly. There is no `clear this section` operation, because there is no section — there are only keys.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| Works locally, wrong value in the container | env var used `:` (or a single `_`) instead of `__` |
| `appsettings.Development.json` loads on Windows, ignored on Linux | Linux file paths are case-sensitive; the file name must match `ASPNETCORE_ENVIRONMENT` exactly, even though *keys* are case-insensitive |
| Shortened an array in the environment file, the old trailing item is still there | providers merge per leaf key — index `:2` was never overridden |
| Options object is all defaults, no error anywhere | section name typo; the binder ignores what it cannot match. No `ValidateOnStart()` |
| `ASPNETCORE_ENVIRONMENT` set in `appsettings.json` has no effect | environment is resolved by the bootstrap stack *before* the file layer is chosen |
| Edited `appsettings.json`, running app keeps the old value | `IOptions<T>` is computed once; and `reloadOnChange` file watching is unreliable on mounted ConfigMaps/volumes, which swap symlinks rather than writing the file |
| A `--flag` on the command line is ignored | `args` was not passed to `CreateBuilder(args)` |
| Value set in the right file, still overridden | something was registered later in the stack; last provider wins |
| Build fails only on the CI agent | `global.json` pins an SDK the agent does not have, with `rollForward` too strict |

---

## Check yourself

Answer out loud, without looking:

1. Why can `ASPNETCORE_ENVIRONMENT` not be set in `appsettings.json`, in terms of the order things are built?
2. You remove one entry from a three-item array in `appsettings.Production.json` and it still runs in production. Why is that the *expected* result of the mechanism?
3. What exactly does `dotnet user-secrets` protect you from, and what does it not protect you from at all?
4. A singleton service needs to see a configuration change without a restart. Which options interface, and why do the other two fail?
5. Your `Options` object binds to nothing but the app starts cleanly and reports healthy. What is missing, and why is silence the default behaviour?

---

## Questions this chapter answers

Eight of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch02):

| # | As originally asked | Answered by section |
|---|---|---|
| 9 | Purpose of the `global.json` file | §7 — the same precedence idea, resolved at build time by directory proximity |
| 20 | Use and configuration of `appsettings.json` | §1–2 — the base layer, and why it is flat |
| 23 | What are environment variables and how do they work | §3 — the override layer, and the `__` mapping that makes it usable |
| 70 | Managing user secrets in development | §5 — a Development-only provider whose only property is "outside the repo" |
| 75 | Managing configuration for multiple environments | §2 — one artifact, environment selects a layer name |
| 76 | The `IOptions` pattern | §6 — binding the flat stack into typed objects, and the three lifetimes |
| 78 | How Configuration Builders are used | §2 — the builder *is* the provider list; appending means winning |
| 79 | Reading command-line arguments | §4 — the topmost layer, and why `args` must reach `CreateBuilder` |

Chapter 12 covers the adjacent mechanical question this one deliberately leaves out: 8 (how multiple SDKs install side by side). `global.json` is here because pinning is a layering concept; side-by-side installation is a fact about the installer.

## Next

→ [`03-di-and-service-lifetimes.md`](03-di-and-service-lifetimes.md) — `IOptions<T>` was already a lifetime question. The next chapter is the model that generated the answer.
