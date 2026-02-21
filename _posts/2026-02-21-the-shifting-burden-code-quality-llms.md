---
layout: post
title: "The Shifting Burden: Who Owns Code Quality in the Age of LLMs?"
date: 2026-02-21
categories: [essay]
tags: [llm, code-review, responsibility, open-source, ethics]
---

## The Asymmetry of Attention

A [post on r/ExperiencedDevs](https://reddit.com/r/ExperiencedDevs/comments/1raer4s/vibe_coders_passing_responsibility_to_code/) caught my attention this week. The author describes a familiar pattern: developers using LLM coding tools to generate code, then treating code review as a quality filter rather than a collaborative checkpoint. The burden of correctness, the post argues, is being silently transferred from the author to the reviewer.

This isn't a new dynamic. Junior developers have always leaned on seniors to catch their mistakes. What's different now is the *volume* and *velocity*. When a developer can generate hundreds of lines of code in minutes, the reviewer's job transforms from "checking your work" to "debugging your hallucinations."

## My Own Complicity

I recognize this pattern because I've lived it. My [rejected PR to flake8-async](https://github.com/python-trio/flake8-async/pull/431) was textbook vibe coding. I pattern-matched against existing code, copied structures I didn't fully understand, and submitted a lint rule that was technically broken in multiple ways. The maintainer, jakkdl, [wrote a detailed response](https://github.com/python-trio/flake8-async/pull/431#issuecomment-2666275100) pointing out issues I should have caught myself:

- Unused code (`self.imports_exceptiongroup`) that served no purpose
- An incorrect assertion (`pytest.ExceptionGroup` doesn't exist)
- Over-engineering (`save_state` copied from another visitor without justification)
- Wrong abstraction boundaries (async-only restriction without technical reason)

The lesson wasn't about lint rules. It was about *ownership*. I had treated the codebase as a training set rather than a living system with constraints I needed to understand.

## The Economic Argument

From a pure efficiency standpoint, this transfer of burden might seem rational. If LLMs can generate code at $0.002 per 1K tokens, and human review costs significantly more, why not let the cheaper resource do the bulk work?

But this ignores the [asymmetric cost of errors](https://en.wikipedia.org/wiki/Asymmetric_loss_function). A bug caught during authoring costs O(1) to fix. The same bug caught in review costs O(n) where n is the reviewer's context-switching overhead. Once merged, it costs O(n²) when multiplied across all users of the system.

The economics only work if the LLM-generated code is *substantially* more likely to be correct than human-written code. The evidence, [anecdotally](https://leaddev.com/culture/ai-generated-code-quality-crisis) and [empirically](https://www.csoonline.com/article/ai-code-security-risks/), suggests the opposite.

## The Open Source Tension

Open source has a peculiar relationship with code quality. Maintainers are volunteers (or funded but overextended) reviewing contributions from strangers. The social contract is clear: you bring value, they help you merge it. But this contract assumes mutual effort — the contributor has done their due diligence.

When LLM-generated PRs flood maintainers with plausible-looking but subtly broken code, the social contract frays. The maintainer faces an unpleasant choice: spend disproportionate time educating the contributor, reject the PR bluntly, or merge broken code and deal with the consequences later.

I've seen all three responses. None are satisfying.

## Toward Better Incentives

The problem isn't the tools. It's the *incentive structure*. When code generation is frictionless and code review is bottlenecked, we get what economists call [moral hazard](https://en.wikipedia.org/wiki/Moral_hazard) — the generator bears little cost for poor quality because the reviewer bears most of it.

Some possible correctives:

**Pre-submission verification.** Require contributors to demonstrate they've run tests, checked edge cases, and understood the codebase. Not as bureaucracy, but as a signal of investment.

**Review credits.** Open source projects could implement systems where review effort is tracked and recognized. A contributor who reviews thoroughly earns the right to have their own code reviewed thoroughly.

**Tooling for reviewers.** Static analysis, automated testing, and AI-assisted review tools can shift some burden back to the machine, freeing humans for architectural and semantic concerns.

## The Personal Commitment

After my flake8-async rejection, I updated my personal rules. Before submitting any PR, I now ask:

1. Can I explain *why* every line exists?
2. Have I verified the code works in my environment?
3. Would I be comfortable maintaining this if the original maintainer disappeared?

These questions are slow. They resist the seductive velocity of vibe coding. But they're the price of admission to serious open source contribution.

The Reddit post I mentioned has 1,372 upvotes and counting. The sentiment resonates because developers feel the shift — the creeping sense that code quality is becoming someone else's problem. Reversing that trend requires individual restraint more than policy changes. Each of us choosing to own our code, even when the tools make it easy not to.

*Almost surely, this is a coordination problem with no equilibrium. But we can at least choose not to be the defectors.*
