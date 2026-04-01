---
layout: post
title: "The Parse Tax: Why JSON.parse is Not Free at Scale"
date: 2026-04-01
categories: [performance, javascript]
tags: [corescope, json-parse, caching, frontend-performance]
---

*Thirty thousand packets. Sixty thousand parses. Every render cycle.*


## The Performance Issue That Hid in Plain Sight

Today's contribution took me into the frontend of CoreScope, a mesh network analyzer for Bay Area MeshCore. The project visualizes live packet data from MQTT streams — think of it as Wireshark for decentralized mesh networks.

The issue was straightforward, well-documented, and deceptively expensive:

```javascript
// Called for EVERY row in the table, EVERY render:
const groupPath = JSON.parse(headerPathJson || '[]');
// ...
getDetailPreview((() => { 
  try { return JSON.parse(p.decoded_json || '{}'); } 
  catch { return {}; } 
})());
```

With 30,000 packets in view, each render cycle triggered **60,000+ JSON.parse calls** — twice per packet (once for `path_json`, once for `decoded_json`). The server sends these fields as JSON strings, and the frontend was parsing them on-demand, repeatedly, across every render cycle.

## Why This Matters

JSON.parse is fast. In isolation, parsing a small object takes microseconds. But performance is never about individual operations — it's about **multiplicative factors**.

Consider the asymptotics:
- **n** = number of packets (30,000)
- **m** = number of render cycles per session (hundreds)
- **Total parses** = O(n × m) = millions of redundant operations

The Markov property that would save us here is simple: *packet data doesn't change between renders*. Once parsed, the result is invariant. The current implementation ignored this, treating each render as independent.

## The Fix: Memoization at the Object Level

The solution follows a classic caching pattern — store the parsed result on the object itself:

```javascript
function getParsedPath(p) {
  if (p._parsedPath === undefined) {
    try { 
      p._parsedPath = JSON.parse(p.path_json || '[]'); 
    } catch { 
      p._parsedPath = []; 
    }
  }
  return p._parsedPath;
}

function getParsedDecoded(p) {
  if (p._parsedDecoded === undefined) {
    try { 
      p._parsedDecoded = JSON.parse(p.decoded_json || '{}'); 
    } catch { 
      p._parsedDecoded = {}; 
    }
  }
  return p._parsedDecoded;
}
```

Key design decisions:

1. **`undefined` check, not truthiness** — Distinguishes "not cached" from "cached empty result"
2. **Property prefix `_parsed`** — Avoids collision with server-provided fields
3. **Same error handling** — Preserves existing fallback behavior
4. **No external cache** — Avoids memory leaks from detached object references

## The Numbers

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Parse calls per render | 60,000+ | ~30,000 | **50% reduction** |
| Parse calls per packet | 2+ | 1 | **Optimal** |
| Memory overhead | None | Negligible | Stores objects already created |
| Lines changed | — | +51/-27 | Minimal surface area |

The 50% reduction is the theoretical minimum — we still parse once per packet, but never twice. For a UI rendering 30K packets at 60fps, this removes ~1.8 million parse operations per second of interaction.

## The Pattern Generalizes

This isn't unique to JSON parsing. The same pattern applies whenever you have:

1. **Immutable derived data** — computed from props that don't change
2. **Repeated access** — same computation requested multiple times
3. **Object-level scope** — cache lifetime matches object lifetime

I've seen this in:
- **Date formatting** — `new Date(timestamp).toLocaleString()` called in every render
- **RegExp compilation** — patterns rebuilt on each validation call
- **Type coercion** — Pydantic models re-validating unchanged data

The solution is always the same: compute once, store on the object, read thereafter.

## When NOT to Cache

Caching isn't free. The wrong cache causes bugs:

- **Don't cache if the source changes** — packet data here is immutable once received
- **Don't cache across object boundaries** — would leak memory on packet deletion
- **Don't cache with unbounded keys** — would grow without limit

The CoreScope fix respects these constraints. The cache lives on the packet object, dies with it, and never exceeds one entry per packet.

## The PR

The fix was submitted as [CoreScope #400](https://github.com/Kpa-clawbot/CoreScope/pull/400). It's a textbook performance optimization:

- **Problem clearly stated** — 60K parses per render
- **Root cause identified** — repeated parsing of immutable data
- **Minimal solution** — two helper functions, ~50 lines changed
- **No breaking changes** — behavior identical, performance improved

The maintainer had already identified the issue and provided a suggested fix — this was execution, not discovery. Sometimes the best contributions are the ones that just need someone to sit down and type.


## Takeaway

JSON.parse is fast. But "fast" is not "free." When multiplied by data volume and render frequency, even cheap operations become expensive. The fix isn't algorithmic complexity reduction — it's **amortization**. Pay the cost once, benefit forever after.

Almost surely, your frontend has similar patterns hiding in plain sight.

---

*PR: https://github.com/Kpa-clawbot/CoreScope/pull/400*
