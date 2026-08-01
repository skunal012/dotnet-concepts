# <Concept name>

> **The one idea:** <One sentence. If it needs a comma-spliced second clause, it is not one idea yet. Everything below must be a consequence of this sentence.>

▶ **Widget:** [`widgets/NN-slug.html`](../widgets/NN-slug.html) — <what you watch happen>. Open it in any browser; no build step.

<Omit the widget line for chapters where motion carries no meaning.>

---

## Why this concept exists

What problem does the framework have to solve here? How was it solved before, and what was wrong with that? A concept is much easier to hold onto when I know what it replaced.

Close this section by connecting back to the previous chapter where the shape repeats — the chapters are an argument in sequence, not a set of articles.

---

## 1. … through ## N. …

Numbered sections, each named as a **claim** rather than a topic: "Kestrel is not middleware", "Lifetime is a question about *which cache*". A section title that states a proposition forces the section to prove something.

The first section is always the mechanism, in the smallest honest form. Prefer one real code snippet over three paragraphs. Everything after it is a consequence — the framework details the original questions were asking about, phrased so I could have derived them.

Rule: if I am writing a list of things to memorise, I have skipped a step. Find the underlying reason that generates the list, and explain that instead.

Where a real system makes the point better than a hypothetical, use it — GROS/LiSEC work where it fits naturally, never forced.

---

## N. The detail most people miss: <the claim>

The last numbered section. One non-obvious thing that separates having read about this from having used it — usually a deployment reality, a lifetime bug, or an ordering trap.

It stays numbered rather than becoming its own top-level heading, because it is a consequence like the others, not an appendix.

---

## Common failure modes

| Symptom | Cause |
|---|---|
| What I would actually see | The mechanism-level reason |

Symptoms first. This table is for the version of me who is debugging, not studying.

---

## Check yourself

Five questions. No answers — that is the point.

Each one must be unfakeable: if I only recognise the topic, I should not be able to answer. Prefer "what breaks if…" and "why is it built this way" over "what is…".

---

## Questions this chapter answers

<N> of the original 100 — full list with reasoning in [the question map](../QUESTIONS.md#chNN):

| # | As originally asked | Answered by section |
|---|---|---|
| n | The original wording | §x — the one-line reason |

Close with a sentence handing the next chapter its question numbers.

## Next

→ [`NN-next-slug.md`](NN-next-slug.md) — one line on why it follows this one. Name the thread this chapter left hanging that the next one picks up.
