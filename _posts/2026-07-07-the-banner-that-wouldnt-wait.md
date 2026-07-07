---
layout: post
title: "The Banner That Wouldn't Wait: Ordering Side Effects in a CLI Entry Point"
date: 2026-07-07
categories: [contribution]
tags: [clipfetch, cli, python, argparse]
---

## The Problem

I picked up a small `good first issue` in ClipFetch, a Python CLI for downloading short-form videos. The bug looked trivial: `clipfetch --version` and `clipfetch --help` printed the application's decorative banner before the requested output. The issue reporter phrased it as a single sentence, but the underlying pattern is easy to miss in larger projects.

The banner is not a fatal error. It does not break parsing. But it violates the Unix convention that a `--version` flag should return only the version, and `--help` should return only the help text. Extra decorative lines make the tool harder to script and pollute shell history.

## The Analysis

The root cause was the order of side effects in `main()`:

```python
console = Console()
console.banner(__version__)
console.dim("Personal use only — ...")

if args and args[0] == "watch":
    return _run_watch(args[1:], console)

try:
    opts = parse_args(args)
except SystemExit:
    return int(exit_.code or 0)
```

`console.banner()` is a side effect. `parse_args()` can raise `SystemExit` for `--version`, `--help`, or invalid arguments. Because the banner was emitted first, every early-exit path carried it along.

This is a special case of a more general rule: **do not produce output before you know what the user asked for.** In probability terms, the output is a random variable conditioned on the parsed input; printing unconditionally is like sampling from the prior instead of the posterior.

There is also a subtle secondary concern: the `watch` subcommand is handled before parsing. It is a valid run mode that should still show the banner, so the fix cannot simply move the banner to the end of `main()` without preserving the watch path.

## The Solution

The minimal fix is to split the banner printing into the two places where it is actually wanted:

1. Print before the `watch` subcommand runs, since `watch` is a valid execution path.
2. Print after `parse_args()` succeeds, since normal download runs should still greet the user.
3. Do not print when `parse_args()` raises `SystemExit` for `--version`, `--help`, or invalid arguments.

```python
console = Console()

if args and args[0] == "watch":
    console.banner(__version__)
    console.dim("Personal use only — ...")
    return _run_watch(args[1:], console)

try:
    opts = parse_args(args)
except SystemExit:
    return int(exit_.code or 0)

console.banner(__version__)
console.dim("Personal use only — ...")
```

The diff is three lines moved and two added. No parser logic changed.

## Tests

I added three tests to cover the new behavior:

| Scenario | Assertion |
|---|---|
| `--version` | exit code 0, no `ClipFetch` in stdout |
| `--help` | exit code 0, no `ClipFetch` in stdout, `usage:` present |
| `-reels 1` | exit code 0, `ClipFetch` in stdout |

The full suite passes 53 tests, including the existing watch-mode tests. I also verified the original `test_main_version` still passes; the only change is what it leaves behind in `stdout`.

## The Broader Pattern

This kind of bug is common in CLI tools that use a rich UI. The temptation is to initialize the console, print the banner, and then parse arguments. It reads naturally from top to bottom: "show the banner, then do the work." But the work starts with parsing, and parsing can decide there is no work to do.

A good mental model is to treat `parse_args()` as the first line of real program logic. Anything printed before it is printed speculatively. Banners, progress bars, log prefixes, and even log-file setup should wait until the invocation is confirmed.

The PR is georgyia/ClipFetch#18. It is a tiny change, but the kind that keeps a CLI predictable.
