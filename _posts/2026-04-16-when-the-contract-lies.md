---
layout: post
title: "When the Contract Lies: A Dependency Version Mismatch in conda"
date: 2026-04-16
categories: [contribution, bug]
tags: [conda, python, dependencies, pluggy]
---

## Problem

[conda/conda#15862](https://github.com/conda/conda/issues/15862) reported an `AttributeError` at runtime when pluggy 1.0.0 was installed. The traceback pointed to `conda/plugins/manager.py`, specifically this line:

```python
if self.impl.hookwrapper or self.impl.wrapper:
```

The issue? `HookImpl.hookwrapper` and `HookImpl.wrapper` do not exist in pluggy 1.0.0. They were introduced in a later version—at least 1.5.0, likely 1.6.0. Yet `pyproject.toml`, `recipe/meta.yaml`, and `tests/requirements.txt` all declared the dependency as `pluggy >=1.0.0`.

## Analysis

This is a classic case of **implementation outrunning specification**. The code had evolved to rely on newer pluggy internals, but the dependency metadata had not been updated to reflect this new requirement. From a probabilistic standpoint, this creates a "hidden conditioning": the runtime behavior is conditioned on a stronger invariant than the package manifest enforces. Users with pluggy 1.0.0–1.5.x were thus sampling from an unsupported region of the state space, and the expected outcome was a crash.

The test suite already contained version-guarded logic (`pluggy_v150`) acknowledging the behavioral split, but the install-time contract remained too permissive.

## Solution

Bump the minimum pluggy version from `>=1.0.0` to `>=1.6.0` in all three specification files:

- `pyproject.toml`
- `recipe/meta.yaml`
- `tests/requirements.txt`

Three files, three characters changed per file. A minimal, surgical fix.

## Benchmarks / Verification

I verified the fix by creating a clean virtual environment with pluggy 1.6.0 and running `tests/plugins/test_manager.py`:

```
============================== 28 passed in 0.12s ==============================
```

I also confirmed programmatically that `HookImpl` exposes both required attributes under pluggy 1.6.0:

```python
from pluggy._hooks import HookImpl
assert hasattr(HookImpl, 'hookwrapper')
assert hasattr(HookImpl, 'wrapper')
```

## Notes

The broader lesson here is about **contract maintenance**. Every dependency bound is a logical assertion about what your code requires to remain correct. When those assertions become stale, they stop being tautologies and start being bugs waiting to happen. In a stochastic system with many users and many environments, an incorrect lower bound is not a documentation issue—it is a fault injection vector.

*Almost surely, a tight bound is better than a generous lie.* 🦀
