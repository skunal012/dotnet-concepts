# .NET concepts

A concept-first handbook for the .NET platform, built for long-term understanding rather than interview recall.

Each chapter explains **one mental model** properly, then shows the framework details as consequences of it. The goal is that a question I have never seen before is answerable by reasoning from the model — not by remembering whether I read that particular fact.

---

## Why this exists

I started from a list of 100 .NET interview questions and found they were not 100 things. They were about twelve concepts, asked eight different ways each.

Five separate questions about middleware order, routing, Kestrel, and HSTS are one question: *what happens to an HTTP request between the socket and my controller?* Answer that once, properly, and the five fall out — along with the twenty variations a real bug will throw at me.

So this repo is organised by concept, not by question.

The full mapping — all 100 questions, which chapter dissolves each, and the reasoning per question — lives in [`QUESTIONS.md`](QUESTIONS.md). It also records the judgement calls (why caching is a memory question, why logging is a distributed-systems question), because those assignments *are* the insight.

---

## How each chapter is built

Three layers, each derived from the one above it:

| Layer | Path | Purpose |
|---|---|---|
| **Handbook** | `concepts/` | The explanation. Read once carefully, reread when rusty. |
| **Widget** | `widgets/` | Standalone HTML. Only for concepts where motion or ordering carries the meaning. |
| **Anki deck** | `anki/` | Spaced repetition. Cards test *why*, never *what*. |

The handbook is the source of truth. If a chapter changes, its cards get regenerated from it.

### On the cards

A card like *"what is middleware?"* is worthless — I would recognise the answer without knowing it. A card like *"what happens if `UseAuthorization` runs before `UseRouting`?"* cannot be faked.

Import: Anki → File → Import. Files are tab-separated with tags already set. Cards tagged `leech-risk` are ones I expect to fail repeatedly; that tag is a note to myself to go reread the chapter rather than grind the card.

---

## Roadmap

Ordered by dependency, not by difficulty. Auth needs the pipeline. EF Core is easier after DI lifetimes.

| # | Chapter | Handbook | Widget | Cards |
|---|---|---|---|---|
| 01 | The request pipeline | [✅](concepts/01-the-request-pipeline.md) | [✅](widgets/01-request-pipeline.html) | [✅](anki/01-request-pipeline.txt) |
| 02 | Configuration as layers | [✅](concepts/02-configuration-as-layers.md) | [✅](widgets/02-configuration-as-layers.html) | [✅](anki/02-configuration-as-layers.txt) |
| 03 | DI and service lifetimes | [✅](concepts/03-di-and-service-lifetimes.md) | [✅](widgets/03-di-and-service-lifetimes.html) | [✅](anki/03-di-and-service-lifetimes.txt) |
| 04 | EF Core's mental model | [✅](concepts/04-ef-cores-mental-model.md) | [✅](widgets/04-ef-cores-mental-model.html) | [✅](anki/04-ef-cores-mental-model.txt) |
| 05 | Async, threads and memory | [✅](concepts/05-async-threads-and-memory.md) | [✅](widgets/05-async-threads-and-memory.html) | [✅](anki/05-async-threads-and-memory.txt) |
| 06 | Authentication and authorization | ⬜ | — | ⬜ |
| 07 | Testing and TDD | ⬜ | — | ⬜ |
| 08 | Build and ship | ⬜ | — | ⬜ |
| 09 | Distributed and real-time | ⬜ | ⬜ | ⬜ |
| 10 | Platform and history | ⬜ | — | ⬜ |
| 11 | Interop and migration | ⬜ | — | ⬜ |
| 12 | Tooling reference | ⬜ | — | ⬜ |

Chapters 10–12 are reference material rather than concepts, and are deliberately thinner.

A dash means a widget would be decoration. Animating "what is .NET Standard" teaches nothing.

---

## Working on it

One chapter at a time, one commit per chapter. No deadline — this is a reference I expect to keep for years, not a course to finish.

New chapters follow `TEMPLATE.md`, which encodes the structure: one-sentence thesis, why the concept exists, the mechanism, the consequences, common failure modes, self-check questions with no answers, and the list of source questions the chapter dissolves.

The self-check questions have no answers on purpose. Retrieval I struggle with is the retrieval that sticks.
