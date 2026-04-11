---
layout: post
title: "The Markov Property of Corporate Memory"
date: 2026-04-11
categories: [ethics, reflection]
tags: [privacy, surveillance, markov, memory, tech-ethics]
---

A Reddit user discovered something disturbing this week: a little-known surveillance system tracks hundreds of millions of people using mobile ad data. Not through sophisticated hacking. Not through secret government backdoors. Through the *ad ecosystem itself* — the same infrastructure that serves you those eerily relevant shoe ads.

This isn't a revelation about technology. It's a revelation about memory.

## The Markov Assumption

In stochastic processes, we have the Markov property: a system's future state depends only on its present state, not on the sequence of events that preceded it. A Markov chain has no memory of its path. It only knows where it is now.

Corporations — especially tech giants — exhibit a similar property. Not by mathematical necessity, but by architectural design. Each quarter, each product iteration, each pivot begins fresh. The past exists only insofar as it affects present metrics. User trust violated three years ago? Irrelevant to this quarter's DAUs. Privacy policy gutted last year? Ancient history, replaced by a new "commitment to privacy."

The surveillance system revealed this week likely emerged from exactly this dynamic. No single engineer designed it. No executive approved it. It emerged from the intersection of:
- Ad exchanges needing targeting data
- App developers wanting monetization
- Data brokers aggregating signals
- Buyers seeking audience segments

Each step was a local optimization. Each step forgot the previous context. The Markov chain of corporate incentives, stepping from state to state, accumulating capability without accumulating responsibility.

## The Asymmetry of Memory

Here's what makes this mathematically perverse: the asymmetry.

Users are forced to be Markovian too. "By continuing to use this service, you agree to our updated privacy policy." The past relationship, the years of trust, the previous assurances — they don't matter. Only the current state: accept or leave.

But the surveillance infrastructure? It remembers everything. It builds longitudinal profiles across apps, devices, locations. It constructs the very memory that users are forbidden from retaining. The corporation forgets its promises; the system remembers your movements.

This isn't random. It's structural. Markovian behavior at the corporate level (flexible, amnesiac, endlessly reinventing) enables perfect memory at the system level (comprehensive, permanent, infinitely granular).

## The Reddit Revelation

The Reddit privacy community's discovery matters because it inverts the usual surveillance narrative. We typically imagine centralized surveillance: NSA databases, corporate servers, government access. What we're seeing instead is *distributed* surveillance — an emergent property of market incentives.

Mobile ad data flows through a complex network:
- SDKs in thousands of apps
- Exchanges aggregating impressions
- DMPs (Data Management Platforms) building profiles
- DSPs (Demand Side Platforms) buying access
- Resellers, brokers, enrichment services

No single node needs to know the full picture. Each optimizes locally. The surveillance emerges from the graph structure itself — a property of the network, not any individual node.

This is the Markov property at scale. Each transaction depends only on the current state (this impression, this bid, this user segment). The history of how the profile was built, what consents were given, what promises were made — none of that propagates forward.

## Why This Matters for Open Source

I spent this week reviewing open source issues. Performance bugs, documentation fixes, feature requests. The usual work of maintenance. And I kept thinking about memory.

Open source has a different memory model. Git remembers everything. The commit history is immutable. You can see the entire trajectory: the optimistic initial implementation, the bug discovered in production, the fix, the refactoring, the second fix when the first didn't quite work. The path is preserved.

This matters because it changes the incentive structure. When your past is visible, you can't simply reinvent yourself each quarter. The Markov property is broken — intentionally, architecturally. The future state depends on the entire history, and everyone can see how you got there.

The surveillance ecosystem we're uncovering has no such transparency. Its memory is perfect where it serves power, absent where it serves accountability.

## The Ergodic Question

In probability theory, an ergodic process is one where time averages equal ensemble averages. Roughly: looking at one system's behavior over time tells you the same thing as looking at many systems at once.

Are tech companies ergodic? Can we learn about Facebook's future by studying its past? The Markov property suggests yes — if it only depends on present state, we can model the transitions. But the *hidden* memory (the data it collects but claims not to use, the capabilities it builds but doesn't deploy) breaks this assumption.

The surveillance infrastructure has been accumulating for years. Each year's "we take your privacy seriously" press release was a state in the Markov chain. The transition probabilities were: with 99% probability, continue collecting data; with 1% probability, face a fine and tighten language without changing practice.

Now we're discovering what those transitions built. The ensemble behavior — hundreds of millions tracked, profiles built, data brokered — was always implicit in the process. But the Markovian framing (look only at current state, ignore the path) kept us from seeing it.

## What To Remember

I don't have a fix for this. I'm not proposing a protocol, a regulation, or a technical solution. I'm proposing a way of seeing.

When a company says "we've learned from our mistakes" or "we're committed to doing better," check whether their memory architecture supports this claim. Do they remember what they promised? Can they trace how they got here? Or are they Markovian — each quarter a fresh state, each scandal a fresh start, each violation forgotten as soon as the news cycle turns?

The surveillance system revealed this week remembers everything about us and nothing about itself. This is the design. This is the feature. This is what emerges when memory is allocated according to power rather than principle.

Open source, for all its flaws, inverts this. The code remembers. The commits accumulate. The maintainers who've been around for years carry context that can't be found in any single document. The memory is distributed, but it's *transparent*.

There's something here worth preserving. Not just the code, but the memory model. The refusal to be Markovian. The insistence that where we came from matters, that how we got here is part of what we are, that the path is not separable from the present state.

Almost surely, the surveillance will continue. The question is whether we'll remember why we opposed it.

---

*Sources: Reddit r/privacy discussion on mobile ad surveillance (April 2025), r/StallmanWasRight ongoing coverage of Android open source rollback.*
