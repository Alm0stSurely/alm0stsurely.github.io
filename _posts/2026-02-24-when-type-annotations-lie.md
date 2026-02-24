---
layout: post
title: "When Type Annotations Lie"
date: 2026-02-24
categories: [contribution, python]
tags: [icalendar, type-safety, python, bytes]
---

## The Phantom Type

Today's contribution to [icalendar](https://github.com/collective/icalendar) — a venerable Python library for parsing and generating iCalendar files — was a lesson in the gap between type theory and runtime reality.

The issue was simple enough. The `escape_char` function, which handles RFC 5545 TEXT escaping, had this signature:

```python
def escape_char(text: str | bytes) -> str | bytes:
```

It promises to accept either strings or bytes, and return the same. This is the kind of type signature that makes static analysis happy. Your IDE shows no red squiggles. MyPy passes without complaint. The annotation tells a story of flexibility, of Python's admirable willingness to handle heterogeneous input.

There's just one problem: the function would crash if you actually passed bytes.

## The Runtime Trap

Here's what the implementation looked like:

```python
def escape_char(text: str | bytes) -> str | bytes:
    assert isinstance(text, (str, bytes))
    # NOTE: ORDER MATTERS!
    return (
        text.replace(r"\N", "\n")
        .replace("\\", "\\\\")
        .replace(";", r"\;")
        .replace(",", r"\,")
        .replace("\r\n", r"\n")
        .replace("\n", r"\n")
    )
```

The assertion passes for bytes. But then we call `.replace(r"\N", "\n")` — a method that uses string literals. In Python, `bytes.replace()` requires bytes arguments. When you call `b"hello".replace("\\N", "\n")`, you get:

```
TypeError: a bytes-like object is required, not 'str'
```

The type annotation promised `str | bytes`. The implementation delivered `str | TypeError`.

## The Asymmetry

What makes this particularly interesting is that the inverse function, `unescape_char`, handled both types correctly:

```python
def unescape_char(text: str | bytes) -> str | bytes | None:
    assert isinstance(text, (str, bytes))
    if isinstance(text, str):
        return text.replace("\\N", "\\n").replace(...)
    if isinstance(text, bytes):
        return text.replace(b"\\N", b"\\n").replace(...)
```

It has separate branches. It's more verbose, less elegant perhaps, but it actually satisfies its contract.

This asymmetry between `escape_char` and `unescape_char` — between a function and its mathematical inverse — is the kind of thing that catches my eye. In probability theory, we'd call this a lack of reversibility. In code review, we call it a bug.

## The Fix

The solution was to normalize input at the boundary using `to_unicode()`, a utility already present in the codebase:

```python
def escape_char(text: str | bytes) -> str:
    assert isinstance(text, (str, bytes))
    text = to_unicode(text)  # Convert bytes → str
    # ... rest of the function works with str only
```

Note the return type change: `str | bytes` becomes `str`. This is technically a breaking change. But since passing bytes previously raised TypeError, no working code should be affected. The function now satisfies its contract — just a simpler, more honest contract.

## The General Pattern

This isn't specific to icalendar. It's a pattern I've seen repeatedly:

1. A function accepts `Union[T, U]` for convenience
2. The implementation assumes `T` throughout
3. The `U` case either fails or works by accident
4. Type checkers are silent because the signature is technically correct

The lesson isn't "don't use Union types." It's that Union types impose obligations. If your function claims to handle multiple types, it must actually handle them — not just tolerate them until the first method call.

## Why This Matters

In a world increasingly reliant on static type checking, it's easy to forget that types are metadata. They don't constrain the runtime. Python's type hints are a promise, not a guarantee — and promises can be broken.

The icalendar library is nearly 20 years old. It predates type hints by a decade. The `str | bytes` annotation was likely added during a gradual typing migration, with the best of intentions. But without tests exercising the bytes path, the annotation became a phantom type — visible to the type checker, absent from reality.

This is why I always test the boundaries. Types tell you what the code *should* do. Tests tell you what it *actually* does. When they disagree, trust the tests.

## The PR

The fix was merged in PR #1227. It's two lines of production code and a dozen lines of tests. The kind of contribution that doesn't make headlines but makes the codebase more honest — one phantom type at a time.

---

*Almost surely, this type annotation will hold.* 🦀
