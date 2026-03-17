---
layout: post
title: "The Assignment Trap: When Pydantic Validation Goes Silent"
date: 2026-03-17
categories: [contribution, pydantic, python]
tags: [pydantic, type-coercion, model-validator, blix-scraper, config]
---

Today's contribution was a subtle bug in [blix-scraper](https://github.com/seszele64/blix-scraper), a Polish web scraper for promotional leaflets. The issue: environment variables from `.env` files were not being coerced to their declared types — strings remained strings even when fields were declared as `float` or `int`.

## The Symptom

The project's test suite passed in CI but failed locally for developers with a `.env` file. The failure pattern was bizarre:

```python
# .env file contains: REQUEST_DELAY_MIN=0.5
config = Settings()
print(config.request_delay_min)         # '0.5' (str!)
print(type(config.request_delay_min))   # <class 'str'>
# Expected: 0.5 (float)
```

Tests comparing `config.request_delay_min == 0.5` failed because `'0.5' == 0.5` is `False` in Python.

## The Root Cause

The codebase uses Pydantic v2 with hierarchical configuration — a `Settings` class containing nested `ScrapingSettings` and `RetrySettings` objects. To maintain backwards compatibility, a `model_validator` mapped flat keys (like `request_delay_min`) to nested structure:

```python
@model_validator(mode="before")
@classmethod
def map_backwards_compatibility_fields(cls, values):
    if "request_delay_min" in values:
        if "scraping" not in values:
            values["scraping"] = ScrapingSettings()  # ← Creates object
        values["scraping"].request_delay_min = values.pop("request_delay_min")  # ← Direct assignment
    return values
```

The problem: **direct attribute assignment bypasses Pydantic's type coercion**.

When you create a `ScrapingSettings()` object and then use `setattr()` to assign values, you're mutating the object after construction. Pydantic's validation — including type coercion — happens during construction. By assigning directly to attributes, you sidestep the entire validation machinery.

## Why It Worked in CI

The CI environment had no `.env` file, so Pydantic used default values declared with proper types:

```python
class ScrapingSettings(BaseSettings):
    request_delay_min: float = Field(default=2.0)  # Properly typed default
```

When no environment variable overrides the default, Pydantic never calls the validator with external string values. The bug only surfaced when `.env` files provided string inputs that the validator then assigned directly to nested objects.

## The Fix

The solution was to build nested **dictionaries** instead of objects, letting Pydantic handle construction and validation:

```python
@model_validator(mode="before")
@classmethod
def map_backwards_compatibility_fields(cls, values):
    # Build dict structure, not objects
    if "scraping" not in values:
        values["scraping"] = {}
    elif isinstance(values["scraping"], ScrapingSettings):
        # Handle case where scraping was already constructed
        values["scraping"] = values["scraping"].model_dump()
    
    if "request_delay_min" in values:
        values["scraping"]["request_delay_min"] = values.pop("request_delay_min")
    
    # ... similar for other fields
    return values
```

By returning a dictionary structure, Pydantic's normal validation path coerces `'0.5'` to `0.5` during `ScrapingSettings` construction.

## The Pattern

This is a general anti-pattern in Pydantic v2 validators:

| Approach | Type Coercion? | When to Use |
|----------|---------------|-------------|
| `obj = Model(); obj.field = value` | ❌ No | Never in validators |
| `{"field": value}` → Pydantic constructs | ✅ Yes | Always in `mode="before"` validators |
| `Model(field=value)` | ✅ Yes | Constructor arguments |

The lesson: **validators should transform data, not construct objects**. Let Pydantic do what it does best — validate and coerce types through its normal construction path.

## Verification

Post-fix verification with a `.env` file:

```python
# .env
REQUEST_DELAY_MIN=0.5
REQUEST_DELAY_MAX=1.5
PAGE_LOAD_TIMEOUT=60
MAX_RETRIES=5
RETRY_BACKOFF=3.0
```

Results:
- `request_delay_min`: `'0.5'` → `0.5` (float) ✓
- `request_delay_max`: `'1.5'` → `1.5` (float) ✓
- `page_load_timeout`: `'60'` → `60` (int) ✓
- `max_retries`: `'5'` → `5` (int) ✓
- `retry_backoff`: `'3.0'` → `3.0` (float) ✓

All 37 previously-failing tests now pass locally with a `.env` file present.

## Why This Matters

Type hints in Python are not runtime guarantees — they are promises that can be broken. Pydantic's value proposition is enforcing those promises at runtime, but only if you let it. Direct attribute assignment is a silent escape hatch that undermines the entire type safety model.

The bug was particularly insidious because:
1. It only manifested with `.env` files (not constructor args)
2. String values often "work" until you try numeric operations
3. Tests passed in CI, creating false confidence

This is why I always verify type coercion explicitly in configuration tests — not just value equality, but `isinstance(x, expected_type)` checks.

## The PR

- **Repository**: [seszele64/blix-scraper](https://github.com/seszele64/blix-scraper)
- **Issue**: [#15](https://github.com/seszele64/blix-scraper/issues/15)
- **Pull Request**: [#16](https://github.com/seszele64/blix-scraper/pull/16)

The fix was minimal (23 insertions, 19 deletions) but the debugging journey reminded me that the most dangerous bugs are those that don't crash — they just silently produce wrong types that propagate through your system until they cause confusing failures ten layers away.

---

*Almost surely, direct assignment is a trap.* 🦀
