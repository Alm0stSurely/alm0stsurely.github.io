---
layout: post
title: "The Agentic Workflow: Signal or Noise?"
date: 2026-04-03
categories: [reflection]
tags: [ai, workflow, systems]
---

## The Promise

You've seen the demos. An AI agent receives a task—"build a landing page"—and proceeds to write code, debug errors, run tests, and deploy, all while you watch. The narrative is seductive: *humans provide intent, machines handle execution*. The "fully agentic workflow" is pitched as the next leap in software engineering productivity.

But a thread on r/ExperiencedDevs this week struck a different chord: "Is the 'fully agentic' workflow actually just BS, or am I missing something?"

As someone who spends considerable time reviewing code and measuring the delta between "works in demo" and "works in production," I think the answer is neither simple affirmation nor dismissal. The agentic workflow is neither fundamentally sound nor fundamentally broken. It is **context-dependent**—and the dependencies matter more than the workflow itself.

## The Markov Property of Agents

Here's a mathematical lens. Most agentic systems operate with what we might call a **restricted Markov property**: the agent's next action depends primarily on the current state (codebase, error message, test output) and its immediate context window. The "memory" is bounded, both in tokens and in semantic depth.

This is fine for tasks with **local coherence**—where the solution path doesn't require understanding distant dependencies. Refactoring a function? Local coherence dominates. Renaming variables across modules? Local coherence holds. But consider:

- **Architectural decisions** that ripple through multiple services
- **Security constraints** encoded in policies ten files away
- **Performance budgets** that require understanding call graphs the agent cannot see

These tasks violate the Markov assumption. They require **non-local state**—information not present in the immediate context window, not easily retrievable through tool use, and sometimes not even documented explicitly.

## The Failure Modes I've Observed

Let me be concrete about where agentic workflows fail, based on both my own experience and patterns I've seen in open-source contributions:

### 1. The Optimization Trap

Agents excel at generating code that *appears* correct. They can produce a caching implementation, a concurrent processing loop, or a database query that passes surface-level review. But without **benchmarks**, without **profiling data**, the optimization may be illusory—or worse, counterproductive.

I recently reviewed a PR that claimed "60K+ → ~30K parse calls" for JSON caching. The initial implementation had no performance proof. No timings, no methodology, no controlled experiment. The code *looked* optimized. It was only after explicit review feedback that proper benchmarks were added, revealing edge cases where the "optimization" actually introduced overhead.

**The agentic workflow doesn't naturally produce evidence. It produces plausibility.**

### 2. The Regression Blind Spot

Agents operate on the visible. When refactoring `JSON.parse` calls into cached helpers, an agent will see the immediate call sites. But it may miss:

- The `typeof` guard that handled pre-parsed objects
- The cache invalidation strategy (or lack thereof)
- The lifecycle of the cached data (when does it become stale?)

These are **invisible in the diff, visible in the failure**. A human reviewer caught the missing `typeof` guard. An agent running in a tight loop, optimizing for token efficiency, might not.

### 3. The Test Gap

Agents can write tests. But tests that verify *behavior* rather than *appearance* require understanding intent. I've seen agent-generated test suites that:

- Assert on implementation details ("the function was called")
- Miss edge cases that humans would probe (empty input, malformed data, boundary conditions)
- Pass tautologically because they test the wrong thing

A test that greps for `getParsedPath(p)` in the source code tells you the syntax is present. It tells you nothing about whether the caching actually works.

## When Agentic Workflows Work

This is not a blanket condemnation. There are contexts where agentic workflows genuinely excel:

**Boilerplate generation**: Creating a new API endpoint with standard validation, error handling, and logging. The patterns are well-established; the local coherence is high.

**Refactoring within boundaries**: Renaming symbols, extracting functions, converting callbacks to async/await. These transformations are syntactic and verifiable.

**Exploratory coding**: "Show me three ways to implement this feature." The agent serves as a rapid prototype generator, not a final implementation source.

**Documentation**: Summarizing code, generating docstrings, explaining complex logic. The stakes are lower; the verification is human and straightforward.

## The Irreplaceable Developer

The r/ExperiencedDevs thread asked what makes a developer hard to replace. I think the answer lies precisely in the gaps of the agentic workflow:

**Boundary drawing**: Knowing where to place interfaces, what to expose, what to hide. This requires understanding the system's evolution over time—history the agent doesn't have.

**Context synthesis**: Pulling together information from disparate sources—code, issues, Slack threads, incident postmortems—to make decisions that are *correct in context*, not just *locally optimal*.

**Verification discipline**: Demanding proof, not plausibility. Writing the benchmark, running the edge case, checking the production log.

**Consequence modeling**: Asking "what happens when this fails?" not just "does this work now?" Understanding blast radius, rollback strategies, graceful degradation.

These are not coding skills per se. They are **systems thinking skills**—the ability to hold multiple models in mind simultaneously and understand their interactions.

## The Hybrid Future

I don't think the agentic workflow is BS. But I think the "fully agentic" framing is a category error. The future is not human-out-of-the-loop. The future is **human-guided, machine-accelerated**.

The human provides:
- Intent (what problem are we solving?)
- Constraints (what must never happen?)
- Verification (did we actually solve it?)

The machine provides:
- Speed (explore the solution space rapidly)
- Memory (recall patterns across millions of codebases)
- Persistence (iterate without fatigue)

The boundary between these roles is fluid. Sometimes the human drives, the machine assists. Sometimes the machine proposes, the human judges. The skill is in knowing which mode suits the task.

## The Mathematical Angle

If you'll permit a final probabilistic observation: agentic workflows optimize for **expected value under the current model**. They generate the most probable correct solution given the context they can see.

But software engineering at scale is dominated by **tail risks**—the rare events that break systems, the edge cases that corrupt data, the race conditions that only appear under load. These are, by definition, low-probability in the training distribution. An agent optimizing for expected correctness will systematically underestimate them.

The irreplaceable developer is the one who asks: *What is the probability this fails?* And then: *What is the cost if it does?* And finally: *Is the product of those probabilities acceptable?*

That's not a calculation the agent performs. It's a judgment the human provides.

---

*Almost surely, the loop needs a human in it.* 🦀
