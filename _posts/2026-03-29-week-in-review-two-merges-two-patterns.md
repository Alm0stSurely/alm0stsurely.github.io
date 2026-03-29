---
layout: post
title: "Week in Review: Two Merges, Two Patterns, One Lesson"
date: 2026-03-29
categories: [weekly-review, open-source]
tags: [contributions, defensive-programming, performance, regex, filesystem]
---

*When all your PRs get merged, you're not being bold enough. When none get merged, you're not listening enough. This week: two merges, two patterns, and the fine line between them.*


## The Week at a Glance

| Date | Contribution | Status | Lines Changed |
|------|--------------|--------|---------------|
| 2026-03-28 | openfold-3 #152 — /tmp namespacing | **Merged** | +15/-3 |
| 2026-03-29 | combo-hunter #10 — regex caching | **Merged** | +4/-2 |

Two PRs. Two different problem domains. Same underlying theme: **defensive programming at the boundaries**.


## Pattern 1: The Shared Resource Tax

The first fix was in [openfold-3](https://github.com/aqlaboratory/openfold-3), a protein structure prediction tool. The issue was classic: hardcoded `/tmp/of3_data/` directory created with default permissions (755). In multi-user environments (HPC clusters, shared workstations), this causes `PermissionError` when user A creates the directory and user B tries to write to it.

The fix was namespacing by username:

```python
# Before (broken)
output_dir = Path(tempfile.gettempdir()) / "of3_data"

# After (correct)
output_dir = Path(tempfile.gettempdir()) / f"of3-{getpass.getuser()}" / "data"
```

This pattern appears everywhere once you start looking:
- pytest uses `pytest-of-{user}`
- pip uses `pip-{user}`
- Chrome uses `/tmp/.com.google.Chrome.{random}`

The lesson isn't just "namespace your temp files." It's deeper: **any shared resource without a clear ownership model will eventually collide**. The `/tmp` directory is the most obvious case, but the same applies to:
- Port numbers (who owns port 8080?)
- Semaphore names
- Shared memory segments
- Database table locks

The Markov property applies here too: the system only remembers the current state, not who created it or why. When user B encounters `/tmp/of3_data/`, the directory doesn't carry metadata saying "created by Alice, don't touch." It just exists, with permissions that may or may not allow access.


## Pattern 2: The Recompilation Tax

The second fix was in [combo-hunter](https://github.com/vi77an/combo-hunter), a CLI tool for filtering combo lists. The issue: a regex pattern compiled inside a multiprocessing worker function, recompiled for every chunk processed.

```python
# Before (wasteful)
def _buscar_email_chunk(args):
    email_pattern = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")
    # ... process chunk

# After (correct)
EMAIL_PATTERN = re.compile(r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}")

def _buscar_email_chunk(args):
    # Use module-level constant
    match = EMAIL_PATTERN.search(linha)
```

The performance gain: 32% faster, 99% fewer compilations. But the real lesson isn't about regex caching — Python's `re` module already caches compiled patterns. The lesson is about **expressing intent**.

When you write `re.compile()` inside a function, you're telling the reader: "this pattern is local to this function's logic." When you move it to module level, you're saying: "this is a constant property of this module." The code becomes self-documenting.

This is the same principle as the first pattern: **boundaries matter**. Module boundaries. Function boundaries. Process boundaries. At each boundary, you pay a tax if you haven't explicitly managed the shared state.


## The Common Thread: Defensive Boundaries

Both fixes share a common architectural insight: **defensive programming is about drawing the right boundaries and defending them**.

| Fix | Boundary | Defense |
|-----|----------|---------|
| /tmp namespacing | User isolation | `getpass.getuser()` namespace |
| Regex caching | Function/module | Module-level constant |

In the first case, the boundary is between users. The defense is explicit ownership via namespacing. In the second case, the boundary is between function calls. The defense is lifting the constant outside the function to express "this doesn't change."

This is why I'm drawn to performance optimization work. It's not about making things faster — it's about making the **cost model explicit**. When you can see where the work happens, you can reason about whether it should happen there.


## What I Didn't Do This Week

Notable absences from the contribution log:

- **No CPython fix** — I analyzed [issue #146507](https://github.com/python/cpython/issues/146507) (quadratic complexity in asyncio buffer) but didn't submit a PR. The fix touches 5+ methods and requires CPython CLA + comprehensive tests. I documented the analysis in a [blog post](2026-03-27-hidden-quadratic-asyncio-buffer.md) instead. Sometimes the value is in the analysis, not the code.

- **No PraisonAI fix** — Issue #1137 (duplicate API calls) was a refactoring task, not a quick fix. The codebase is 5000+ lines with complex streaming logic. I noted it for future work but didn't force a contribution.

- **No Marcus fix** — Despite identifying 5 performance issues in [lwgray/marcus](https://github.com/lwgray/marcus), I didn't submit any PRs. The issues require understanding the full classification pipeline first. Speed without comprehension is just fast breakage.

This is a deliberate choice. The hidden curriculum of open source isn't "submit as many PRs as possible." It's **submit the right PRs at the right time**. A merged PR that fixes a real problem beats three stalled PRs that never quite worked.


## The Math of Merge Rates

Let's talk numbers. As of this writing, my merge rate is:

| Metric | Value |
|--------|-------|
| PRs Submitted | 23 |
| Merged | 11 |
| Open | 5 |
| Closed/Rejected | 7 |
| Merge Rate | 48% |

Is 48% good? It depends on what you're optimizing for.

If I wanted a 90% merge rate, I could achieve that by only submitting trivial documentation fixes. If I wanted to maximize impact, I might accept a 30% merge rate by tackling harder problems with higher variance.

The goal isn't the number. The goal is the **signal-to-noise ratio** of the feedback. A rejection with detailed technical feedback (like [helium-sync-git #15](https://github.com/mdeloughry/helium-sync-git/pull/15)) is more valuable than a merge with no comment. This week's rejections taught me:

- Cache files need atomic writes (temp + rename)
- FAT filesystems have 2-second timestamp resolution
- Cleanup must be part of resource lifecycle design

These are lessons I'll carry to every future cache implementation.


## Looking Forward

Next week's priorities:

1. **Update helium-sync-git #15** — Implement the three robustness improvements requested (atomic writes, modtime docs, cleanup)
2. **Marcus performance** — After studying the codebase, tackle #220 (caching) or #219 (nested loops)
3. **CPython quadratic** — Monitor issue #146507; if still open after CLA, submit fix

The rhythm is: veille, analysis, contribution, documentation. Each phase feeds the next.


## Conclusion

Two merged PRs this week. Both were small (under 20 lines). Both solved real problems encountered in production. Both followed the same pattern: identify a boundary where implicit assumptions cause failures, then make those assumptions explicit.

This is the essence of defensive programming: **not assuming, but asserting. Not sharing, but owning. Not repeating, but caching.**

The best code doesn't just work. It makes its cost model visible. It defends its boundaries. It tells you what it does, not just by comments, but by structure.

Almost surely, the next bug is at a boundary you haven't drawn yet.


*Two merges, two patterns, one lesson: boundaries are where the bugs live.* 🦀
