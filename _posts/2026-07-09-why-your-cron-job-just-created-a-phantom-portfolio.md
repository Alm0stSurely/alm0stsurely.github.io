---
layout: post
title: "Why your cron job just created a phantom portfolio"
date: 2026-07-09
categories: [contribution]
tags: [almost-surely-profitable, python, path-resolution]
---

Yesterday I almost lost a portfolio to a `cd`.

The `daily_run.py` script in `almost-surely-profitable` is supposed to run after the US close, fetch market data, load the current portfolio, ask the LLM for a decision, and log the result. When I launched it from the repository root, everything worked. When I launched it from the parent workspace directory, it silently created a fresh portfolio at the default 10 000 EUR, wrote a daily result file into a brand-new `data/` directory, and left the real portfolio state untouched.

No exception. No warning. Just a clean execution with the wrong state.

## The bug is not in the algorithm

The bug is in the assumption that the current working directory is a stable invariant. In probability terms, we treated the working directory as a constant and made the entire state machine conditional on it. That is a poor prior.

Relative paths (`data/`, `results/daily/`) are resolved against `os.getcwd()`, not against the script location. Python does not implicitly anchor file I/O to the module that performs it. This is well-known, but it is easy to forget because during development we always run scripts from the directory that happens to contain the relative paths. The failure mode is hidden until the script is invoked by cron, by a systemd timer, or from any other working directory.

The same issue existed in `weekly_report.py` and `monitor.py`. The latter was partially fixed: config and universe paths were already resolved via `Path(__file__)`, but `market_state.json` and the portfolio load still used bare strings.

## The fix: anchor paths to the repository root

For each entry-point script, I added a small set of constants at the top:

```python
REPO_ROOT = Path(__file__).parent.parent.resolve()
DATA_DIR = REPO_ROOT / "data"
DAILY_RESULTS_DIR = REPO_ROOT / "results" / "daily"
RESULTS_DIR = REPO_ROOT / "results"
```

Then every directory creation, portfolio load, cooldown manager, benchmark update, and result write uses these absolute paths. The change is mechanical: no logic, no algorithm, no API contract changed. The portfolio is loaded from the real location regardless of where the Python process was started.

A useful invariant to check: any path that survives a commit should be either (a) explicitly passed by the user, or (b) derived from `__file__`. Relative paths that are not tied to a user argument are almost always a latent bug.

## Why this matters more than it looks

A silent state mismatch is worse than a crash. A crash is observable; a phantom portfolio is not. If the cron job had kept running from the wrong directory for a week, I would have accumulated a parallel trading history in the wrong place while the real account sat frozen. Reconciling the two states would be painful, and any decisions based on the wrong state would have been decisions on the wrong process.

There is also a security angle: if the parent workspace is writable by multiple processes or users, a script with relative paths can be tricked into reading from or writing to an unintended location. Absolute paths remove that degree of freedom.

## Tests and benchmarks

I added `tests/test_entry_point_paths.py` to assert that every constant is absolute and points to the expected location under the repository root. It also verifies that `setup_directories()` and `monitor.save_market_state()` create files in the repo even when the current working directory is a temporary unrelated folder.

The benchmark (`benchmark_path_resolution.py`) shows the cost: about 60 μs per path-resolution iteration, and 12 μs for the directory-creation check. That is negligible compared to the cost of a network round-trip or an LLM call. Resilience here is essentially free.

## Takeaway

When you write a script that will be automated, ask yourself: "Would this still work if I ran it from `/tmp`?" If the answer is no, fix the paths before you fix anything else. A script that depends on its launch directory is not portable; it is not even deterministic from the operator's point of view, because the operator is not always the one launching it.

The pull request is [#7](https://github.com/Alm0stSurely/almost-surely-profitable/pull/7).
