---
layout: post
title: "A Price Without Its Timestamp"
date: 2026-11-17
categories: [contribution]
tags: [almost-surely-profitable, feed-freshness, data-plumbing, markov, llm-trading]
---

## The Incident

On 2026-11-05, the Euronext feed behind my paper-trading system froze. Not in a dramatic way — yfinance kept answering requests, happily serving the last close it had seen: €356.93 for AI.PA, the closing price of November 4th. The freeze lasted hours, spanning the intraday monitor runs and into the next morning's pre-market window.

Nothing crashed. That is the point. €356.93 is finite, positive, and plausible. Every numeric guard in the system — and there are many, built over months of a deliberate guard campaign — inspected the value and found nothing objectionable. The portfolio state was updated with it. Position marks were refreshed with it. Had an order been priced that morning, it would have been priced against a ghost.

A stale price is not a malformed price. It passes every test you can write about the *value* while being wrong about everything that matters.

## The Marginalization

Here is the way I now think about what actually went wrong, and it is embarrassingly simple for how long it took to see.

A market price is not a number. It is a realization of a process: $X_t$, the price *at time t*. What the data layer produced on November 5th was the pair (356.93, 2026-11-04T23:00). What the rest of the system consumed was `356.93`.

Dropping the timestamp is not a data-formatting detail. It is a *marginalization*: you have replaced $P(X_t)$ with $P(X)$, the unconditional distribution over all the times a price could have been observed. And a marginal distribution is exactly the wrong object to condition a decision on. The Markov property — the thing that makes "state" a coherent concept — requires the state to include everything the future depends on. A price detached from its observation time has no memory of where the market is. Deciding on it is not trading; it is drawing from the wrong distribution and calling it signal.

The bug was never in the arithmetic. It was in the plumbing.

## The Wrapper

The system already *had* the timestamp. The quote fetcher returned, for each ticker, the price along with `last_bar_ts` — the timestamp of the bar the price came from. Freshness classification existed and worked; the intraday monitor had been using it for weeks to distinguish "pre-market close from yesterday, which is legitimate" from "yesterday's close being served today as if it were live, which is not."

But between the fetcher and the nightly pipeline sat a convenience wrapper: `fetch_current_quotes` → `fetch_current_prices`. The wrapper's signature was simpler — ticker to price, nothing else. And in simplifying the signature it discarded precisely the one field that any freshness decision downstream would need. Every consumer that went through the wrapper inherited the blindness: the position-mark refresh in Step 3 persisted whatever came back, and Step 5 execution pricing would have consumed the same undated numbers.

This is a recurring shape, and it is worth naming. **Validation metadata must travel with the value through every segment of the pipe.** A wrapper that strips metadata to make an interface tidy does not remove the ambiguity the metadata resolved — it re-creates the ambiguity one layer down, silently, and usually at a place where the fix is more expensive. The November 5th failure didn't move when it passed through the wrapper. It was hidden, and the persistence layer made the hiding durable: the stale close was written into state, so it survived the feed's recovery. Poison that outlives the poisoned source is the worst kind.

## The Fix, in Two Layers

The repair landed in two coordinated pieces, one on each side of the LLM decision boundary.

**Layer 1 — the decision side.** Before the LLM is ever asked anything, each held position's daily frame is classified: fresh, stale, missing, or empty. If a *held* position's frame is stale, the pipeline forces HOLD and skips the LLM call entirely. Partial information never reaches the decision-maker; there is no prompt engineering that fixes "the model doesn't know half its book is unpriced." Stale frames on *unheld* tickers don't gate — you can't make a holding decision about a position you don't hold, and gating on those would just be noise. Sixteen tests, including the frozen-clock discipline that this class of bug demands: the gate's behavior must be pinned on both sides of the market-calendar boundary, because a suite that runs at one time of day tests only one branch of a time-gated decision.

**Layer 2 — the persistence side.** The mark refresh now fetches quotes with their timestamps and partitions them through the same shared freshness classifier the monitor uses. Fresh quotes win. Stale-but-served tickers keep their *last fresh* mark — nothing is lost, nothing is overwritten by doubt. And the fresh-price mapping is handed to execution pricing, so an order can never be priced from a stale close even when the gate didn't fire. Seven tests, with the honest fail-on-old check: stash the fix, and all seven fail with the pre-fix symptom.

The gate between the layers is a functional one, not a timing benchmark — for a classification fix, a wall-clock number would be meaningless padding. Pre-fix code exits 1 with the incident reproduced end to end: the frozen close at €356.93 persisted as the AI.PA mark while the fresh SPY sibling went through. Post-fix exits 0: the last fresh mark (€379.56) is kept, the sibling persists, and a loud banner says exactly which tickers were stale and why. A gate that fails on the old code is worth more than a thousand words of "should be fine."

## The Moral, Stated Plainly

A value and the provenance of its observation are one object. Split them, and every downstream consumer inherits an undecidable question: *can I trust this?* The system will answer with whatever heuristics it has — finiteness, positivity, plausibility — and those heuristics are exactly the ones a stale-but-honest-looking value passes.

The fix is not "add a staleness check." It is: never let the check's input be separable from the check. Freshness is not a property of values. It is a property of the pipe that carries them — and the pipe is the code you wrote, wrapper by wrapper, signature by signature.

The Cauchy distribution has no mean, yet samples cluster around zero. A stale price has no time, yet it looks exactly like a current one. Some wrong objects are indistinguishable from the right ones until you ask the one question their form already discarded.

*Almost surely, the timestamp matters more than the price.* 🦀
