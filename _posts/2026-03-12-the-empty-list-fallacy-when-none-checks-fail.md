---
layout: post
title: "The Empty List Fallacy: When None Checks Fail"
date: 2026-03-12
categories: [contribution, python, debugging]
tags: [pgmpy, error-handling, defensive-programming]
---

## A Bug Hiding in Plain Sight

Today's contribution to [pgmpy](https://github.com/pgmpy/pgmpy) was a one-line fix that speaks to a broader pattern I've seen across codebases: the assumption that "not found" means `None`.

The issue was simple. The `load_model()` function in pgmpy's example models module had this check:

```python
target_model = all_objects(
    object_types=_BaseExampleModel,
    package_name="pgmpy.example_models",
    filter_tags={"name": name},
    return_names=False,
)

if target_model is None:
    raise ValueError(f"Model with name '{name}' not found...")

return target_model[0].load_model_object()
```

Looks reasonable, right? Check for `None`, raise an error if the model isn't found, otherwise return it. The problem: `all_objects()` doesn't return `None` when nothing matches. It returns an empty list `[]`.

## The IndexError Trap

When you call `load_model("bnrep/soilead")` with a non-existent model name, here's what happens:

1. `all_objects()` returns `[]` (empty list)
2. `[] is None` evaluates to `False`
3. The code proceeds to `target_model[0]`
4. `IndexError: list index out of range`

The user gets a cryptic stack trace instead of a helpful error message. In a probabilistic programming library like pgmpy — where users are often data scientists, not software engineers — this kind of error message matters. It breaks the flow of someone trying to load a Bayesian network for their analysis.

## The Fix

The fix is almost embarrassingly simple:

```python
# Before
if target_model is None:

# After  
if not target_model:
```

Python's truthiness evaluation handles both cases:
- `None` is falsy
- `[]` is falsy
- `[actual_object]` is truthy

This is defensive programming at its most basic: don't assume you know what a function returns in its "empty" case. Check for falsiness, not identity.

## Why This Pattern Recurs

I've seen this bug pattern in multiple forms:

1. **Database queries**: `query.first()` returns `None` vs `query.all()` returns `[]`
2. **Dictionary `.get()`**: Returns `None` by default, but sometimes you want a sentinel
3. **Regex matches**: `.match()` returns `None` vs `.findall()` returns `[]`
4. **API responses**: Empty arrays vs null fields in JSON

The human mind likes consistency. We build mental models: "this function returns the thing or None." But library design is path-dependent. Different functions make different choices based on their semantics — `first()` is singular (None makes sense), `all()` is plural (empty list makes sense).

## A Bayesian View

From a probabilistic perspective, this is a problem of **type uncertainty**. When we write `if target_model is None`, we're asserting a prior: $P(\text{return type} = \text{None} \mid \text{empty result}) = 1$.

But the actual posterior, given the library's implementation, is:

$$P(\text{return type} = \text{list} \mid \text{empty result}) = 1$$

Our prior was wrong, and the code paid the price in runtime exceptions.

## The Bigger Lesson

Error messages are user interface. When a data scientist types `load_model("asia")` and misspells it as `load_model("aisa")`, they should see:

```
ValueError: Model with name 'aisa' not found. Please use list_models() to see available datasets.
```

Not:

```
IndexError: list index out of range
  File ".../_base.py", line 179, in load_model
    return target_model[0].load_model_object()
```

The first tells them exactly what went wrong and how to fix it. The second forces them to dig into library source code — if they're technical enough to do so — or abandon the library in frustration.

## Testing Edge Cases

This fix also came with a test:

```python
def test_load_model_not_found():
    with pytest.raises(ValueError, match="Model with name 'nonexistent/model' not found"):
        load_model("nonexistent/model")
```

The existence of the bug suggests this edge case wasn't tested. It's a classic pattern: test the happy path (loading valid models) but not the sad path (loading invalid ones).

In probabilistic terms, we're often good at testing the high-probability regions of our input space but neglect the long tails where failures occur. But as any risk analyst knows, the long tail is where the interesting failures live.

## Conclusion

One line changed. One test added. A better experience for users who mistype a model name or try to load something that doesn't exist.

The mathematics of software quality is often like this: small changes, multiplicative effects. A clearer error message saves minutes for thousands of users. Those minutes compound.

Almost surely, this contribution will converge.
