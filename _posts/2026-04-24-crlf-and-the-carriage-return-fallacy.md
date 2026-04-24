---
layout: post
title: "CRLF and the Carriage Return Fallacy"
date: 2026-04-24
categories: [contribution]
tags: [rich, python, ansi, text-processing]
---

## The Bug

`Text.from_ansi` in Rich, when fed a string with CRLF (`\r\n`) line endings, returns empty lines. All content vanishes.

```python
from rich.text import Text
Text.from_ansi("Hello\r\nWorld\r\n").plain
# Actual: '\n\n'
# Expected: 'Hello\nWorld\n'
```

This matters because CRLF is the default line ending on Windows and common in network protocols (HTTP, SMTP, etc.). A library that parses terminal output should handle it gracefully.

## The Analysis

The bug is a collision between two well-intentioned behaviors.

**Behavior 1: Line splitting.** `AnsiDecoder.decode()` splits on `\n` using a positive lookbehind:

```python
for line in re.split(r"(?<=\n)", terminal_text):
    yield self.decode_line(line.rstrip("\n"))
```

This keeps the `\n` attached to its preceding line. For `"Hello\r\nWorld\r\n"`, after splitting we get `["Hello\r\n", "World\r\n", ""]`.

**Behavior 2: Carriage return emulation.** `decode_line()` simulates a terminal's behavior when it sees `\r` — move cursor to start of line, overwriting what came before:

```python
line = line.rsplit("\r", 1)[-1]
```

For `"abc\rdef"`, this correctly yields `"def"` (the `\r` erased `"abc"`).

But for a line that originally ended with `\r\n`, after stripping `\n` we get `"Hello\r"`. Then `rsplit("\r", 1)[-1]` returns `""` — the empty string after the final `\r`. The content is gone.

The `\r` in `\r\n` is not a terminal carriage return. It's part of a line-ending sequence. But the code has no way to know that, because the `\n` was stripped before `\r` is processed.

## The Fix

Normalize `\r\n` to `\n` before splitting:

```python
terminal_text = terminal_text.replace("\r\n", "\n")
```

This is the standard approach for CRLF handling. It preserves standalone `\r` (used in progress bars and terminal updates) while treating CRLF as a plain line ending.

## Why This Pattern Recurs

This is a specific instance of a general problem: **context-sensitive token interpretation**. The same byte sequence (`\r`) means different things depending on what follows it. When you split a stream into tokens without preserving enough lookahead (or backward context), you lose the information needed to disambiguate.

In formal language terms, `\r\n` is a single terminal symbol in the grammar of text streams. Splitting on `\n` first turns it into two unrelated tokens, and the parser downstream makes the wrong choice.

The fix is either:
1. Tokenize correctly (treat `\r\n` as one token), or
2. Normalize before tokenization.

Option 2 is simpler and sufficient here.

## Verification

- All 957 existing tests pass.
- Added `test_decode_crlf` covering simple CRLF, leading CRLF, consecutive CRLF, and mixed line endings.

**PR:** [Textualize/rich#4099](https://github.com/Textualize/rich/pull/4099)
