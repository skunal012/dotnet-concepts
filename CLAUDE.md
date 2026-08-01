# CLAUDE.md — project context for Claude Code

## What this repo is

A concept-first .NET learning handbook. 100 interview questions were reduced to 12 underlying mental models. Each chapter explains one model properly, then the questions fall out as consequences — not memorised, derived.

Public repo at: https://github.com/skunal012/dotnet-concepts

## Who I am

Kunal — full-stack developer (C#/.NET Core + React/TypeScript), 5+ years, currently at LiSEC building the AI Glass Platform. Primary production work: Rack/Container Optimization Engine (gRPC, PostgreSQL, OR-Tools, NUnit/Moq). Transitioning into AI engineering, staying in the Microsoft/Azure/Semantic Kernel ecosystem. Based in Dubai, originally from Bihar.

When writing examples in chapters, anchor to real GROS/LiSEC work where it fits — the optimizer's `ConcurrentDictionary` metric cache, SignalR progress streaming, gRPC service tokens, EF Core auto-migration, OpenAPI-generated typed clients, nginx-fronted deployments. This is my edge: textbook + "here's how I actually used it".

## Repo structure

```
README.md           — purpose, method, roadmap table
QUESTIONS.md        — all 100 questions mapped to chapters with per-question reasoning
TEMPLATE.md         — enforced chapter structure (follow exactly)
concepts/           — handbook chapters (markdown, source of truth)
widgets/            — standalone HTML files, open offline, no build step
anki/               — tab-separated Anki import files
```

## What's done

- Scaffold commit: README, TEMPLATE.md, .gitignore
- Chapter 01: the request pipeline (concepts/ + widgets/ + anki/)
- Chapter 02: configuration as layers (concepts/ + widgets/ + anki/)
- Chapter 03: DI and service lifetimes (concepts/ + widgets/ + anki/)
- Chapter 04: EF Core's mental model (concepts/ + widgets/ + anki/)
- Chapter 05: async, threads and memory (concepts/ + widgets/ + anki/)
- QUESTIONS.md: all 100 questions assigned, verified no gaps or duplicates

## Chapter roadmap (dependency-ordered)

| #  | Chapter                          | Status | Widget needed? |
|----|----------------------------------|--------|----------------|
| 01 | The request pipeline             | ✅     | ✅ done         |
| 02 | Configuration as layers          | ✅     | ✅ done         |
| 03 | DI and service lifetimes         | ✅     | ✅ done         |
| 04 | EF Core's mental model           | ✅     | ✅ done         |
| 05 | Async, threads and memory        | ✅     | ✅ done         |
| 06 | Authentication and authorization | next   | no              |
| 07 | Testing and TDD                  | ⬜     | no              |
| 08 | Build and ship                   | ⬜     | no              |
| 09 | Distributed and real-time        | ⬜     | yes             |
| 10 | Platform and history             | ⬜     | no              |
| 11 | Interop and migration            | ⬜     | no              |
| 12 | Tooling reference                | ⬜     | no              |

Chapters 10–12 are reference material, deliberately thinner.

## How to write a chapter

### 1. Read TEMPLATE.md — follow the structure exactly

Every chapter has: one-sentence thesis → why the concept exists → the mechanism → consequences → the detail most people miss → common failure modes table → self-check questions (no answers) → questions this chapter answers (linked to QUESTIONS.md).

### 2. Core writing rules

- **Explain the model, not the facts.** If you're writing a list to memorise, you skipped a step — find the underlying reason that generates the list.
- **Self-check questions have no answers.** Retrieval you struggle with is retrieval that sticks.
- **Each question lives in exactly one chapter.** Assigned by which mental model *generates* the answer, not which topic word appears.
- **Examples from real work.** Use GROS/LiSEC examples where they fit naturally — never force them.
- **Honest calibration.** For topics I work adjacent to (Razor, Blazor, Tag Helpers), say "I work React-side, but here's my understanding" rather than bluffing.

### 3. The three layers per chapter

**Handbook** (`concepts/NN-slug.md`): The explanation. Written as an argument with a thesis running through it, not a reference page. This is the source of truth — the other two derive from it.

**Widget** (`widgets/NN-slug.html`): Only for concepts where motion or ordering carries meaning (~5 of 12). Standalone HTML, no build step, single file — all CSS and JS inline. Dark mode via `prefers-color-scheme`, `prefers-reduced-motion` respected, real `<button>`s with `aria-pressed` rather than clickable divs. IBM Plex is pulled from Google Fonts with a full system-font fallback stack, so the widget renders correctly offline — it just falls back to system fonts. See `widgets/01-request-pipeline.html` for the reference implementation.

Every widget must be linked from its chapter (the `▶ **Widget:**` line under the thesis) and from the README roadmap table. A widget nobody can find from the handbook is a widget that does not exist.

**Anki cards** (`anki/NN-slug.txt`): Tab-separated, tags in column 3. Cards test *why*, never *what*. "What is middleware?" is worthless. "What happens if UseAuthorization runs before UseRouting?" is unfakeable. Format:

```
#separator:tab
#html:true
#tags column:3
Question text here	Answer with <b>HTML</b> formatting	dotnet::chapter-slug core
```

Tag `leech-risk` on cards expected to fail repeatedly.

### 4. Commit convention

One commit per chapter. Message format:

```
Chapter NN: <concept name>

The one idea: <thesis sentence>.

<2-3 lines of what the chapter covers>

Dissolves questions X, Y, Z from the source list.
```

### 5. After writing a chapter

- Update README.md roadmap table: ⬜ → ✅
- Update QUESTIONS.md: ⬜ → ✅ for that chapter's header
- Verify: every question number 1-100 still appears exactly once
- Link the chapter's closing "questions this chapter answers" table to QUESTIONS.md

## Question-to-chapter mapping (quick reference)

Ch 01 pipeline:       16,17,18,19,24,25,33,77,83
Ch 02 configuration:  9,20,23,70,75,76,78,79
Ch 03 DI/lifetimes:   21,22,61,62,63,64,65
Ch 04 EF Core:        28,29,56,57,58,59,60,72
Ch 05 async/memory:   30,36,37,38,39,40,95,96,97,98
Ch 06 auth:           27,48,50,80,81,82,84
Ch 07 testing:        41,42,43,44,45
Ch 08 build/ship:     13,46,47,49,51,52,54,55
Ch 09 distributed:    31,32,35,53,89
Ch 10 platform:       1,2,3,7,14,85,86,87,88,93,94,99,100
Ch 11 interop:        26,34,66,67,68,69,90,91,92
Ch 12 tooling:        4,5,6,8,10,11,12,15,71,73,74

## Tone and style

- Direct, no fluff. Written for a developer who builds full-stack end-to-end.
- Code examples: C# / .NET 8+, minimal but real — one honest snippet beats three paragraphs.
- No performative caution. If something is a security hole, call it a security hole.
- Silently correct grammar in any text I draft.
- Sentence case for all headings.
