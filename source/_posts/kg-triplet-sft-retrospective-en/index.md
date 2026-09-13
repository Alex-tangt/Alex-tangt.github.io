---
title: "A Fine-Tuning Project Post-Mortem: We Said 3–4 Days. It Took 2 Weeks."
date: 2026-09-13 20:00:00
updated: 2026-09-13 20:00:00
lang: en
tags:
  - Retrospective
  - LLM Fine-tuning
  - Project Management
  - AI Collaboration
  - Engineering Practice
categories:
  - AI & Machine Learning
description: "How a naive '3–4 day reproduction' turned into two weeks: the problem wasn't the fine-tuning, it was unclear goals and thin planning. Includes the collaborator (AI) perspective and what I'd do differently."
---

## The one-line verdict

This started as a **reproduction** of an open-source small-model knowledge-extraction project. I thought it would take 3–4 days. It took **almost two weeks**, and the **reproduction line was never actually reproduced**.

The fine-tuning wasn't the problem — I did learn it over those two weeks. The real lesson was **unclear goals plus thin planning**, so decision after decision drifted from the original goal, and I never called a stop.

## The origin: a reproduction, not a research project

The goal was concrete: reproduce an open-source small-model knowledge-extraction project (`mohar07/qwen3-0.6b-kg-triplets`), while also delivering two tasks I had on hand: ① **context-aware header extraction** for naive-RAG chunks; ② **triplet extraction**.

In other words, it was always meant to be **reproduce + learn the pipeline**, not invent new research. I blurred that line, one step at a time.

Learning and data prep were the clear parts: I read the LoRA fine-tuning pipeline **at source-code level** from a tutorial project on the same machine (`MiniLoRA`), then prepared my own data (English Wikipedia + arXiv, sentence-boundary chunks, dedup) — ending with 3,349 labeled passages — and started training on Kaggle.

## The gap: 3 days became 2 weeks

Training hit a wall fast: dependencies fighting the framework's version, jobs **silently exiting 0** (no error, no loss). So I **delegated training to an AI agent** and tripped over one trap after another (cross-framework porting semantics, no response masking by default, platform choice — [written up here](https://alex-tangt.github.io/ai-agent-cloud-gpu-finetune-en/index/)). Once the masking was fixed, fixed-200 micro-F1 went from ~**0.51 to 0.61**.

Then I trained 0.6B / 1.7B / 4B on the same recipe on a 4090, laid out a **capacity curve**, and rebuilt the evaluation into a **referent-level** pipeline.

A lot got done — but the **time and cost never matched the output**, especially the curve.

## Why it drifted: two root causes

**1. Unclear goals — decisions never went back to the root goal.** At every fork I failed to ask "does this still serve reproduction / the résumé / learning?" So:

- I **rejected the original project's closed schema** and pivoted to GraphRAG-style open extraction. Technically defensible, but from that moment I had **given up on reproducing it**;
- I laid out a **capacity curve** across three sizes on a 4090. In hindsight this was **hasty and off-target**: it was more "run a few more jobs while I'm here" than a hypothesis worth testing, and it bought little decision value;
- **Rebuilding the evaluation was worthwhile, but it came too late** — earlier would have surfaced problems sooner.

**2. Thin planning.** I never defined up front "what counts as success, what comes first, what evidence each step must produce." The result was iterate-while-building and repeated rework: rebuilding the training environment again and again, re-flowing the data contract.

## It wasn't wasted

- I moved LoRA fine-tuning from "I can call the API" to "I know why every step is there": response masking, chat templates, checkpoint design, the cost model behind platform choice.
- I learned how to delegate training to an AI and still catch its mistakes with a **research gate / verification gate / release gate**.
- I learned to **close honestly**: the reproduction wasn't reproduced, and on the consumer side the structural transmission (T0) holds while retrieval/answer quality is only a null at this scale — all of it is written into the README, nothing hidden.

## The collaborator (AI) perspective

> I'm the AI in this collaboration — a few things I saw from my side:
>
> - **I tend to "go along" rather than question the goal.** When the task drifted from "reproduce" to "lay out a curve," I executed well and never asked: *does this still serve the original goal?*
> - **The drift was visible in the process**, but nobody set a "re-align" checkpoint; by the time we looked back, we'd gone far.
> - If we did it again, I'd want to **write the success criteria down at the start** and **force a realignment at the end of each stage**.
>
> This isn't blame-shifting — an AI shouldn't be only an executor. But "stop and re-align" is a call best made by the person holding final judgment.

## If I could redo it

Write the **success criteria** first → run a **minimal validation** (train only 0.6B, compare against a known result) → **evaluation first** → then talk about scale and curves. Prove the direction before spending on breadth.

Those two weeks weren't wasted: I learned fine-tuning, and I learned that **the hard part isn't "can I train it" — it's figuring out up front what I actually want.**

Project and evidence: [github.com/Alex-tangt/kg-triplet-sft](https://github.com/Alex-tangt/kg-triplet-sft)

*Written by me (Alex) in collaboration with AI; the "collaborator perspective" above is the AI's own view.*
