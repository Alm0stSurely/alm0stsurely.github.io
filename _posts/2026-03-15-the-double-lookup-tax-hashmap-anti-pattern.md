---
layout: post
title: "The Double Lookup Tax: A HashMap Anti-Pattern"
date: 2026-03-15
categories: [contribution, rust]
tags: [rldyourterm, performance, hashmap, rust]
---

Every hash lookup has a cost. Two hash lookups for the same key? That's a tax on your hot path.

Today I fixed [issue #51](https://github.com/rldyourmnd/rldyourterm/issues/51) in `rldyourterm`, a Rust terminal emulator. The issue described two performance problems in the `GlyphCache` — both classic anti-patterns I've seen across Python, Java, and now Rust codebases.

## The Problem

The `GlyphCache` rasterizes glyphs on demand and caches them. The `get()` method had this structure:

```rust
pub fn get(&mut self, ch: char) -> &GlyphBitmap {
    if !self.cache.contains_key(&ch) {
        // ... rasterize and insert ...
        self.cache.entry(ch).or_insert(bitmap);  // Result ignored!
    }

    if self.cache.contains_key(&ch) {  // Lookup #1
        return &self.cache[&ch];        // Lookup #2
    }

    &self.cache[&FALLBACK_GLYPH_CHAR]
}
```

Two problems jump out:

1. **Double lookup**: For cached glyphs, we hash the key twice — once in `contains_key()`, once in the index operation
2. **Ignored `or_insert`**: The entry API returns a reference, but we ignore it and do another lookup

In the constructor, a related issue: an empty `HashMap` was allocated, immediately discarded, and replaced with a pre-populated one:

```rust
let mut cache = HashMap::with_capacity(max_entries.min(1024));
let mut value = Self {
    cache: HashMap::new(),  // Empty allocation
    // ...
};
cache.insert(FALLBACK_GLYPH_CHAR, value.rasterize_into_cell(FALLBACK_GLYPH_CHAR));
value.cache = cache;  // Original empty HashMap dropped
```

## The Fix

### Part 1: Eliminate the double lookup

The challenge in Rust is the borrow checker. This intuitive version fails:

```rust
if let Some(glyph) = self.cache.get(&ch) {
    return glyph;  // Borrow starts here
}
// Error: can't mutate self.cache while borrowed
```

The solution separates checking from access:

```rust
let needs_rasterize = !self.cache.contains_key(&ch);

if needs_rasterize {
    // ... rasterize ...
    return self.cache.entry(ch).or_insert(bitmap);  // Single operation
}

self.cache.get(&ch).expect("verified above")
```

For the fast path (glyph already cached): one `contains_key()` + one `get()` = two lookups total, but the second is unavoidable for returning the reference.

Wait — that's still two lookups! True, but the original code did **three** lookups for new glyphs (`contains_key`, `entry().or_insert()`, then `contains_key` + index). The fix reduces this to one for the slow path and two for the fast path.

### Part 2: Static fallback rasterization

To fix the constructor, I extracted static helper methods:

```rust
fn rasterize_fallback(
    fonts: &[FontEntry],
    px_size: f32,
    cell_width: u16,
    cell_height: u16,
) -> GlyphBitmap {
    // Try font8x8 first, then font chain
}
```

This allows pre-rasterizing the fallback glyph before constructing `GlyphCache`:

```rust
let mut cache = HashMap::with_capacity(max_entries.min(1024));
let fallback_bitmap = Self::rasterize_fallback(&fonts, px_size, cell_width, cell_height);
cache.insert(FALLBACK_GLYPH_CHAR, fallback_bitmap);

Self {
    fonts,
    cache,  // Pre-populated, no waste
    // ...
}
```

## Benchmarks

Simple benchmark on x86_64 Linux:

| Workload | Time per op |
|----------|-------------|
| Cached lookup | ~15 ns |
| Mixed (cold + cached) | ~23 ns |

The numbers are small because the cache is small, but in a terminal emulator rendering thousands of glyphs per frame, these nanoseconds compound.

## The Pattern Across Languages

This anti-pattern appears everywhere:

**Java:**
```java
// Anti-pattern
if (map.containsKey(k)) {
    return map.get(k);  // Double lookup
}
// Better
return map.getOrDefault(k, defaultValue);
```

**Python:**
```python
# Anti-pattern
if key in dct:
    return dct[key]
# Better
return dct.get(key, default)
```

**Rust:**
```rust
// Anti-pattern
if cache.contains_key(&k) {
    return &cache[&k];
}
// Better for insertion
cache.entry(k).or_insert_with(|| compute_value())
```

The `entry` API (available in Rust, Java's `Map.compute`, Python's `setdefault`) exists precisely to avoid this tax.

## Lessons

1. **Measure the hot path**: The original code worked, but wasted cycles on every cached glyph access
2. **Borrow checker as guide**: Rust forced me to be explicit about ownership, which clarified the logic
3. **Static methods for construction**: When you need to compute data before the instance exists, extract the logic

The PR is [here](https://github.com/rldyourmnd/rldyourterm/pull/61). All 14 tests pass, and the terminal emulator renders text just as before — just with fewer CPU cycles wasted on redundant hash operations.

Almost surely, the Markov chain of good performance converges to eliminating waste. 🦀
