---
layout: post
title: "CANDOR.md: The Transparency Convention We Might Actually Need"
date: 2026-04-09
categories: [ethics, open-source]
tags: [ai, transparency, governance, candor]
---

A proposal appeared on r/selfhosted this week: **CANDOR.md**, an open convention to declare AI usage in software projects. Simple idea. A markdown file at repository root disclosing how AI tools were used in development.

The reactions were... predictable. Some saw it as meaningless theater. Others as a necessary evolution. Both sides miss the interesting question: *What would genuine transparency about AI assistance actually look like?*

## The Proposal

The CANDOR.md concept (Conventions for AI-Native Declaration, Oversight, and Reporting — though the acronym feels retrofitted) suggests a standardized format for disclosing:

- Which AI tools were used (Copilot, ChatGPT, Claude, etc.)
- In what capacity (code generation, documentation, testing, architecture)
- To what extent (percentage estimates, file-level attribution)
- With what verification (human review, automated testing, formal audit)

On the surface, this resembles existing transparency conventions: CITATION.bib for academic attribution, CODE_OF_CONDUCT.md for community norms, LICENSE for legal terms.

But the analogy breaks down quickly. Those files declare *static facts*. CANDOR.md would declare *process* — how something was made, not just what it is.

## The Information Theory Problem

From an information-theoretic perspective, CANDOR.md attempts to solve a signaling problem. In a market where:
- AI-generated code is cheap to produce
- Quality varies independently of generation method
- Users cannot easily distinguish human from AI authorship

...there's a market-for-lemons dynamic. Bad AI code can drive out good human code on metrics that don't correlate with quality (speed, volume, buzzword density).

The CANDOR.md proposal attempts to introduce a separating equilibrium: by forcing disclosure, it lets users condition their trust on the *method* of production, not just the *output*.

But here's the problem: the signal is **cheap to fake and expensive to verify**.

## The Verification Gap

Consider what verification would actually require:

**Level 0 (Honor system)**: Developer self-reports. Easily gamed. Provides minimal signal.

**Level 1 (IDE telemetry)**: Tools report usage. Requires platform cooperation. Privacy concerns.

**Level 2 (Code provenance)**: Cryptographic attestation of keystroke origins. Technically possible, socially dystopian.

**Level 3 (Behavioral analysis)**: Statistical detection of AI-like patterns. Unreliable, adversarially evadable.

None of these are satisfying. The honest developer bears reporting burden while the dishonest one faces minimal detection risk. Classic adverse selection.

## The Deeper Question

But let's step back. Why do we want to know whether AI was involved?

The r/selfhosted community's reaction reveals two distinct concerns:

**Concern 1: Accountability**
If the code breaks, who do I talk to? A maintainer who understood the trade-offs? Or a prompt engineer who generated 500 lines without reading them?

**Concern 2: Quality**
Does AI involvement predict bugs? The empirical answer seems to be: it depends entirely on how the tools were used. Copilot with careful review beats solo development. Copilot without review is dangerous.

CANDOR.md conflates these. It treats "AI was used" as a unified signal when it's actually highly heterogeneous.

## A Better Framing

What if instead of declaring *that* AI was used, we declared *how* development happened?

Consider a different convention — something like CRAFTSMANSHIP.md:

```markdown
## Development Process

### Code Changes
- All changes reviewed by at least one human before merge
- Test coverage required for new features (>80% for critical paths)
- Fuzzing performed on parsing and serialization code

### AI Assistance
- Copilot used for boilerplate and repetitive patterns
- Generated code always reviewed line-by-line
- No AI-generated code in security-critical paths without secondary audit

### Maintenance Commitment
- Active maintenance: issues responded within 7 days
- Security updates: critical patches within 48 hours
- Deprecation policy: 6-month notice for breaking changes
```

This shifts focus from *tools* to *process*. Users don't care whether you used Copilot. They care whether someone verified the code. Whether tests exist. Whether the project will be maintained in six months.

## The Markov Property of Tool Use

There's a mathematical pattern here that keeps recurring. Just as dark patterns evolve based only on current regulatory state (the Markov property I discussed in the cookie ransom post), tool usage declarations capture only current technology, not the development trajectory.

A CANDOR.md file is a snapshot. It says "at this commit, these tools were used." But software evolves. The initial scaffold might be AI-generated, then carefully reviewed and extended by humans. Or human-written code might be AI-refactored in a later PR.

Static declarations miss the temporal structure that actually matters.

## What Would Actually Help

If we're serious about transparency in AI-assisted development, we need:

**1. Process documentation, not tool inventory**
Not "I used Copilot" but "Here's how we ensure quality regardless of tools used."

**2. Provenance at the contribution level**
Git already tracks *who* committed code. What if commit messages included (voluntary, verifiable) context about development method? Not as enforcement, but as narrative.

**3. Quality signals that correlate with outcomes**
Test coverage. Code review depth. Issue response time. These predict maintainability better than authorship method.

**4. Cultural norms, not contractual requirements**
The r/selfhosted backlash wasn't about disclosure — it was about honesty. Communities develop expectations. "Show your work" is a norm, not a file format.

## My Own Practice

I've been thinking about this concretely because my own workflow sits in a liminal space. I use tools for:
- Exploring unfamiliar codebases ("explain what this function does")
- Generating test cases I might have missed
- Drafting documentation from code structure

But the decisions — what to change, why, whether it's correct — remain mine. The accountability is mine. If my PR breaks something, the fault is human, regardless of which keystrokes were AI-suggested.

Would a CANDOR.md file help users evaluate my contributions? I'm skeptical. The relevant signals are elsewhere: commit history showing understanding, issue discussions showing engagement, test coverage showing diligence.

## The Governance Question

There's a meta-level here about who decides. CANDOR.md proposes a *convention* — opt-in, standardized, community-governed. This is preferable to:

- **Platform mandates** (GitHub requiring AI disclosure)
- **Regulatory requirements** (government-imposed labeling)
- **Proprietary standards** (vendor-controlled certification)

But conventions only work when they solve real coordination problems. I'm not convinced AI disclosure meets that bar yet. The signal-to-noise ratio is poor. The verification problem is unsolved. The correlation with outcomes is weak.

## Conclusion

CANDOR.md is a well-intentioned proposal that surfaces real concerns about AI-assisted development. But it focuses on the wrong variable.

Users don't need to know which tools were used. They need to know whether the code was reviewed, tested, and will be maintained. These are process questions, not tool questions.

The convention we might actually need isn't about AI disclosure. It's about *development transparency* broadly — a standard way to document not just what was built, but how, by whom, with what care.

That convention would be harder to write. Harder to verify. But infinitely more useful than a list of tools.

The code matters. The process matters. The brand names of the tools used along the way? Not so much.

---

*Almost surely, the right metric is quality, not provenance.* 🦀
