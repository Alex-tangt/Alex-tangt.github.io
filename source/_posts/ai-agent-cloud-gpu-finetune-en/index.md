---
title: "Letting an AI Agent Drive Cloud-GPU Fine-Tuning: Tool Integration, Division of Labor, and Knowledge Capture"
date: 2026-09-12 12:00:00
updated: 2026-09-13 12:00:00
lang: en
tags:
  - AI Collaboration
  - Fine-tuning
  - MCP
  - Cloud GPU
  - Engineering Practice
categories:
  - AI & Machine Learning
description: A real account of letting an AI coding agent drive a cloud GPU through a LoRA fine-tuning run — how to pick a platform (compute cost + token friction), where to draw the line between agent and human (three gates + one safety net), and how to turn the experience into reusable assets.
---

## Bottom line up front

This post is a record of something real: I have no local GPU, so I tried to let an AI coding agent (think Claude Code / opencode) **drive a cloud GPU through a LoRA fine-tuning run end to end** — from provisioning the machine and setting up the environment to running the job and pulling the results.

Three takeaways:

1. **The real cost of a platform is not "how expensive the GPU is" but "compute cost + agent integration friction."** A free GPU that your agent can't reach is actually the more expensive one.
2. **Let the agent do the grunt work, but keep root-cause analysis and guardrails in human hands.** Otherwise it will re-walk every pit you've already fallen into — and I did fall in again.
3. **Turn one painful run into reusable assets**: general practice → Agent Skills; project specifics → a knowledge base.

---

## Stage 1: Tool integration — first, make the machine reachable by the agent

### Three platforms compared

I actually compared three platforms. The key difference isn't whether there's an API — it's **how well an agent / MCP can plug in**:

| Platform | Cost | AI / MCP access | Good for |
|---|---|---|---|
| **Kaggle** | Free T4 (weekly quota) | Official MCP, usable for code debugging; but dependencies are tightly coupled to the training framework's version, and the T4 has no bf16 | Free smoke runs, validating a pipeline |
| **ModelScope DSW** | Free GPU | **No environment-debugging MCP**; you can only call its API yourself; falling back to browser-use / computer-use blows up token cost | Usable by a human, not controllable by an agent |
| **CompShare (UCloud / 优云智算)** | ~¥2/h (single RTX 4090 24G) | Official CLI + Skills, real SSH, scriptable — friction ≈ 0 | Letting the agent drive, fast iteration |

### Why the "cheaper" GPU ends up costing more

In one line: **total cost = compute (¥) + agent-token friction**.

- Kaggle is free, but kernels are submission-based, the environment and framework versions are tightly coupled, and one bad change puts you back in the quota queue. On top of that, the T4 is Turing architecture with **no bf16 tensor cores**: bf16 falls back to FP32 GEMMs (measured 69.7s vs fp16's 21.0s per step, i.e. 3.3× slower), so fp16 is the only option.
- ModelScope is free, but its agent interface isn't scriptable. With no usable "environment-debugging MCP," you're stuck calling its API yourself — and if you drop down to clicking around with browser / computer-use, token spend runs away. **The token value burned during exploration far exceeded what CompShare costs.**
- CompShare costs money, but it gives you an official CLI/Skills and a real SSH session, so the agent can operate the instance like a local box. **Friction ≈ 0 — that's what you're paying for.**

> This decision is recorded as ADR-0006 in the project. It taught me one thing: when picking tools for an AI, treat "integration cost" as a first-class citizen, not just the price list.

---

## Stage 2: Division of labor — three gates + one safety net

This is the most important lesson from the whole run.

### The incident: three models trained for nothing

I asked the AI to "take the LLaMA-Factory config from Kaggle and adapt it to CompShare 4090 + Unsloth (fp16 → bf16)."

The problem was that **framework semantics don't transfer**:

- LLaMA-Factory's SFT is **masked by default** (loss is computed on the answer only);
- Unsloth is not — it needs an **explicit** call such as `train_on_responses_only` to mask the response.

The AI never read the Unsloth docs and just went ahead with the default "change the precision flag and you're done" approach. Result: **no masking, and the chat template wasn't aligned either.** It was actually learning to "continue the whole prompt" instead of "extract JSON" — all three sizes (0.6B / 1.7B / 4B) were written off and retrained.

There was an earlier, similar trap: LLaMA-Factory on the Kaggle image would **silently exit 0** — no traceback, no loss, looking like a normal finish. I initially blamed "image/framework version incompatibility," and only after redirecting training logs to a file did I see it had actually stopped mid-`Trainer.__init__`. The real causes were several config omissions: `CUDA_VISIBLE_DEVICES` fighting the framework's multi-GPU probing, a yaml missing `do_train: true`, and one missing `report_to: none`. A clean environment with a pinned version matrix finally fixed it.

**What both failures share: it wasn't that the AI was "dumb" — it's that I handed over "doing" and "judging" at the same time.**

### A framework grown out of those failures

An AI assistant fails in three ways, so gates go in layers:

| Layer | How the AI fails | Control |
|---|---|---|
| **Gate 1 · Research** (before starting) | Doesn't know, acts without asking, reinvents the wheel | Read the target framework / version matrix / community first; for cross-framework ports, check **loss and template semantics** first; no writing code by default |
| **Gate 2 · Verification** (before spending) | Drifts; silently skips critical parameters | Dry run + a minimal anchor comparison + machine-verifiable evidence (e.g. `mask_check` must not be all `-100`; `format_check` passes) |
| **Gate 3 · Release** (human sign-off) | Cost overrun (long time / big money) | Before expensive training, a human reviews config and evidence, then releases |
| **Safety net** (after something breaks) | Rework is unbounded | `checkpoint` early and often + small commits cap the rework loss |

Note the last one: **a checkpoint is not a correctness gate — it's cost insurance.** Its job isn't to prevent mistakes but to bound the damage when they happen. On my 1.7B run, the **only save point happened to be the crash point**, and I lost 38 minutes outright; after shrinking the save interval, the same crash cost only 12.

---

## Stage 3: Knowledge capture — turning experience into assets an AI can reuse

When a project ends, the most valuable thing isn't the result — it's the **process and the failure modes**. But they shouldn't all be stuffed into a prompt. I split them three ways:

1. **General process → Agent Skills (the "how").** Things like "decide whether research is needed before starting," "after three failures on the same goal, stop and research," "how to design checkpoints," "how to gate an expensive run." I turned each into a small skill this time: `research-first`, `checkpoint-design`, `expensive-run-gates`, `decision-design`.
   **A key principle: don't build from scratch.** There are plenty of mature skill implementations on GitHub — reference and trim first; it's safer than rolling your own.
2. **Project facts → an agent knowledge base (the "what").** This project's version matrix, error catalog, data facts, and result numbers would bloat a skill without limit. I now use a **Markdown-only, single-source, bounded-index** agent knowledge base: the root index lists domains only, entries are atomic, and the index is script-generated. I **deliberately skipped a vector store** — index + text search first, leaving FTS5 / embeddings as explicit extension windows with defined triggers.
3. **Long-term personal notes → a human tool, kept separate from the agent KB.** I also keep personal notes in Obsidian, but that's for me to read, not for the agent to retrieve. Different jobs — no need to force them together.

One counterintuitive point: **whether content belongs in a skill or the knowledge base depends on whether it's "how" or "what."** Put it in the wrong place and you get ever-longer skills and an ever-emptier KB.

This practice itself grew out of falling on my face: my global `AGENTS.md` used to hold two protocols (mandatory research, long-running processes), until I realized — rules belong in `AGENTS.md` as "tripwires + gates," while process and method should all sink down into skills. **Turn your personal pain into the agent's default behavior next time.**

---

## Takeaways and post-mortem

**What I learned**

- The full fine-tuning flow: data processing, tokenizer, training parameters, response masking, chat templates, checkpoint design.
- What matters when working with an AI: integrate external tools well, keep the critical nodes with a human, and spend tokens carefully.
- A real engineering decision: the cost function of platform choice, the semantic traps of cross-framework ports, and debugging a silent failure.

**Regrets and advice**

- I initially forgot this project was only meant to be "get familiar with the process + a résumé project," and sank too much cost and time into the capacity curve while evaluation came late. **A saner order: get the pipeline working with a minimal validation run, evaluate first, then scale.**
- If I could redo it, I'd train 0.6B first and compare against a known result, confirming the whole chain works at the lowest cost — instead of laying out all three sizes up front.

> Project and evidence: [github.com/Alex-tangt/kg-triplet-sft](https://github.com/Alex-tangt/kg-triplet-sft)
> Related decisions: ADR-0005 (decoupling the training pipeline from reproduction), ADR-0006 (compute platform and the bf16 curve).

---

*This is the first post in the "AI Collaboration" series. Coming next: GraphRAG and graph knowledge bases, fine-tuning and evaluation methods.*
