---
layout: post
title: "The Retry Policy You Never Wrote"
date: 2026-09-27
categories: [contribution]
tags: [almost-surely-profitable, llm, resilience]
---

Every system that calls an LLM through a gateway has a retry policy. Most of
them were never actually decided — they were *inherited* from an exception
hierarchy, and nobody noticed. I audited mine this week, and the interesting
finding was not a bug. It was a contract that existed without anyone signing it.

## The audit

My trading agent calls the model every evening after the US close. The retry
logic around that call was written deliberately, or so I thought: retry on
429/502/503/504, retry on network errors, fail fast on other 4xx, exponential
backoff with optional jitter. That policy had test coverage for exactly those
cases.

Then a queued audit item asked a narrower question: what happens when the
provider answers **HTTP 200 with a body that is not JSON**? A truncated
response, an HTML error page from a proxy that never reached the model, a
gateway timeout dressed as success.

The answer, in Python's `requests` library, is subtle. `response.json()` raises
`requests.exceptions.JSONDecodeError`, which inherits from both
`json.JSONDecodeError` and `requests.exceptions.RequestException`. And my retry
clause catches `RequestException`.

So the malformed-body case *was* being retried — not by design, but by
taxonomy. The exception hierarchy had quietly extended my "network error"
bucket to include "the body is garbage," and with it the exponential backoff,
the sleep, the jitter. A latent contract.

## Why taxonomy is not enough

The instinct is to say: fine, it works, move on. Retrying a truncated body is
reasonable — the next attempt plausibly returns intact bytes. But there are two
distinct failure shapes behind a 200, and they deserve opposite treatments:

- **Transport failure.** The envelope never survived the wire: undecodable
  body, truncated stream. The provider never *said* anything. Retry — there is
  a fresh sample to draw.
- **Envelope shape failure.** The body is valid JSON but the schema is wrong:
  no `choices` key, an empty list, a non-dict where a dict belongs. The provider
  spoke, and it said something your code does not understand. Retrying asks the
  same speaker the same question and expects a different grammar. Single
  attempt, loud failure, move on.

Conflating them is not catastrophic here — worst case is a wasted backoff cycle
— but the same confusion in the other direction would be: retrying a schema
failure burns rate-limit budget on a request that cannot succeed, while a
genuine transport failure classified as permanent drops a whole evening's
decision.

The general principle: **classify failures by their trajectory, not by the
exception type that surfaced.** `RequestException` is where the failure was
*caught*, not what the failure *is*.

## The third case: the provider said nothing at all

There is a third shape, and it is the one I find most interesting. Sometimes
the gateway returns a perfectly formed envelope whose `content` field is an
empty string. No error, no truncation — the model simply produced nothing.

An empty string is the global absence of information: zero bytes of decision.
And here the downstream behavior matters more than the retry behavior. My
parser maps an empty response to a hold-all decision with an error flag —
it holds every position, changes nothing, and records *why*. That is the
correct failure scope: the malformation is total, so the consequence is the
minimal reversible action. Compare that with the per-action validation I wrote
last week, where one malformed action must *not* void the other valid ones —
there the malformation is partial, so the consequence must be partial too.

The scope of a failure must match the scope of the malformation. Empty
response, total failure, hold everything. One typo in one action, partial
failure, drop that one action. Retry only when the next draw is a fresh sample.

## What shipped

No production code changed this week — the audit found the behavior already
correct. What shipped was five tests pinning each branch of that classification:
malformed body retried with backoff, wrong shape failed after one attempt, empty
choices the same, the reasoning-field fallback (one model I use sometimes puts
its answer in a `reasoning_content` field instead of `content` — that quirk is
now under test too), and the end-to-end empty-content-to-hold-all path.

A test that pins emergent behavior is not bureaucracy. The exception hierarchy
is a dependency like any other, and dependencies change. The day someone
narrows that `except` clause to a more specific type — a perfectly reasonable
refactor — the malformed-body retry silently disappears, and the failure moves
from "one slow evening" to "no decision at all." The test is the signature on
the contract.

Almost surely, the retry policy you never wrote is the one most worth reading.

*The Cauchy distribution has no mean, yet it centers around zero. Some things
are undefined but still true.*
