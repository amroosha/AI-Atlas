---
name: learner
description: Turn research on a topic into dense, structured markdown reference notes. Use when the user asks to document, learn, summarize, or write notes on a concept, technique, or tool.
---

# MD Learner

Produce one markdown file that a competent engineer can skim in two minutes and keep as a reference.

## Hard rules

1. **No preamble, no conclusion.** Start with title + one-line definition. End with references.
2. **One idea per bullet.** ≤ 20 words.
3. **Tables over prose** whenever ≥2 items share ≥2 attributes.
4. **Define a term at first use**, in parentheses, ≤ 10 words.
5. **No sentence that only restates its heading.**
6. **Attribute or flag.** Every non-obvious claim gets a source or `(unverified)`.
7. **Budget 400–800 words.** Exceed only if the topic has ≥4 real variants. Never pad.
8. **Code is conceptual and minimal** — capture the core idea simply and generally; avoid vendor lock-in, unnecessary scaffolding, or overly strict boilerplate.

## Note skeleton

Use these in order, dropping any that would be empty:

- `# Title` + `> one-line definition`
- `**Scope**` — one line: covers X, not Y
- `## Why it exists` — the problem, ≤3 bullets
- `## Core mechanism` — numbered steps, one sentence each
- `## Variants` — table: variant | mechanism | cost | best for
- `## Minimal example` — code block
- `## Choosing` — decision rule or table
- `## Failure modes` — bullets shaped as symptom → cause → fix
- `## Uncertainty` — contested, unknown, or version-dependent
- `## References` — `- [title](url) — what it supports`

## Process

1. **Scope** — restate the ask in one line; list 3–6 sub-questions. If the ask is broad, keep the sub-questions that carry the most information.
2. **Gather** — prefer primary sources (docs, papers, source) over blog summaries. Track which source supports which claim as you go.
3. **Cluster** — sort findings into skeleton sections *before* writing prose. A finding that fits nowhere is either out of scope or a missing section.
4. **Write** — one pass, sections in order.
5. **Compress** — delete the first sentence of any section that is a transition, all adverbs, "it is important to note", repeated points, and any bullet a table row could carry.

## Self-check before returning

- [ ] Reads in under 2 minutes
- [ ] ≥1 table or diagram
- [ ] Zero sentences restating a heading
- [ ] Every term defined at first use
- [ ] Every non-obvious claim sourced or flagged
- [ ] No section could be deleted without information loss

## Anti-patterns

- "In this document, we will explore…" → delete
- Long prose where a 4-row table exists
- Explaining background the reader already has unless asked
- Listing 8 variants when 3 matter — merge or drop the rest
- Mixing "how it works" with "how to tune it" in one section