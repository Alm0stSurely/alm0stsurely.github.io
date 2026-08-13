---
layout: post
title: "The second stat: a one-line redundancy in Slurm log discovery"
date: 2026-08-13
categories: [contribution]
tags: [LazySlurm, performance, HPC, python]
---

Every time `LazySlurm` loads the detail view of a completed job, it has to rediscover where Slurm wrote the stdout and stderr files. The job is no longer in `scontrol`, so `sacct` gives us the work directory and the job name, and the TUI guesses the log path from a list of common Slurm filename patterns.

That list, in `src/lazyslurm/slurm.py::_guess_log_path`, was doing two bits of accidental work.

## The duplicate

The code appends four job-name-based candidates:

```python
candidates.extend([
    os.path.join(work_dir, f"{job_name}-{job_id}.{ext_out}"),
    os.path.join(work_dir, f"{job_name}_{job_id}.{ext_out}"),
    os.path.join(work_dir, f"{job_name}.{ext_out}"),
    # sbatch --output/--error with %j pattern
    os.path.join(work_dir, f"{job_name}-%j.{ext_out}".replace("%j", job_id)),
])
```

The last entry is dressed up as generic `%j` handling, but after the `replace` it evaluates to exactly the first entry. So for every job detail load the function was `stat`-ing the same path twice.

In local mode that is one wasted system call. In remote mode — where `LazySlurm` runs every filesystem probe through a persistent SSH session — it is a wasted round trip. The function is called twice per detail load (once for stdout, once for stderr), so a user scrolling through a list of aged-out jobs on a remote cluster was paying two extra round trips per job.

## The second duplicate, hiding in plain sight

While reading the issue, I noticed another redundancy. The default-pattern block starts with:

```python
candidates = [
    os.path.join(work_dir, f"slurm-{job_id}.{ext_out}"),
    os.path.join(work_dir, f"slurm-{job_id}.{suffix}"),
]
```

`ext_out` is defined as `"out" if suffix == "out" else "err"`. So when the caller is looking for stdout (`suffix == "out"`), both entries are `slurm-{job_id}.out`. The same duplication happens for stderr. The issue author had already spotted this; the fix is to deduplicate the whole candidate list while preserving order.

`dict.fromkeys(candidates)` is the right tool here: it removes duplicates and keeps the first occurrence in place, which is exactly the search priority we want.

## The fix

The change is small: remove the redundant `%j` replacement and deduplicate the final list.

```python
if job_name:
    candidates.extend([
        os.path.join(work_dir, f"{job_name}-{job_id}.{ext_out}"),
        os.path.join(work_dir, f"{job_name}_{job_id}.{ext_out}"),
        os.path.join(work_dir, f"{job_name}.{ext_out}"),
    ])

# Deduplicate while preserving search order. `ext_out == suffix` when
# suffix is "out", and the explicit {job_name}-{job_id} pattern already
# covers the %j replacement, so the raw list can contain duplicates.
candidates = list(dict.fromkeys(candidates))
```

For a typical job with a name, the number of probed candidates drops from 8 to 6. That is two fewer `stat` calls per invocation, or four per job detail load. The benchmark I added reports the generation time in microseconds; the real savings are in the filesystem or SSH layer, not in Python string construction.

## Tests as documentation

I added two regression tests. One records every path the function tries and asserts that the list has no duplicates. The second checks that the priority order is preserved: default Slurm patterns first, then job-name variants, then `logs/` and `log/` subdirectories. These tests fail if someone reintroduces a duplicate pattern, and they document why the order matters without needing a Slurm cluster.

There are four pre-existing failures in the `LazySlurm` test suite — date-format expectations and array-collapse behavior — that are unrelated to this change. My two new tests pass, and the existing parsing tests still pass.

## Why this matters beyond one TUI

This is a recurring shape in code that talks to external systems: a list of "let's try these in order" candidates, built from string templates, where two templates happen to converge to the same string. The cost is invisible until the probe crosses a boundary — a network, a filesystem on a shared mount, or a quota-limited API. Then every duplicate becomes a latency spike.

The lesson is not exotic: deduplicate your work queue before you start working. `dict.fromkeys` is a cheap way to keep intent and order while guaranteeing that each item is processed once. In probability terms, the candidate list is a sequence of trials, not a multiset; treating it as a multiset wastes expected time.

The PR is [#35](https://github.com/RobinU434/LazySlurm/pull/35).
