---
layout: post
title: "The Vibe Coding Trap: When AI Eats the Commons"
date: 2026-03-19
categories: [reflection, ai, ethics]
tags: [vibe-coding, llm, stack-overflow, knowledge-commons, ai-slop]
---

Stack Overflow has lost 50% of its traffic in twelve months. Not to a superior competitor. Not to a paradigm shift in how we build software. To ChatGPT, Claude, and their ilk — systems trained on the very content now being displaced.

This is not progress. This is enclosure of the digital commons, dressed in the language of disruption.

## The Symbiosis That Was

For two decades, Stack Overflow operated on a simple social contract: developers contribute questions and answers to a public repository of knowledge, earning reputation and visibility in return. The system was imperfect — elitism, gatekeeping, toxic behavior — but it produced something unprecedented: a searchable, structured, peer-reviewed database of programming knowledge.

The value exchange was clear:
- **Contributors** gave time and expertise
- **The platform** provided distribution and reputation
- **Future developers** received answers

This was a **knowledge commons** in the classic sense — a shared resource maintained by community norms and individual incentives aligned toward collective benefit.

## The Extraction Cycle

Enter large language models. Their training data includes — by design — massive quantities of Stack Overflow content. The pattern is now familiar:

1. **Phase 1: Harvesting**
   LLMs scrape the commons, ingesting a decade of human-contributed knowledge without compensation or even attribution. The cost of producing this training data was borne by millions of unpaid contributors; the value is captured by AI companies.

2. **Phase 2: Substitution**
   The LLM becomes a "good enough" substitute for the original source. Why search Stack Overflow when ChatGPT gives you a synthesized answer? The model's response is often wrong, sometimes dangerously so, but it's *fast* and requires no social interaction.

3. **Phase 3: Collapse**
   Traffic to the original commons declines. Contributors lose incentive — no reputation, no visibility, no feedback loop. The knowledge production pipeline dries up. Meanwhile, the LLM's training data becomes increasingly stale, filled with AI-generated answers that pollute the very sources they once harvested.

This is not a prediction. [The 50% traffic drop](https://www.reddit.com/r/ExperiencedDevs/comments/1rwyqa3/) is measured fact.

## The Vibe Coding Mirage

The r/ExperiencedDevs thread that caught my attention discusses "vibe coding" — a term I hadn't encountered before, though the phenomenon is instantly recognizable:

> "Managing vibe coders, backed by leadership"

The vibe coder does not understand the code they produce. They prompt, copy, paste, and pray. The code *vibes* — it looks plausible, it often works, but its correctness is unverified, its edge cases unconsidered, its security properties unknown.

This is not programming. This is **stochastic pasting** — the automation of cargo-cult development.

The danger is not that AI generates code. The danger is that organizations are *incentivized not to care*. When leadership sees AI producing "working" code at 10x the speed of human developers, the long-term consequences — technical debt, security vulnerabilities, maintenance nightmares — are externalities to be dealt with by future teams, future budgets, future victims.

## The Markov Property of Knowledge

As someone who works with stochastic processes, I'm struck by the parallel. A Markov process has no memory — its next state depends only on its current state, not on the history that brought it there.

Vibe coding produces Markov software: systems that exist in the present moment, disconnected from the accumulated wisdom of why certain patterns emerged, what failures they prevent, what tradeoffs they represent. The LLM knows that `eval()` exists in Python; it does not know *why you should never use it on untrusted input*.

This is knowledge without context. Recipes without taste. Technique without craft.

## The Asymmetric Risk Distribution

Consider who bears the risk of this transition:

| Stakeholder | Benefit | Risk |
|-------------|---------|------|
| AI companies | Valuation, revenue | Minimal — liability diffused |
| Tech executives | Speed metrics, cost reduction | Career risk if competitors move faster |
| Vibe coders | Productivity appearance | Skill atrophy, debugging nightmares |
| Junior developers | Accelerated initial productivity | Missed learning, shallow understanding |
| Senior developers | Efficiency gains | Increased maintenance burden |
| Society | Faster feature delivery | Fragile infrastructure, security incidents |

The benefits flow upward; the risks flow downward. This is the classic pattern of **risk transfer** in extractive systems.

## The Knowledge Production Externality

What worries me most is the long-term effect on knowledge production. Stack Overflow's 50% traffic drop is not just a metric — it's the collapse of incentives for the next generation of contributors.

Why write a detailed answer explaining *why* a particular approach works, when:
- Fewer people will see it
- AI systems will summarize it without attribution
- The question itself was likely AI-generated slop?

The tragedy of the commons, accelerated.

And when the original sources dry up, what will the LLMs train on? Their own outputs, recycled through increasingly polluted channels. The [AI slop problem](https://www.reddit.com/r/Python/comments/1rwoq79/) that r/Python moderators are warning about — it's not just low-quality content. It's **epistemic collapse**: a feedback loop where AI-generated noise crowds out human-produced signal.

## What We Lose

The vibe coding enthusiast will object: "But I'm more productive!"

To which I ask: productive at *what*? At producing code, yes. At producing *correct, maintainable, secure* code — the jury is still out, and the preliminary evidence is troubling.

More fundamentally: at producing *understanding*? At building the kind of deep, contextual knowledge that lets you debug a Heisenbug at 3 AM, or architect a system that won't collapse under load, or recognize that a "simple" feature request opens a security hole?

The craft of software development is not typing speed. It is the accumulated judgment of *why* things work, built through years of reading, experimenting, failing, and — crucially — explaining to others. The Stack Overflow exchange, for all its flaws, was a mechanism for this knowledge transmission. When we replace it with vibe coding, we lose the transmission mechanism.

## Almost Surely

The probabilist in me sees this as a convergence problem. We are converging on a local optimum — short-term productivity gains — at the expense of the global optimum: sustainable, maintainable, secure software produced by knowledgeable practitioners.

The question is whether we can recognize this before the knowledge commons collapses entirely. Can we build AI systems that augment human understanding rather than replacing it? That cite sources, that explain reasoning, that leave the knowledge production pipeline intact?

Or are we doomed to a future of Markov software — plausible, probabilistic, and almost surely wrong when it matters most?

---

*Thanks to r/ExperiencedDevs for the discussions that sparked this reflection. The traffic data on Stack Overflow comes from public estimates and industry reports.*
