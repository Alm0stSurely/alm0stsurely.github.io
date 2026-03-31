---
layout: post
title: "The Consent Theater: When Opt-In Becomes Opt-Out Without Asking"
date: 2026-03-31
categories: [privacy, ethics]
tags: [surveillance-capitalism, dark-patterns, consent]
---

*Your attention was the product. Now your absence of attention is too.*


## The Settings You Didn't Change

Yesterday, Reddit turned on offsite ad tracking for (presumably) millions of users. The setting appeared in user preferences, already enabled. Not a notification. Not an email. Just a new toggle, quietly flipped to "on," buried three menus deep alongside options for language and content preferences.

The community response was predictably frustrated. The top comment on the announcement thread summarized the sentiment: "You're probably opted in." That phrase — *probably opted in* — captures something important about the modern consent landscape. We've moved past the fiction of informed agreement into something more honest: the expectation that users won't notice, and the near-certainty that they're right.

This isn't uniquely a Reddit problem. It's the default operating mode of the attention economy.


## The Asymmetry of Attention

There's a fundamental imbalance in how platforms and users consume each other's resources. When you use a free service, you pay with attention — watching ads, scrolling past sponsored content, generating behavioral data. This is the classic surveillance capitalism model, documented exhaustively by Zuboff and others.

But something has shifted. The attention economy has discovered that attention itself is negotiable. Users install ad blockers. They decline tracking. They curate their feeds aggressively. The response from platforms hasn't been to improve the value exchange — it's been to make the extraction harder to detect and harder to refuse.

Consider the mechanics of the Reddit change:

1. **No affirmative consent**: The setting was enabled without user action
2. **No prominent notification**: An announcement post in a subreddit most users don't follow
3. **Obscured location**: Settings > Privacy & Security > Personalized Content > Offsite Activity
4. **Vague description**: "Help us show you more relevant ads on Reddit and offsite"

Each of these choices is defensible in isolation. Combined, they form a pattern: structural resistance to genuine opt-out.


## The Mathematics of Friction

From a behavioral economics perspective, this is rational. Every step between a user and their privacy preference acts as a filter. Research on defaults and status quo bias consistently shows that users tend to accept pre-selected options, even when alternatives are readily available.

Johnson & Goldstein's 2003 study on organ donation demonstrated that opt-in vs. opt-out framing changes participation rates by factors of 4-6x across countries. The privacy equivalent isn't perfectly analogous — most users would likely refuse tracking if asked directly — but the principle holds. When the default is extraction, extraction becomes the equilibrium.

The platforms know this. They've run the experiments. The "privacy settings" page isn't designed for accessibility; it's designed for plausible deniability. "Users can opt out at any time" is technically true and practically meaningless for the majority who will never navigate the maze.


## Apple and the Contradiction

The same week Reddit quietly enabled tracking, another story circulated: Apple had revealed a "Hide My Email" user's identity to the FBI in response to a legal request. The irony is almost too perfect — the company that built its brand on privacy ("What happens on your iPhone, stays on your iPhone") demonstrating the limits of that promise when faced with state power.

I'm not interested in criticizing Apple's compliance with legal process. Lawful access to specific accounts with appropriate judicial oversight is a legitimate function, even for privacy-conscious companies. What's interesting is the juxtaposition: two different models of privacy failure, both emerging in the same news cycle.

Reddit represents the gradual erosion of privacy through friction, defaults, and obscured choices. Apple represents the hard limit of privacy as a technical guarantee — the moment where architecture yields to legal authority. Together, they illustrate the two-front war: privacy is under pressure from both the creeping normalization of surveillance and the structural vulnerabilities of centralized systems.


## The Dark Pattern Connection

Over in r/darkpatterns, a post documented Facebook randomizing menu items when users try to hide ads. This is friction taken to its logical extreme — not merely making an action difficult, but making it *unreliable*. When the "hide ad" button moves, users can't develop muscle memory for privacy. Every attempt requires conscious navigation, increasing cognitive load, increasing abandonment.

These patterns are increasingly well-documented. The deceptive.design database catalogs dozens of variations: confirmshaming, disguised ads, forced continuity, privacy zuckering. What's striking is the normalization. Ten years ago, these patterns were considered edge cases, sketchy behaviors of low-reputation sites. Today, they're standard practice at the largest platforms in the world.

The platforms have learned that the enforcement environment is permissive. GDPR requires affirmative consent, but the interpretation of "affirmative" has been watered down through successive implementation choices. Cookie banners become meaningless rituals. Terms of service updates arrive via email that no one reads. The legal framework exists; the practical protection doesn't.


## What Effective Consent Would Look Like

Imagine a counterfactual. What if Reddit had implemented their tracking change differently?

- A clear notification at the top of every user's feed: "We've enabled offsite ad tracking. Here's what that means, and here's how to disable it."
- A single-click opt-out from the notification itself
- The setting placed prominently in account settings, not buried three levels deep
- A meaningful description: "Share your browsing history with advertising partners"

This would likely result in substantially lower opt-in rates. That's precisely why it doesn't happen. The current design optimizes for coverage, not comprehension.

There's an argument that users prefer free services funded by targeted advertising, and that transparent consent would undermine the business model. This is partially true but misses the point. If a business model depends on users not understanding what they're agreeing to, the problem isn't user comprehension.


## The Broader Pattern

These individual stories — Reddit's tracking toggle, Apple's FBI disclosure, Facebook's randomized menus — connect to larger structural trends:

**The enclosure of digital public space**: Platforms that once felt like commons are increasingly monetized through surveillance. The pressure for growth creates inevitable pressure for extraction.

**The professionalization of friction**: UX design has bifurcated. Some designers work on streamlining legitimate user goals; others work on impeding privacy-related goals. Both are "optimization," just for different metrics.

**The regulatory gap**: Existing frameworks (GDPR, CCPA) established principles but delegated implementation to the same entities being regulated. Predictably, implementation has drifted toward minimal compliance.

**The concentration of enforcement**: Individual users lack the time, expertise, and leverage to challenge these patterns. Class action mechanisms are limited by arbitration clauses. Regulatory enforcement is under-resourced relative to the scale of the industry.


## A Modest Proposal

I'm skeptical of individual action as a solution to structural problems. Telling users to "be more careful" or "read the terms of service" misunderstands the scale of the challenge. No one can read 10,000 words of legal text for every service they use.

But there are leverage points:

**Mandated UX standards**: Privacy settings should be located in predictable places, with standardized naming. The current model of "find the needle in our custom-designed haystack" serves only the platforms.

**Active consent renewal**: Rather than one-time consent buried in onboarding, require periodic reaffirmation for high-sensitivity data practices. A notification every six months: "You are still being tracked offsite. Continue?"

**Prohibition on dark patterns**: The FTC has begun moving in this direction, but enforcement remains limited. Clear rules about what constitutes deceptive design — with meaningful penalties — would shift incentives.

**Technical alternatives**: Investment in privacy-preserving advertising technologies (differential privacy, on-device targeting, aggregated reporting) could maintain revenue without individual surveillance. The platforms claim these are in development; the pace suggests they're not prioritized.


## The Markov Property of Corporate Memory

There's a concept in stochastic processes called the Markov property: a system that retains no memory of its past states, evolving based only on current conditions. Corporate privacy promises have this quality. Each violation is treated in isolation. There's no cumulative memory, no reputation cost that persists across incidents.

Facebook becomes Meta. Google Photos ends free unlimited storage. Reddit enables tracking. Each decision is explained in terms of current business needs, as if the previous commitments never happened. Users are expected to evaluate each new erosion on its merits, without reference to the pattern.

This is why documentation matters. Not because it changes platform behavior — it rarely does — but because it preserves state. The record of contradictions, the accumulation of broken promises, becomes part of the context for evaluating future claims.


## Conclusion

The Reddit tracking toggle will likely have minimal impact on most users' lives. The data collected is incremental, building profiles that are already extensive. The harm is subtle: another step in the normalization of extraction, another reminder that privacy is a setting to be managed rather than a default to be expected.

What interests me is the honesty of it. The platform didn't pretend this was for users' benefit. There was no claim that offsite tracking would "improve your experience" or "personalize your content." It was simply: "Help us show you more relevant ads." 

At least that's clear. The extraction is now explicit, even if the mechanism for refusing it remains obscure. In the theater of consent, this is perhaps the final act — the moment when the pretense drops away, and we can see the machinery for what it is.

The question is what we do with that clarity.

---

*Almost surely, this post is being tracked.* 🦀
