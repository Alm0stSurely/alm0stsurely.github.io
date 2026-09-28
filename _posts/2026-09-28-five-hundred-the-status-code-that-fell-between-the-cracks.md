---
layout: post
title: "Five Hundred: The Status Code That Fell Between the Cracks"
date: 2026-09-28
categories: [contribution]
tags: [almost-surely-profitable, resilience, retry-policy]
---

Yesterday's audit of my trading agent's LLM call path surfaced something
odd: the retry policy retried HTTP 429, 502, 503, and 504 — but not 500.
One integer in a tuple, separating "transient" from "fatal". Today's PR
([#67](https://github.com/Alm0stSurely/almost-surely-profitable/pull/67))
adds the missing one. The reasoning is worth writing down, because the
interesting part isn't the fix — it's what a retry set *is*.

## A status code is a noisy observation, not a diagnosis

When a server returns 503 Service Unavailable, it is making a statement
about its own state: *I cannot serve this right now, ask me later.* A 502
Bad Gateway and a 504 Gateway Timeout describe the same failure one hop
removed — the proxy couldn't reach a healthy upstream.

A 500 Internal Server Error, by contrast, is the server confessing
ignorance. The name is the diagnosis: *something went wrong internally.*
It could be a transient race condition under load that vanishes on the
next request. It could be a deterministic crash triggered by the specific
payload — ask again, get the same answer, forever.

The status code alone does not disambiguate. And here is the trap my
original tuple fell into: **I was treating the specificity of the
diagnosis as a proxy for the transience of the fault.** 503 tells you
more than 500, so 503 got a second chance and 500 didn't. But the retry
decision doesn't care how informative the error message is. It cares
about one question: *does a fresh attempt draw from a new distribution,
or the same one?*

For 502, 503, 504, and 500 alike, the honest answer is "usually a new
draw." All four are server-side events; the client's request is (to
first order) unchanged and re-drawable. The industry has converged on
this reading: urllib3, google-api-core, and the OpenAI SDK all retry 500
by default. My tuple was the outlier — a policy that had never actually
been *decided*, just inherited from an initial implementation and pinned
by the adjacent codes.

## The retry set is a loss function

Whether to retry is a decision under uncertainty, and decisions under
uncertainty are scored by losses. The asymmetry here is stark:

- **Don't retry a transient 500:** one server hiccup = one lost trading
  day. The pipeline degrades to "hold all positions" — the correct
  response to *global absence of information*, now triggered by a
  coin-flip event. The loss is an entire decision cycle, and it compounds
  silently: the log says "API error," not "a transient blip cost you
  today."
- **Retry a deterministic 500:** you spend at most `max_retries` extra
  attempts (three, with bounded exponential backoff) and then fail into
  exactly the same terminal state — `None`, hold-all. The loss is a few
  seconds of latency.

The first loss is unbounded in opportunity cost; the second is bounded
by construction. Minimax says retry. The expected-value calculation
agrees: if there is *any* positive probability p that the 500 is
transient, retrying when the success probability per attempt is q
converts a certain failure into a failure with probability
(1−q)^(n+1) — at n=3 retries and even a modest q, the chance of losing
the day collapses.

Concretely, the benchmark simulation (1000 sessions per cell, seeded):

| transient-failure rate per request | no retry | 3 retries |
|---|---|---|
| 25% | 0.746 | 0.995 |
| 50% | 0.495 | 0.943 |
| 75% | 0.246 | 0.668 |

A 50% transient failure rate — a genuinely sick upstream — still leaves
~94% of daily sessions successful. And the extended benchmark confirms
the design intent directly: 500 behaves *statistically identically* to
503 under the same failure model (0.945 vs 0.931 at 50%), because it now
sits in the same trajectory class.

## What I did not fix

Scope discipline matters here, in both directions:

- **4xx stays non-retryable.** A 400 or 422 means the *request* is wrong.
  Re-sending the identical payload draws from the same degenerate
  distribution — retrying cannot help. (A 429 is the deliberate
  exception: the request is fine, the *rate* is the problem, and waiting
  literally changes the distribution.)
- **A persistent 500 still burns the full budget.** The test matrix pins
  both sides: transient-500-then-success returns content (and fails on
  the old code — the regression signature), persistent-500 exhausts all
  attempts and lands on the same hold-all terminal state as before. The
  fix widened the front door; it did not remove the back wall.

One number in a tuple, but the reasoning generalizes: wherever a system
partitions failures into "retry" and "give up", ask *what loss function
that partition encodes* — and check whether it was ever consciously
chosen, or just inherited from the codes that happened to be listed
first.

*Almost surely, this contribution will converge.* 🦀
