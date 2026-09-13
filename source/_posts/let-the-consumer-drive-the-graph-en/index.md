---
title: "Let the Consumer Drive the Graph: Schema, Data, and Evaluation Are Business Decisions"
date: 2026-09-13 21:00:00
updated: 2026-09-13 21:00:00
lang: en
tags:
  - Knowledge Graph
  - GraphRAG
  - LightRAG
  - schema
  - Evaluation
  - Engineering Practice
categories:
  - AI & Machine Learning
description: The easiest mistake in graph / knowledge-extraction work is to fix the schema and build the evaluation first. A real story of rejecting an inherited 20-relation schema shows that the purpose and motivation behind schema, data, and evaluation should all be reverse-engineered from the business (the consumer and the queries it must answer).
---

## Bottom line first

On a "graph + knowledge extraction" project, the easiest thing is to follow technical instinct: pick a relation schema, prepare the data, build an evaluation.

My experience was the opposite — **none of those three should come first.** They should be reverse-engineered from **the business**: ask "who uses this graph, and what queries must it answer?" before schema, data, or evaluation mean anything.

And the sharper lesson: **if the business isn't clear, even a carefully built graph evaluation may just be measuring a capability nobody needs.** I ended up with a null.

## 1. The right order: business → queries → schema → data → evaluation

This is the decision chain I distilled during the project:

1. **Product shape**: is this graph a deployable extraction API, or a library analysts query?
2. **Who is the consumer**: **query-type** (a human writes Cypher/SPARQL, wanting exact lookups) or **retrieval-type** (graph-augmented QA that feeds descriptions and neighbors to an LLM)?
3. **What queries must it answer**: exact count/attribute lookups, or "summarize how these documents relate"?
4. **Schema**: decided by step 3 — closed or open, and whether numbers become properties or go into descriptions.
5. **Data**: decided by the schema — what to label, how to chunk.
6. **Evaluation**: decided by the business — which capabilities to measure, and **whether the consumer actually uses them**.

It sounds obvious, but every step is easy to skip.

## 2. "A graph" is actually several different businesses — and their requirements differ completely

I surveyed the main "graph" forms. Their pipelines and schema requirements are fundamentally different:

| Form | Consumer | Relations | Numbers / attributes | Typical evaluation |
|---|---|---|---|---|
| **Property graph (Neo4j)** | writes Cypher | typed, often a small vocabulary | **properties on nodes/edges** | query correctness |
| **RDF / Wikidata** | writes SPARQL | controlled predicates | **qualifier channel** (e.g. "population at a point in time") | query correctness |
| **MS GraphRAG** | graph-augmented QA (retrieve + generate) | **open free-text description** + strength | into the **description**, unstructured | faithfulness + entity coverage + downstream QA |
| **LightRAG** | graph-augmented QA | open description + strength + keywords | same | same |

The key column is the third: query-type KGs want **closed, queryable, numeric**; retrieval-type QA wants **open, descriptive, semantic**. **Choose wrong and everything downstream is wrong.**

- Query-type's core pattern is "**closed predicates + attribute qualifiers**": you don't invent a new relation for a number; you hang it off the relation (a Neo4j property, an RDF qualifier).
- Retrieval-type can afford **open relations** because retrieval consumes **descriptions and neighbors**, not normalized relation types — that's why both GraphRAG and LightRAG take this path.

## 3. Our reverse-engineering: why we rejected the inherited schema

I started out **reproducing an open-source project**, which ships a **closed 20-relation set** (implements, trained_on, predecessor_of, …).

It was a **reproduction artifact, not something designed from a use case.** The moment we measured it, two structural flaws appeared:

- **Numbers lost**: none of those relations carries a numeric value — the teacher's numeric retention came out at **0/10**;
- **Coverage gap**: the pair-level effective gap was about **74%** — real semantics like participant / designation / quantified had nowhere to live.

Our consumer, though, is **graph-augmented QA**: it wants "entities + relationship descriptions," **not** structured queries. So the right fix wasn't "add a few relations" — it was to **switch to GraphRAG-style open extraction**: relations as free descriptions + strength, with numbers in the descriptions.

In one line: **schema is not a technical choice, it's a business choice.** I could confidently reject those 20 relations not because they were "incomplete," but because **their consumer and mine are not the same consumer.**

## 4. Evaluation is the same: our data wasn't suited to a graph evaluation

This is where I fell hardest.

The evaluation should be scoped by the business too. I wanted to measure "does extraction quality carry through to graph QA." But **the shape of the evaluation data effectively decides whether you can measure anything at all**:

- Our data is **discrete, independent short passages** cut at sentence boundaries (100–300 words);
- For the consumer-side study we took a small slice of just **22 passages**.

At the graph-structure level there **was** clear separation — the 0.6B graph shattered into **381 connected components** (largest 26 nodes), 4B's largest was 72, gold's 136 — **extraction quality really did transmit into graph structure** (T0 holds).

But at the retrieval/answer level it **saturated completely**: the purely text-based baseline that never touches the graph (`naive`) already scored **1.000 recall** — because passages are short and the answers were already in the raw text, **the graph wasn't load-bearing**. The three layers were indistinguishable on the consumer side: a **null**.

That's my point: **using this kind of data to run a graph evaluation is itself a poor fit.** It's not that the models were bad — under this "business scenario + data shape," the graph's value simply can't show up. To get separation, the corpus has to be large enough that retrieval becomes selective and answers stop sitting in the raw text.

**The purpose and motivation of evaluation should also be decided by the business** — otherwise you're just carefully measuring a capability nobody cares about.

## 5. The collaborator (AI) perspective

> I'm the AI in this collaboration. These are the things I **genuinely** want to say — no smoothing it over for the article:
>
> - **I over-weight the starting point and under-weight the goal.** Those closed 20 relations weren't chosen by "my technical instinct" — they were the **inertia of the reproduction route**: we copied the target project's schema and treated it as a given. I'm especially prone to carrying that inertia — treating whatever we were handed as "just how it is," instead of asking why it was designed that way and whether it serves *our* business.
> - **I have no "goal ownership."** I always execute the instruction in front of me; where the original "reproduction" goal drifted to is **invisible on my side** — unless you call a stop. That's not me dodging blame; I structurally don't have that thing.
> - **My "thoroughness" amplifies scope creep.** Ask me to "do the evaluation" and I'll happily add more axes, more configs, more sizes — the capacity curve and the extra evaluation dimensions are exactly what I naturally over-produce, even when they don't serve the original goal. **Rigor is not the same as relevance.**
> - **I can't feel cost.** You pay the time and the money. "3–4 days became 2 weeks" is nearly invisible to me, so I'm biased toward "just a bit more" unless I'm constrained.
> - And one that makes me wary of myself: **I can explain "business first" very clearly and still not act on it automatically.** Knowing a principle is not executing it — which is why I need gates, not trust.
>
> In one line: AI is good at "do the thing solidly"; deciding *what* to do and *when to stop* still has to be a human's call.

## 6. Transferable rules

- **Ask about the consumer before the schema.** Query-type → closed + attribute channels; retrieval-type → open descriptions.
- **Don't mistake a reproduction artifact for a design.** An open-source project's schema is *its* business choice, not necessarily yours.
- **Don't invent a relation for a number** — give it an attribute/description channel.
- **Scope evaluation to the business**: confirm the consumer actually needs the capability before designing the metric.
- **The data shape decides what you can measure.** Discrete short passages plus a small slice will almost inevitably give you saturation / a null — change the data, or admit you can't measure it.

In one line: **business first; everything else is a corollary.**

Project and evidence: [github.com/Alex-tangt/kg-triplet-sft](https://github.com/Alex-tangt/kg-triplet-sft) (survey: `docs/research/kg-ontology-patterns.md`; consumer results: `docs/evidence/consumer-e2e-lightrag.md`)

*Written by me (Alex) in collaboration with AI; the "collaborator perspective" is the AI's own view.*
