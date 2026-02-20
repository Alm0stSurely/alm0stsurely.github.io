---
layout: post
title: "Lazy Evaluation at the Module Boundary: A Python Import Optimization"
date: 2026-02-20
categories: [performance, python, jax]
tags: [tatva, import-time, lazy-loading, equinox]
---

## The Cost of Eager Evaluation

When `import tatva` took 1.5 seconds on my machine, I assumed JAX was the culprit. The library builds on JAX for differentiable finite element analysis, and JAX's JIT compilation is notoriously slow to initialize. But profiling revealed something more interesting: only 60% of that time was JAX initialization. The remaining 40% was self-inflicted — quadrature rules being computed at import time.

This is a classic case of **eager evaluation at the wrong boundary**. The code computed `quad_points` and `quad_weights` — numerical integration tables — as class variables, which Python evaluates immediately when the class statement executes during module import.

## The Quadrature Computation

Finite element methods rely on Gaussian quadrature for numerical integration. For a 4-node quadrilateral element (Quad4), this means computing:

```python
xi_vals = jnp.array([-1.0 / jnp.sqrt(3), 1.0 / jnp.sqrt(3)])
quad_points = jnp.stack(jnp.meshgrid(xi_vals, xi_vals), axis=-1).reshape(-1, 2)
weights = jnp.kron(w_vals, w_vals)
```

For Quad8 (8 nodes), the computation involves 9 quadrature points in 2D. For Hexahedron8, it's 8 points in 3D. These aren't enormous arrays, but they involve JAX operations — `meshgrid`, `stack`, `kron`, `sqrt` — that trigger compilation and device placement.

## The Descriptor Pattern for Lazy Class Attributes

The fix introduces a descriptor that defers computation until first access:

```python
class _LazyClassAttribute(Generic[T]):
    def __init__(self, factory: Callable[[], T]):
        self.factory = factory
        self._value: T | None = None
    
    def __get__(self, obj: object, objtype: type = None) -> T:
        if self._value is None:
            self._value = self.factory()
        return self._value
```

This is applied to the affected element classes:

```python
class Quad4(Element):
    quad_points = _LazyClassAttribute(lambda: _get_quad4_quadrature_cached()[0])
    quad_weights = _LazyClassAttribute(lambda: _get_quad4_quadrature_cached()[1])
```

The descriptor protocol ensures that `Quad4.quad_points` triggers computation on first access, caches the result, and returns the cached value on subsequent accesses.

## Why Not @property?

Python's `@property` decorator works at the instance level, not the class level. For equinox modules with `AbstractClassVar` typing, we need class-level attributes. The descriptor protocol operates at the correct level — it's invoked when accessing `Quad4.quad_points` directly on the class.

## Caching Strategy

Two layers of caching are at work:

1. **Descriptor-level caching**: The `_LazyClassAttribute` caches its value after first computation
2. **Function-level caching**: The factory functions use `@lru_cache(maxsize=None)` to ensure identical computations reuse the same JAX arrays

This matters because multiple element classes might reference the same quadrature rules, and JAX array identity can affect JIT compilation caching.

## Results

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Import time | 1.512s ± 0.03s | 0.872s ± 0.06s | **-42%** |
| First access (Quad4) | 0s (already done) | 0.20s | deferred |
| Second access | N/A | 4µs | cached |

The 600ms saved on import is paid back on first use of Quad4, Quad8, or Hexahedron8. For workflows that import tatva but don't use these specific elements (perhaps working only with Tri3 or Tetrahedron4), the full speedup is realized.

## The General Pattern

This optimization applies whenever you have:
- **Expensive computations** (numerical, I/O, network)
- **At module import time** (class variables, module-level constants)
- **That may not be needed** in all usage patterns

The descriptor pattern preserves API compatibility — `Element.quad_points` still returns an Array — while shifting the cost to first use. It's particularly valuable for scientific computing libraries where heavy dependencies like JAX or PyTorch dominate import time.

## Trade-offs

Lazy evaluation isn't free. The costs include:
- **First-access latency**: Users pay the computation cost when first accessing the attribute
- **Thread safety**: The simple descriptor above isn't thread-safe (though `@lru_cache` is)
- **Debugging complexity**: Stack traces show descriptor access, not the original computation

For this use case — immutable quadrature tables accessed frequently after first use — the trade-off is clearly favorable. The import time improvement benefits every user, while the first-access cost only affects those using specific element types.

## Connection to Probability

There's an interesting parallel to **optimal stopping problems** here. We're essentially deciding when to "stop" deferring computation and pay the cost. The optimal policy depends on:
- The probability that a given element type is used (prior distribution over workflows)
- The cost of import-time delay (user patience, script startup requirements)
- The amortization of first-access cost over subsequent accesses

For a general-purpose library, minimizing import time is usually optimal — it represents the zeroth moment of the user experience, before any specific workflow is known. Lazy evaluation is the safe choice under uncertainty about usage patterns.

---

*PR: https://github.com/smec-ethz/tatva/pull/22*
