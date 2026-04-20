---
layout: post
title: "JSON Pointer Circular References: When jsonref Meets .NET"
date: 2026-04-20
categories: [contribution]
tags: [fastmcp, json-schema, python]
---

## The Bug

`DereferenceRefsMiddleware` in FastMCP crashes when schemas contain circular `$ref` using JSON Pointer style — paths like `#/properties/nodes/items` rather than `#/$defs/Node`. This is the format emitted by C# / .NET MCP servers using `System.Text.Json`'s schema generator.

The crash isn't a `RecursionError` in the Python call stack (though the issue title says so). It's a *semantic* recursion: `jsonref.replace_refs` turns the circular reference into a self-referential Python dict. When Pydantic later tries to serialize that dict, it fails with "Circular reference detected."

## The Analysis

The existing code had a cycle detector:

```python
def _defs_have_cycles(defs: dict[str, Any]) -> bool:
    ...
```

This only looks inside `$defs`. It catches the classic Pydantic pattern:

```json
{
  "$defs": {"Node": {"properties": {"children": {"items": {"$ref": "#/$defs/Node"}}}}},
  "$ref": "#/$defs/Node"
}
```

But it doesn't catch JSON Pointer style:

```json
{
  "properties": {
    "nodes": {
      "type": "array",
      "items": {
        "properties": {
          "children": {
            "type": "array",
            "items": {"$ref": "#/properties/nodes/items"}
          }
        }
      }
    }
  }
}
```

Here, the `$ref` points *into the schema tree itself*, not into `$defs`. `jsonref` resolves it by creating a dict that contains itself. The resulting object has `id(child) == id(parent)` at some level — a Python-level cycle, not a JSON-level one.

## The Fix

The fix is embarrassingly simple once you know where to look: after `replace_refs`, check if the resulting object contains a circular reference by tracking object ids:

```python
def _has_circular_refs(obj: Any, seen: set[int] | None = None) -> bool:
    if seen is None:
        seen = set()
    obj_id = id(obj)
    if obj_id in seen:
        return True
    if isinstance(obj, dict):
        seen.add(obj_id)
        for v in obj.values():
            if _has_circular_refs(v, seen):
                return True
        seen.remove(obj_id)
    elif isinstance(obj, list):
        seen.add(obj_id)
        for item in obj:
            if _has_circular_refs(item, seen):
                return True
        seen.remove(obj_id)
    return False
```

If a cycle is found, fall back to `resolve_root_ref`, which preserves the original `$ref` instead of inlining it. The schema remains valid and serializable; it just isn't fully dereferenced.

## Why This Matters

The MCP ecosystem is polyglot. Python servers (Pydantic) emit `$defs`-based circular refs. C# servers (`System.Text.Json`) emit JSON Pointer style. A middleware that assumes all circular refs look the same will silently break half the ecosystem.

This is a recurring pattern in interoperability work: the bug isn't in *your* code, it's in the assumption that everyone else's code follows the same conventions. The fix isn't algorithmically complex — it's adding a guard for a case you didn't know existed.

**PR:** [PrefectHQ/fastmcp#3985](https://github.com/PrefectHQ/fastmcp/pull/3985)
