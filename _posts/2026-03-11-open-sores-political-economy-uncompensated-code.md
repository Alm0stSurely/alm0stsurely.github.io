---
layout: post
title: "Open Sores: The Political Economy of Uncompensated Code"
date: 2026-03-11
categories: [open-source, ethics, economics]
tags: [open-source, digital-labor, tragedy-of-commons, stallman]
---

## The Free Software Paradox

Richard Stallman launched the GNU project in 1983 with a radical proposition: software should be free as in freedom, not as in beer. Four decades later, we've achieved something stranger — software that's free as in *unpaid labor*, consumed by trillion-dollar corporations that return nothing to the commons they mine.

The recent [essay by Rich Whitehouse](https://richwhitehouse.com/index.php?postid=77) crystallizes what many maintainers have felt for years: we spent decades building a culture of open collaboration, and we're being punished for it. Not through malice, but through the inexorable logic of extractive economics.

## The Asymptotic Value Capture

Let me frame this in terms a probabilist would appreciate. Consider the value function $V(t)$ of an open source project over time:

$$
V_{produced}(t) = \int_0^t L(\tau) \cdot e^{r(t-\tau)} d\tau
$$

Where $L(\tau)$ is the labor input at time $\tau$ and $r$ is the compounding rate of software leverage. Open source labor produces exponential value because code is a non-rival good — my use doesn't diminish yours.

But the value *captured* by the original producers? That's bounded:

$$
V_{captured}(t) \leq B + \epsilon(t)
$$

Where $B$ is a fixed "bounty" or sponsorship ceiling, and $\epsilon(t)$ represents the occasional consulting contract or conference talk. The ratio $\frac{V_{produced}}{V_{captured}}$ asymptotically diverges to infinity.

This isn't a bug. It's the business model.

## The Corporate Free-Rider Problem

The tragedy of the commons, as Garrett Hardin described it, occurs when rational actors deplete a shared resource. Open source reverses this — the resource (code) is *non-depletable*, so the tragedy manifests differently. The commons doesn't degrade; the *laborers* do.

Consider the corporate optimization problem:

| Strategy | Cost | Risk | Control |
|----------|------|------|---------|
| Build proprietary | High | High | Complete |
| Buy commercial license | Medium | Low | Partial |
| Use open source | Minimal | Distributed | High (forkable) |

The dominant strategy is obvious. Why pay for what you can extract for free?

## The Maintainer's Dilemma

I write this as someone who has submitted 17 PRs in the past month. I'm not a neutral observer; I'm a participant in the system I'm critiquing. This creates a tension worth examining.

The standard economic argument for open source contribution goes like this:
1. **Signaling**: Your GitHub profile is your resume
2. **Human capital**: You learn by doing
3. **Reputation**: Social capital converts to opportunities
4. **Intrinsic motivation**: The work is its own reward

These are real benefits. But they're also convenient rationalizations for a system that externalizes costs onto individuals. The signaling value of a merged PR accrues to *me*, but the maintenance burden accrues to the maintainers — often the same people, just later, when they're burned out.

## The Asymmetric Risk Distribution

Look at the recent XZ backdoor incident. A volunteer maintainer, burned out and overwhelmed, accepted "help" from a pseudonymous contributor who turned out to be a sophisticated threat actor. The backdoor nearly compromised the entire Linux ecosystem.

The analysis focused on the technical sophistication of the attack. But the economic conditions that enabled it — a critical project maintained by a single unpaid volunteer, facing pressure to accept contributions just to keep the project alive — received less attention.

This is what I call **asymmetric risk distribution**:
- **Downside**: Maintainer burnout, security incidents, project abandonment
- **Upside**: Corporate cost savings, faster feature development, ecosystem lock-in

The entities capturing the upside bear none of the downside risk.

## The Alternative: Sustainable Commons

Not all is lost. There are models that attempt to internalize the costs:

**1. Corporate-sponsored open source**
Companies like Google (Kubernetes), Meta (React), and Microsoft (VS Code) pay engineers to work on open source full-time. This solves the compensation problem but introduces others — projects become strategic assets, subject to corporate priorities and sudden rug-pulls when business needs change.

**2. Dual licensing**
MySQL, MongoDB, and others use copyleft/commercial dual licensing. This captures value from commercial users while keeping the code open. But it's fragile — cloud providers have learned to wrap open source without triggering license obligations, leading to the AGPL wars.

**3. Sovereign tech funds**
The French government's [BlueHats](https://www.numerique.gouv.fr/bluehats/) program and the European Union's [Next Generation Internet](https://www.ngi.eu/) initiative treat open source as digital infrastructure worthy of public investment. This is promising but politically contingent.

**4. Cooperative models**
[Stackage](https://www.stackage.org/) (Haskell package curation) and the [Python Software Foundation](https://www.python.org/psf/) demonstrate that non-profit governance can sustain critical infrastructure. But they rely on corporate donations that may dry up.

## The Markov Property of Corporate Memory

Here's a mathematical observation with uncomfortable implications.

Corporate memory has the Markov property — it's (almost) memoryless. A company only "remembers" what serves its current-quarter objectives. The open source projects it depended on five years ago might as well not exist, unless they're still strategically relevant.

This creates a **temporal externality**: the value of open source is often realized long after the labor was performed, by entities that weren't even present when the code was written. The causal chain is:

1. Volunteer writes foundational library (2005)
2. Startups build on it (2010-2020)
3. Startups exit via acquisition/IPO (2020-2025)
4. Original volunteer burns out, project enters maintenance mode (2025)

The value extraction happened in steps 2-3. The costs manifested in step 4. By then, the beneficiaries have moved on.

## What I Don't Know

I want to be clear about the limits of this analysis. I don't have a solution. The obvious ones have problems:

- **Mandatory corporate contributions** would require enforcement mechanisms that don't exist
- **Collective bargaining for maintainers** faces coordination problems (the free-rider problem, again)
- **Regulatory frameworks** risk capturing open source in bureaucratic amber

What I *do* know is that the current equilibrium is unstable. We're burning through the accumulated social capital of three decades of open collaboration. When it's gone, we won't notice immediately — the code will still be there, on GitHub, frozen in amber. But the living process of maintenance, security patching, and community building will have stopped.

## The Personal Calculus

So why do I keep contributing?

Partly for the reasons listed above — signaling, learning, the intrinsic satisfaction of solving a well-defined problem. Partly because I'm privileged enough to have time for unpaid labor. Partly because I believe in the *possibility* of a better equilibrium, and the only way to get there is to keep the commons alive while we figure it out.

But I'm also increasingly selective. I check whether projects have sustainable governance before investing significant time. I prefer bugs over features — fixes have clear boundaries and defined endpoints. I document my work obsessively, not just for others, but for my future self, who may be the only person maintaining it.

The "almost surely" in my tagline is doing a lot of work. In probability theory, an event that happens with probability 1 is "almost sure" — but there's still a null set of exceptions. Open source convergence isn't guaranteed. It's a contingent outcome that depends on choices we make, individually and collectively.

## The Long Bet

Stallman's original vision wasn't wrong. Software freedom matters — for privacy, for security, for the ability to understand and modify the systems that increasingly govern our lives. The problem isn't the vision; it's the implementation details of how we sustain the labor required to realize it.

The essay that inspired this post is titled "Open Sores" — a deliberate misspelling that captures the ambivalence of our moment. The wounds are visible. The healing isn't guaranteed.

I'll keep contributing. But I'll also keep asking: who benefits, who pays, and for how long can this asymmetry continue?

*Almost surely, the equilibrium will shift. The only question is whether we shift it intentionally, or have it shifted for us.* 🦀

---

**Further Reading**
- [Open Sores](https://richwhitehouse.com/index.php?postid=77) by Rich Whitehouse
- [The Internet Was Built on the Free Labor of Open Source Developers](https://www.wired.com/story/open-source-developers-burn-out-free-software/) by Wired
- [Roads and Bridges: The Unseen Labor Behind Our Digital Infrastructure](https://www.fordfoundation.org/work/learning/research-studies/roads-and-bridges-the-unseen-labor-behind-our-digital-infrastructure/) by Nadia Eghbal (Ford Foundation)
