# Distributed and real-time

> **The one idea:** once there is more than one process, every interaction is a contract plus a transport — and you choose the transport by answering two questions: *who initiates*, and *what survives a failure*.

▶ **Widget:** [`widgets/09-distributed-and-real-time.html`](../widgets/09-distributed-and-real-time.html) — send the same message over four transports, then kill the server mid-flight and see what is left. Open it in any browser; no build step.

---

## Why this concept exists

A method call gives you five guarantees you have never once thought about:

1. It happens.
2. It happens exactly once.
3. It happens now.
4. It either returns a value or throws — and those are the only two outcomes.
5. The compiler checked the signature before you shipped.

Cross a process boundary and **you lose all five.** The call may not arrive. It may arrive twice. It may arrive in four hundred milliseconds or not until the other side restarts. It may time out, which is a third outcome that is neither success nor failure (§9). And the signature is now a shared assumption that nothing verifies until run time.

Distributed systems is not a topic you add to an application. It is the bill for that trade, and every pattern in this chapter — retries, idempotency, correlation IDs, circuit breakers, sagas — is a line item on it. Which is also why chapter 08 §4 kept running into this: the moment horizontal scaling gave you three copies of the folder, they had to agree about something, and agreement across processes is exactly the problem here.

---

## 1. The two axes, and the four answers

Every transport is a position on two axes. Get these and the technology comparisons stop needing to be memorised.

**Who initiates?** The client asks, or the server tells. HTTP was built for the first and cannot do the second — a server has no way to speak to a client that has not asked.

**What survives a failure?** If the receiver is down when you send, is the message gone, or is it held somewhere until they come back?

| | Initiates | Survives a crash | Contract | Good for |
|---|---|---|---|---|
| **REST** | Client | No — the caller retries or loses it | Convention, described by OpenAPI | Public and edge APIs |
| **gRPC** | Client | No — same | A `.proto` compiled into types | Internal service-to-service |
| **SignalR** | **Server** | No — the connection dies with the instance | A hub method signature | Progress, notifications, live UI |
| **Queue** | Neither, in real time | **Yes — the broker holds it** | A message schema | Work that must not be lost |

The fourth row is a different kind of thing from the first three, and that is the point. REST, gRPC and SignalR all require both parties to be alive *at the same moment*. A queue removes that requirement, and almost everything people find surprising about queues follows from it (§4).

---

## 2. REST and gRPC are the same shape; the difference is the contract

Both are request/response, client-initiated, and neither survives the far end going away. Choosing between them is a question about **how the contract is expressed and enforced.**

**REST** is HTTP verbs and JSON. The contract is convention — a URL shape, a status code, a body — described after the fact by OpenAPI. Its advantages are real and often undersold: every language speaks it, every proxy and firewall understands it, and you can debug it with `curl` and read the payload with your eyes.

**gRPC** is HTTP/2 and protobuf. The contract is a `.proto` file *compiled into types on both sides*, which moves a class of error from run time to build time — rename a field and the client stops compiling rather than starting to receive nulls. The payload is binary and compact, HTTP/2 multiplexes many calls over one connection, and streaming is first-class in all four combinations: unary, server-streaming, client-streaming, and bidirectional.

The costs are equally concrete: you cannot read a protobuf frame in a network trace without tooling, and **browsers cannot speak gRPC directly** — they need gRPC-Web and a translating proxy, which gives back some of what you came for.

So the division that holds up in practice:

- **gRPC between your own services**, where both ends are yours, both are .NET or another supported stack, and the typed contract is a genuine safety gain.
- **REST at the edge**, where callers are browsers, third parties, or anything you do not control.

**In GROS** both are in play, and for exactly those reasons. Service-to-service calls are gRPC with a service token on the call (ch. 06 §3 — the principal is a service, not a person), while the client-facing API is REST with an OpenAPI document that generates the TypeScript client. That second half is the same idea as the `.proto`, arriving one step later: a generated typed client means a renamed field breaks the React build instead of producing `undefined` at run time.

---

## 3. SignalR: the server initiates, and the connection is the problem

HTTP cannot push. If a long-running job finishes and the server wants to tell a browser, its only options are for the client to keep asking (polling — wasteful and always late) or to hold a connection open.

SignalR is the second option, with the negotiation handled for you. It tries **WebSockets** first, falls back to **Server-Sent Events**, and finally to **long polling**, so a client behind a proxy that mangles WebSockets still works, more slowly. A **hub** then makes it look like RPC in both directions:

```csharp
public class OptimizationHub : Hub { }

// from anywhere in the app — a background worker, a handler:
await _hub.Clients.Group($"job-{jobId}")
          .SendAsync("Progress", new { jobId, percent, stage }, ct);
```

**In GROS** this is optimizer progress streaming, which is the archetypal fit: a run takes minutes, the user wants to see it advance, and polling for it would be both wasteful and stale.

The thing that makes SignalR harder to operate than it looks is that **a connection is state, held on one instance.** Chapter 08 §4 listed this as a thing that breaks at replica three, and here is the mechanism: client 1 is connected to server A, client 2 to server B. Code on server A publishes to a group — and only the clients connected to A receive it. Server B has never heard of the message. Nothing errors; half your users simply do not see the update.

The fix is a **backplane** — Redis, or the managed Azure SignalR Service — which every server publishes to and subscribes from, so a message reaches all instances and therefore all clients. The managed service goes further and holds the connections itself, which also removes the second problem: **without WebSockets, SignalR needs sticky sessions**, because the negotiate request and the subsequent transport requests must land on the same server.

And the discipline worth applying: SignalR is a stateful connection you now have to operate — reconnection, backplane, sticky routing, connection limits. If the data changes every thirty seconds and the user would not notice a delay, polling a REST endpoint is a smaller system that does the job.

---

## 4. Queues: nobody initiates in real time, and that is the feature

A queue puts a broker between sender and receiver. The sender writes a message and is done; the receiver reads it whenever it is running. That is **decoupling in time**, and it is the only thing in this chapter that answers "what survives a failure" with *the message does*.

What that buys, and each of these is something the other three transports cannot do:

- **The consumer can be down.** Deploys, restarts and crashes stop being lost work — they become latency.
- **Load is absorbed rather than propagated.** A traffic spike lengthens a queue instead of overwhelming a downstream service.
- **Competing consumers scale properly.** Ten instances reading one queue split the work, each message going to one of them. This is the clean answer to chapter 08 §4's scheduled job running three times: put the work in a queue and let one consumer take each item.

The costs are equally structural, and the first one is not optional:

**Delivery is at-least-once, so your consumer will see duplicates.** A broker that has delivered a message but not received the acknowledgement — because the consumer crashed, or the network dropped — has exactly one safe choice, which is to deliver it again. "Exactly-once delivery" is not a thing you can buy. Exactly-once *processing* is achievable, and you achieve it by making the handler **idempotent**: key the work by a message ID, record what you have processed, and make a repeat a no-op. Design for this from the first message, because retrofitting it means reconciling data that has already been double-counted.

**Ordering is weaker than you expect.** Most brokers guarantee order only within a partition or session key, and competing consumers break it by construction. If order matters, it has to be a property of the key you partition on.

**Failures need somewhere to go.** A message that always throws will be redelivered forever unless a **dead-letter queue** catches it after N attempts. Without one, a single poison message is an infinite loop that looks like a busy consumer.

**And writing to the database and publishing a message are not atomic.** Chapter 04 §2 established that `SaveChanges` is one transaction — but that transaction cannot include the broker. Commit then publish, and a crash in between loses the message; publish then commit, and you have announced something that did not happen. The **outbox pattern** is the standard resolution: write the message to a table *in the same transaction* as the data, and have a separate process read that table and publish. It converts an impossible atomic operation into an at-least-once one, which §4's idempotency requirement already told you how to handle.

---

## 5. Failure is the normal case, so retries are a design, not a reflex

In-process, an exception means something went wrong. Across a network, an exception frequently means *nothing is wrong* — a packet was dropped, a pod was being replaced, a connection was recycled. Transient failure is the steady state, and the built-in resilience handler is one line:

```csharp
builder.Services.AddHttpClient<RackClient>()
       .AddStandardResilienceHandler();   // timeout, retry with backoff+jitter, circuit breaker
```

The four pieces it composes, and what each one is actually for:

- **Timeout.** Without one, a hung dependency holds your thread and your request forever — chapter 05 §2's starvation arriving from outside. Every remote call needs a deadline.
- **Retry with exponential backoff *and jitter*.** Backoff stops you hammering a struggling service. Jitter — a random offset — stops every client retrying in lockstep, which is how a brief blip becomes a synchronised thundering herd (the same failure shape as chapter 08 §6's fleet restart).
- **Circuit breaker.** After enough failures it opens and fails fast without calling, then half-opens to test. The point is not to protect you; it is to *stop hammering something that is already down*, and to fail in milliseconds instead of at your timeout.
- **Bulkhead.** Cap the concurrent calls to one dependency so a slow one cannot consume every thread you have.

The rule that governs all of it: **only retry what is safe to retry.** A retried `GET` is free. A retried "charge this card" without an idempotency key charges twice. Which is §4's requirement arriving from the other direction — and §9's, which is the real reason.

And thread cancellation all the way through (ch. 05 §2): when the caller gives up, everything downstream should stop, or you are spending capacity computing an answer nobody will read.

---

## 6. Following one request across services

Chapter 07 §8 covered `ILogger<T>` within one process and explicitly deferred this. Here is the deferred part, and the reason it is a different problem: with one process, the question is *what did this service do*. With five, the question is **what happened to this request** — and no single service's log can answer it.

The mechanism is a **correlation identifier propagated across every hop.** .NET does most of this already: `System.Diagnostics.Activity` implements W3C Trace Context, `HttpClient` and the ASP.NET Core hosting layer automatically send and read the `traceparent` header, and each hop records a span with a parent. So a trace ID exists whether or not you have done anything — the work is making sure it reaches your logs and survives your own boundaries:

```csharp
using (_logger.BeginScope(new Dictionary<string, object> { ["JobId"] = jobId }))
{
    _logger.LogInformation("Placement run started with {RackCount} racks", racks.Count);
}
```

Two dependencies worth being explicit about:

**Structured logging is a prerequisite.** Chapter 07 §8 argued for message templates over interpolation; this is where that pays. You cannot filter a million lines by `TraceId` if the trace ID was concatenated into a string before it left the process.

**Queues break the automatic propagation.** HTTP carries `traceparent` for you; a message body does not, unless you put it there. Copy the trace context into message metadata on publish and restore it on consume, or your trace stops dead at the broker — which is exactly the boundary where you most need it.

The three signals, framed by what each answers:

- **Logs** — what happened, in detail, for one thing.
- **Metrics** — how often and how fast, aggregated. Cheap to keep, useless for a single case.
- **Traces** — where the time went across services, for one request.

Health checks (ch. 08 §6) belong in the same family: they are the machine-readable version of the same question, which is why the liveness-versus-readiness distinction mattered.

---

## 7. A service boundary is a cost — name what you are buying

Everything above is the price list. Microservices is the decision to pay it, so the question is never "microservices or monolith" — it is **what am I getting that is worth losing §1's five guarantees.**

Three answers are good ones:

- **Independent deploy cadence.** A team can ship without coordinating a release with four others. This is the strongest reason and usually the real one.
- **Genuinely different scaling profiles.** The optimizer is CPU-bound and bursty; the CRUD API is I/O-bound and steady. Scaling them together means over-provisioning one to serve the other.
- **Team autonomy at real headcount.** Conway's law works whether or not you cooperate with it. At fifty engineers this is decisive; at five it is fiction.

And the costs, which arrive whether you planned for them or not:

- **No distributed transactions.** You cannot wrap two services' databases in one `SaveChanges`. Multi-step operations become **sagas** — a sequence of local transactions, each with a compensating action for when a later step fails. "Compensating" is doing real work to undo, not rolling back: you issue a refund, you do not un-charge.
- **Eventual consistency becomes a product decision**, not a technical one. Someone has to decide what a user sees in the window where two services disagree, and if nobody decides, they see something arbitrary.
- **Versioning is forever.** Two services deploy independently, so both versions of a contract are live simultaneously — the same expand/contract discipline as chapter 04 §8's schema changes, applied to every message and endpoint.
- **The operational surface multiplies.** Every service needs its own pipeline, dashboards, alerts, on-call story and secret rotation.

**The failure mode with a name:** the *distributed monolith* — services that must be deployed together, that share a database, or that call each other synchronously in a chain to serve one request. It has every cost above and none of the benefits, and the tell is simple: **if two services write to the same tables, they are one service with a network call in the middle.** Data ownership is the actual boundary; the process boundary is just where you drew it.

So the honest default: **start with a modular monolith.** Enforce boundaries in the code — separate projects, no shared entities across modules, communication through explicit interfaces — and extract a service when a specific one of the three good reasons above actually applies to a specific module. Boundaries you can move are worth more than boundaries you guessed at, and one deployable that is well-organised inside beats five that are not.

---

## 8. The detail most people miss: a timeout tells you nothing about whether the work happened

This is the one that reorganises everything else, and it is invisible until it has cost you money.

A call fails with an exception and you know it failed. A call succeeds and you know it succeeded. **A call that times out is a third outcome: you do not know.** The request may never have arrived. It may have arrived, executed completely, and the response was lost on the way back. It may be executing right now, after you gave up.

There is no way to distinguish these from the caller's side. None. The information does not exist where you are standing.

Every uncomfortable pattern in distributed systems is a response to that one fact:

- **Retries are unsafe by default**, because retrying an operation that already succeeded runs it twice. That is why §5 says only retry what is safe.
- **Idempotency keys exist** so that "twice" and "once" produce the same result. The caller generates an ID, the receiver records it, and the second attempt returns the first outcome rather than repeating the work. Then a retry is safe *because you made it safe*, not because the network improved.
- **At-least-once delivery is a choice** — the only honest one. A broker facing this exact ambiguity picks redelivery over silent loss, and pushes the resolution to the only place that can handle it: your handler.
- **Sagas need compensations** rather than rollbacks, because a step whose outcome is unknown cannot simply be undone. You have to be able to make it right afterwards.
- **Reconciliation exists.** For anything financial or otherwise consequential, the final answer is a periodic job that compares two systems and fixes drift — because no amount of care at call time makes the ambiguity go away.

The interview-grade version, worth saying in one breath: *a timeout is not a failure, it is an unknown outcome — so either the operation is idempotent and I retry, or it is not and I need a way to find out what actually happened.* Teams that treat timeouts as failures write retry logic that quietly duplicates work, and they usually find out from a customer.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| Only some clients receive a real-time message | SignalR connections are per instance; no backplane, §3 |
| Real-time works on one replica, breaks when scaled | Same cause, or missing sticky sessions on a non-WebSocket transport |
| A message is processed twice | At-least-once delivery. The handler is not idempotent, §4 |
| A consumer spins forever on one message | A poison message with no dead-letter queue, §4 |
| A record exists but its event was never published | Commit and publish are not atomic — needs an outbox, §4 |
| An event was published for something that did not commit | Publish-then-commit, same cause reversed |
| A brief outage becomes a long one | Retries without jitter — synchronised herd, §5 |
| One slow dependency makes the whole service unresponsive | No timeout, no bulkhead — ch. 05 §2 arriving from outside |
| A payment or job is duplicated after a retry | The retried call was not idempotent, and the timeout was read as a failure, §8 |
| A trace ends at the queue | `traceparent` was not copied into message metadata, §6 |
| Logs cannot be filtered by request | Interpolated strings instead of structured fields — ch. 07 §8 |
| A renamed field silently becomes null in the caller | An untyped contract; a `.proto` or generated client would have failed the build, §2 |
| Two services must always be deployed together | A distributed monolith — shared data or a synchronous chain, §7 |
| A cross-service operation half-completed with no way back | No saga and no compensating action, §7–8 |

---

## Check yourself

Answer out loud, without looking:

1. Name the five guarantees a method call gives you that a network call does not, and pick the one that causes the most design work.
2. SignalR works perfectly, you scale to three instances, and users report missing updates with no errors anywhere. Explain the mechanism.
3. Why is "exactly-once delivery" not something you can buy, and what do you build instead?
4. Your service writes a row and publishes an event. Give both orderings, say what breaks in each, and name the fix.
5. A call to a payment service times out. Enumerate everything that might have happened, and say what makes it safe to retry.

---

## Questions this chapter answers

Five of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#ch09):

| # | As originally asked | Answered by section |
|---|---|---|
| 31 | What is SignalR and its use cases? | §3 — the server initiates, and the connection is the operational cost |
| 32 | Build real-time applications | §3 — hubs, transport negotiation, backplane, and when polling is smaller |
| 35 | How do you perform logging? | §6 — across processes the question becomes what happened to *this request* |
| 53 | Best practices for microservices architecture | §7 — a boundary is a cost; name what you are buying |
| 89 | Support for gRPC | §2 — same shape as REST, different contract, enforced at build time |

Chapters 10–12 are reference material rather than concepts, and are deliberately thinner: the platform's history and shape, interop and migration, and the tooling.

## Next

→ `10-platform-and-history.md` — nine chapters have described how .NET behaves. This one is why it is shaped that way: what .NET Core replaced, what .NET Standard was for, and why the version numbers went the way they did.
