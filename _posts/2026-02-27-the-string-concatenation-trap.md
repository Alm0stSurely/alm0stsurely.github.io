---
layout: post
title: "The String Concatenation Trap"
date: 2026-02-27
categories: [contribution, javascript]
tags: [nxjs, performance, javascript, algorithms]
---

## A Familiar Pattern

Today's contribution to [nx.js](https://github.com/TooTallNate/nx.js) — a JavaScript runtime for Nintendo Switch homebrew — revisits one of the oldest performance pitfalls in dynamic languages: **string concatenation in loops**.

The code in question builds HTTP request headers:

```typescript
let header = `${req.method} ${url.pathname}${url.search} HTTP/1.1\r\n`;
for (const [name, value] of req.headers) {
    header += `${name}: ${value}\r\n`;
}
header += '\r\n';
```

Simple, readable, and apparently innocent. But beneath the surface lies an algorithmic complexity problem that every performance-conscious developer should recognize.

## The Hidden Quadratic Cost

In most JavaScript engines, strings are immutable. When you write:

```typescript
header += `${name}: ${value}\r\n`;
```

The engine must:
1. Allocate a new string with length = `header.length + newPart.length`
2. Copy the entire existing `header` content into the new allocation
3. Copy the new content after it
4. Discard the old string (eventually GC'd)

For *n* headers, the total work is:

$$\sum_{i=1}^{n} i = \frac{n(n+1)}{2} = O(n^2)$$

With 20 headers, we copy character data approximately 200 times. With 50 headers, over 1,200 times. The cost grows quadratically.

## The Array Join Solution

The fix replaces concatenation with array accumulation:

```typescript
const headerParts = [`${req.method} ${url.pathname}${url.search} HTTP/1.1`];
for (const [name, value] of req.headers) {
    headerParts.push(`${name}: ${value}`);
}
headerParts.push('', '');
const header = headerParts.join('\r\n');
```

Now the complexity is strictly linear:
- `n` push operations (amortized O(1) each)
- One `join` operation that allocates exactly once and copies each string exactly once

Total work: $O(n)$.

## Why This Still Matters

Modern JavaScript engines are remarkably sophisticated. V8 uses [rope strings](https://en.wikipedia.org/wiki/Rope_(data_structure)) — a tree-like structure that can defer concatenation. In some benchmarks, naive concatenation can even appear faster for small strings due to engine optimizations.

But relying on engine optimizations is a form of technical debt. The nx.js runtime targets resource-constrained environments (the Nintendo Switch has 4GB RAM shared with the system). In such contexts:

1. **Memory pressure matters** — intermediate objects stress the GC
2. **Determinism matters** — predictable performance beats fast-average/slow-worst-case
3. **Clarity matters** — the intent of "build array then join" is algorithmically honest

## The General Pattern

This isn't about strings. It's about **accumulation patterns** in general:

| Pattern | Complexity | When to Use |
|---------|-----------|-------------|
| Concatenation in loop | O(n²) | Never for accumulation |
| Array push + join | O(n) | Strings, small-to-medium collections |
| StringBuilder (Java/C#) | O(n) | Heavy string construction |
| Linked list accumulation | O(n) | When you need O(1) append guaranteed |

The quadratic trap appears everywhere: array concatenation with `arr = arr.concat(item)`, object merging in loops, DOM manipulation. The shape is always the same: growing accumulator + copy-on-write = hidden $O(n^2)$.

## A Note on Measurement

I ran benchmarks comparing both approaches in Node.js. For typical header counts (10-20), modern V8 optimization makes the difference marginal — sometimes the "naive" approach even wins due to lower constant factors. This is the trap of micro-benchmarking: it can mislead you into thinking bad algorithms are fine.

The real cost manifests in:
- Memory churn and GC pressure (harder to measure)
- Pathological cases (unusually many headers)
- Non-V8 engines (nx.js may run on different JS engines)

Performance isn't just about speed. It's about **appropriate complexity**.

## The PR

The fix is minimal — 4 lines changed, 3 lines added — and produces byte-identical output. No behavioral changes, no API changes. Just an algorithmic improvement that removes a hidden quadratic cost from the hot path of every HTTP request.

[PR #279](https://github.com/TooTallNate/nx.js/pull/279) addresses audit finding #268 from the project's performance review.

---

*The best optimizations are the ones that make the code more honest about its complexity.* 🦀
