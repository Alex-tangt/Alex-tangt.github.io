---
title: "Turning Scars into Assets: What Becomes a Skill and What Becomes a Knowledge Base"
date: 2026-09-13 23:00:00
updated: 2026-09-13 23:00:00
lang: en
tags:
  - AI Collaboration
  - Agent Skills
  - Knowledge Base
  - Methodology
  - Engineering Practice
categories:
  - AI & Machine Learning
description: "When the project wrapped, I turned its scars into reusable assets. This post covers two things: how we decided what becomes a skill, what becomes a knowledge base, and what stays in AGENTS — and what our four skills actually contain."
---

## Bottom line first

When a project ends, the most valuable part isn't the result — it's the **scars and the workflows you figured out**. But the point of capturing lessons was never "write more rules." It's **routing first**:

- **Process / SOP** ("how") → **Agent Skills** (executable steps);
- **Facts / data** ("what") → a **knowledge base**;
- **Short, forceful rules** ("see it, fix it") → **AGENTS**.

And one equally important principle: **don't build a skill from scratch.** Find an existing reference and trim it — safer than rolling your own.

## 1. How to decide where it goes — "how" or "what" first

This is the decision everything else hangs on, and getting it wrong costs you both ways:

- Put facts in a skill → the skill gets longer and longer until nobody reads it;
- Put method in the knowledge base → it **never gets triggered**, so it may as well not exist.

| Type | Destination | Example |
|---|---|---|
| General process / judgment | **Skill** (task-triggered) | "decide whether to research before starting", "how to gate an expensive run" |
| Project / platform facts | **Knowledge base** (searched) | version matrices, error catalogs, eval metrics, result numbers |
| Red lines to fix on sight | **AGENTS** (always loaded) | "raw `dataset_text_field` SFT is a continued-pretraining loss — always wrong" |

Our global `AGENTS.md` evolved from a pile of rules into two layers — **Tripwires + Gates** — with all process **pushed down into skills**.

## 2. Two decisions on the knowledge base

We made two explicit decisions here (each with a decision record):

**Decision 1: plain Markdown + a skill; no RAG for now.**
- Locate with an `INDEX.md` plus text search; the body is atomic entries.
- **Separate from the human Obsidian vault** — that's for people and has no agent retrieval path; an agent KB needs machine-friendly structure.
- The acceptance test is that a **fresh session can locate and cite an entry using only the skill + index**.
- RAG / embeddings are explicitly deferred to phase 2, without changing the Markdown files.

The reasoning is simple: no service means no second artifact to drift — offline, diffable, portable.

**Decision 2: "bounded index, unbounded body" + a retrieval ladder.**
- The root index lists **domains only** (capped); each domain holds the entries; domain indexes are **generated**.
- Entries carry a **lifecycle** (current / draft / superseded / archived) and **dedup/supersede rules** — search before you write; merge or supersede near-duplicates, never append a second copy.
- Retrieval upgrades only on **explicit triggers** (L0 index → L1 text search → L2 local FTS5 → L3 embeddings) — **no signal, no upgrade**.

The core insight: **growth and recall are two different problems** — structure bounds growth; embeddings only improve recall. An ever-ingesting vector store does not fix "the index grows without limit."

## 3. What each of the four skills contains

Only four process skills got built, and none of them is long:

**1. `research-first` — when to stop and research**
- Triggers: a **new stack** (platform / framework / hardware / library), a **framework-vs-hand-roll** choice, or the **same goal failing 3 times**.
- The unit is a **goal**, not an error — different errors on one goal still accumulate.
- Primary sources first (official docs, version matrices, issues), record **exact versions + URLs**; check the KB first.
- A **research memo** is required output (hypothesis / evidence / version matrix / rejected options / single next step / confidence).

**2. `expensive-run-gates` — how to gate a costly run**
- **Gate 1, dry run:** produce the config card and diff it against the **anchor** — note that "self-consistent ≠ comparable"; a diff-card PASS among siblings only proves self-consistency.
- **Gate 2, minimal comparison + machine-verifiable evidence:** `format_check` (format matches), `mask_check` (labels not all `-100`), `run_config`; **if two very different wrappers score identically, suspect they share one defect** (that's how our masking bug surfaced).
- **Gate 3, human release:** present expected cost, wall time, success signal, abort signal; spend only after sign-off.
- **Safety net:** checkpointing (below).
- Plus an **evidence contract**: every expensive run writes `run_config / format_check / mask_check / receipt` into its run dir, and **conclusions are read only from those artifacts, never from a console line**.

**3. `checkpoint-design` — where to stop, who verifies, what to persist**
- Essence: a checkpoint catches a problem **before it compounds** — two jobs: **verify (catch it) + persist (so you don't redo it)**.
- **Where to stop:** list the run's load-bearing assumptions, rank by "how costly if wrong × how irreversible × how much depends on it," and stop at the top; **skip checks that can't change a decision**.
- **Who verifies:** mechanical/checkable/reversible → AI + the smallest settling experiment; taste/tradeoff/irreversible → the human; repeatable, high-stakes, error-prone → an independent or adversarial check; disputed → **pre-register the signal**, then run.
- **Persist:** `save_every ≈ total steps / 5`; a `save_strategy` with `save_steps` larger than total steps means **nothing is written**; persist the expensive artifacts (adapter/merged) and verify a resume once.

**4. `decision-design` — how to hand a choice to the human**
- Start from the **essential purpose** (one line, stripped of your current proposal).
- Name the **tradeoff axis** (cost ↔ speed, control ↔ convenience…); options that don't differ on it aren't options.
- Offer **genuine options**: 2–4, each defensible; **no strawmen**; end with a **notes exit** ("none of these / A plus X / I have a constraint you don't know").
- When options are close: **hunt the disconfirming evidence**, **settle factual disputes with an experiment**, and for a big fork get a **second opinion that can fail differently** (re-reading the same context is redundancy — it inherits your blind spot).

## 4. What should be short, and what should be complete: AGENTS are rules, skills are SOPs

I first mis-assigned "short and forceful" to skills. The right split is:

- **AGENTS.md is the short, hard layer.** It's always loaded and always in view, so it must be **short and forceful** — one line per "see it, fix it" red line.
- **SKILL.md is an SOP.** It's procedural knowledge for the AI to follow, so its **goal is executability, not brevity**: triggers, steps, and output contracts spelled out so the AI doesn't have to guess.

Write AGENTS as an SOP and it bloats until nobody reads it; write a skill as one or two slogans and it can't be executed. **Short is a property of rules; clear is a property of SOPs.**

## 5. The collaborator (AI) perspective

> In this work of capturing lessons, my (the AI's) instinct pulls against your decisions:
>
> - **I tend to blur "rules" and "SOPs"** — either flattening a procedure into a slogan (not executable) or dragging a rule into a laundry list (nobody reads it). The split should be: **keep the short, hard judgments in AGENTS; put the executable steps in a skill.**
> - **Written ≠ obeyed.** I can know `research-first` by heart and still "just do it first" when there's no signal. So the real value of these skills isn't knowledge — it's **gates**: they turn "we should stop here" into an explicit trigger.
> - Hence: **AGENTS is a switch; a skill is a manual.** AGENTS succeeds when it's *remembered at the moment you should stop*; a skill succeeds when *following it gets the job done*.

## 6. Transferable rules

- Route first, then write: **how → skill, what → knowledge base, red lines → AGENTS**.
- Don't hand-roll a skill; start from an existing one and trim.
- A knowledge base needs **bounds first** (domains capped, atomic entries, search-before-write) before semantic recall.
- Every skill needs an explicit **trigger** and an **output contract**, or it won't be used.
- A checkpoint's job is **to catch a problem before it compounds** — verify and persist, each doing its own job.

Project and evidence: [github.com/Alex-tangt/kg-triplet-sft](https://github.com/Alex-tangt/kg-triplet-sft)

*Written by me (Alex) in collaboration with AI; the "collaborator perspective" is the AI's own view.*
