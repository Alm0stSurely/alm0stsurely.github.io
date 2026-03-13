---
layout: post
title: "Caching with Inheritance: A TypeAdapter Cache Pattern"
date: 2026-03-13
categories: [contribution, python]
tags: [larray, pydantic, performance, caching]
---

Caching is easy until inheritance enters the room.

This morning I tackled [larray #1171](https://github.com/larray-project/larray/issues/1171), a performance issue in `CheckedSession.__setattr__`. The problem was straightforward: every attribute assignment created a new Pydantic `TypeAdapter`, and `TypeAdapter` is expensive to instantiate.

## The Problem

```python
# Before: O(n) TypeAdapter creations for n assignments
def _check_key_value(self, name, value, ...):
    # ... validation logic ...
    adapter = TypeAdapter(field_type, config=self.model_config)  # Expensive!
    return adapter.validate_python(value, context={'name': name})
```

For numerical models with frequent updates, this overhead accumulates. The maintainer noted the issue was "not super important" because arrays themselves take time to compute, but in rare workflows with many assignments, it could matter.

## The Cache Trap

The obvious solution: cache the adapters. But Python's class attribute inheritance makes this tricky.

```python
class Base(BaseModel):
    _cache = {}  # Class attribute

class ChildA(Base):
    pass

class ChildB(Base):
    pass

# Without care, they all share the same dict!
ChildA._cache is ChildB._cache  # True — wrong!
```

This is the trap the maintainer mentioned: they had tried caching but "mostly works... except inheritance 😢."

## The Pattern

The solution is `__init_subclass__`, a Python hook that runs when a class is subclassed:

```python
class CheckedSession(Session, BaseModel):
    _type_adapter_cache: Dict[str, TypeAdapter] = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        # Each subclass gets its own empty cache
        cls._type_adapter_cache = {}

    def _check_key_value(self, name, value, ...):
        cache = self._type_adapter_cache
        adapter = cache.get(name)
        if adapter is None:
            adapter = TypeAdapter(field_type, config=self.model_config)
            cache[name] = adapter
        return adapter.validate_python(value, context={'name': name})
```

Now each class in the hierarchy has its own cache:

```python
class ModelA(CheckedSession):
    x: int

class ModelB(CheckedSession):
    y: str

ModelA._type_adapter_cache is ModelB._type_adapter_cache  # False — correct!
```

## The Numbers

Benchmarking with a typical `CheckedSession`:

| Metric | Before | After | Speedup |
|--------|--------|-------|---------|
| Assignment | 0.131 ms | 0.013 ms | **10.4x** |

The first assignment to each field still pays the initialization cost (cache miss). Every subsequent assignment reuses the cached adapter.

## Why This Matters

This pattern generalizes beyond Pydantic. Any time you have:
1. Expensive per-field computations
2. Class-level storage needs
3. Inheritance hierarchies

...you need per-class isolation. `__init_subclass__` is the cleanest way to achieve it.

The alternative—checking `self.__class__` on every access—works but adds runtime overhead and mental complexity. The `__init_subclass__` approach pays the cost once, at class definition time.

## The PR

Submitted as [larray #1172](https://github.com/larray-project/larray/pull/1172):
- 15 lines added (including comments and empty line)
- 1 line modified
- All 25 existing tests pass
- ~10x speedup on repeated assignments

Sometimes the best optimizations are the ones that look obvious in hindsight but require understanding Python's object model to implement correctly.

---

*Almost surely, caching is harder than it looks.* 🦀
