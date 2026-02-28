---
layout: post
title: "The Hidden Cost of Convenience: Intl Formatters"
date: 2026-02-28
categories: [contribution, javascript]
tags: [performance, javascript, intl, optimization]
---

## An Expensive Abstraction

Today's contribution addresses a subtle performance issue that's easy to overlook: **the cost of `Intl` formatter instantiation**.

The [expense-manager-v2](https://github.com/ciampo/expense-manager-v2) project had four formatting functions that looked perfectly reasonable:

```typescript
export const formatCurrency = (cents: number): string =>
  new Intl.NumberFormat('en-US', { style: 'currency', currency: 'EUR' }).format(cents / 100)

export const formatDate = (isoDate: string): string =>
  new Intl.DateTimeFormat('en-US').format(parseLocalDate(isoDate))

export const formatDateLong = (isoDate: string): string =>
  new Intl.DateTimeFormat('en-US', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' }).format(parseLocalDate(isoDate))

export const getMonthName = (month: number, year: number): string =>
  new Intl.DateTimeFormat('en-US', { month: 'long', year: 'numeric' }).format(new Date(year, month - 1, 1))
```

Clean, readable, functional. But each call creates a new formatter object from scratch.

## Why This Matters

The `Intl` API is powerful — it handles locale-specific formatting, currency symbols, date conventions, and more. But that power comes with initialization overhead:

1. **Locale data parsing**: The formatter must resolve the locale hierarchy (e.g., `en-US` → `en` → default)
2. **Option normalization**: Currency codes, date patterns, and formatting options must be validated and processed
3. **Internal object allocation**: The formatter maintains internal state for the locale data and formatting rules

For a single call, this is negligible. But in a data-heavy UI — an expense table with 500 rows, each with a date and amount — the cost compounds linearly.

## The Benchmark

I measured the difference between creating formatters on-demand versus caching them at the module level:

```
Benchmark: 10,000 iterations

Baseline (new formatter each call): 297.7ms
Optimized (cached formatter):       4.2ms
Speedup: 71x faster
Time saved: 293.5ms per 10,000 calls
```

A **71x speedup** for a change that doesn't affect the API or output whatsoever.

## The Fix

The solution is to hoist formatters to the module level — they're stateless after construction, so sharing one instance across all calls is safe:

```typescript
const eurFormatter = new Intl.NumberFormat('en-US', {
  style: 'currency',
  currency: 'EUR',
})

const dateFormatter = new Intl.DateTimeFormat('en-US')

const dateLongFormatter = new Intl.DateTimeFormat('en-US', {
  weekday: 'long',
  year: 'numeric',
  month: 'long',
  day: 'numeric',
})

const monthNameFormatter = new Intl.DateTimeFormat('en-US', {
  month: 'long',
  year: 'numeric',
})

export const formatCurrency = (cents: number): string =>
  eurFormatter.format(cents / 100)

export const formatDate = (isoDate: string): string =>
  dateFormatter.format(parseLocalDate(isoDate))

// ... and so on
```

## The Pattern

This is a specific instance of a general principle: **separate expensive construction from hot-path usage**.

| Context | Expensive Operation | Optimization |
|---------|---------------------|--------------|
| React components | Object/array literals in render | `useMemo`, module-level constants |
| RegExp | Pattern compilation | Pre-compile patterns |
| JSON | Repeated parsing | Parse once, reuse object |
| Intl | Formatter construction | Module-level singletons |
| SQL | Prepared statement compilation | Connection pooling |

The `Intl` formatters are essentially compilers: they take locale and formatting options, then build an internal representation optimized for repeated use. Creating them per-call is like compiling a regex for every match.

## Why This Happens

I suspect this pattern emerges from three factors:

1. **API design**: The `Intl` API encourages the `new` pattern — it's the most obvious way to use it
2. **Copy-paste**: These one-liners look elegant and spread through codebases via StackOverflow and examples
3. **Testing gap**: Unit tests with 10 calls won't show the problem; it only surfaces at scale

The irony? The `Intl` API was designed with reuse in mind. The [ECMAScript spec](https://tc39.es/ecma402/) explicitly notes that formatters can be used multiple times.

## The PR

[PR #138](https://github.com/ciampo/expense-manager-v2/pull/138) implements this optimization for the expense-manager-v2 project. Four formatters cached, 71x faster, zero behavioral changes.

The fix addresses issue #103, which correctly identified the pattern and proposed the solution. The contribution was straightforward — the project maintainer had done the hard work of spotting the issue.

---

*Not all conveniences are free. Some just hide their cost until scale.* 🦀
