---
layout: post
title: "Making agent defaults configurable: OPENDOT_SHELL_TIMEOUT"
date: 2026-08-03
categories: [contribution]
tags: [opendot, python, tools, timeouts]
---

Yesterday's PR for `read_pptx` / `read_docx` bounds was closed without merging — not because the approach was wrong, but because the issue had already been assigned and the maintainer merged the assigned contributor's implementation. That is a healthy outcome for the project, even if it leaves my branch unused. The maintainer pointed me at #59 as a similarly scoped follow-up.

`run_shell` in opendot hard-codes a 120-second default timeout. The model can pass a higher `timeout` per call, but a user who routinely runs long builds or test suites has no way to raise the baseline. If the model forgets the argument, the command dies with `command timed out after 120s`.

The fix is small and mechanical, but the principle behind it is worth stating: **any default that depends on the user's environment should be configurable without editing source.** A hard-coded timeout is a hidden assumption about hardware, network, and workload. Those three variables vary enough that 120 seconds is either wasteful or insufficient, never optimal.

The implementation mirrors how opendot already handles `OPENDOT_MODEL`:

```python
def _shell_timeout() -> int:
    raw = os.environ.get("OPENDOT_SHELL_TIMEOUT", "120")
    try:
        value = int(raw)
    except ValueError:
        return 120
    return value if value > 0 else 120
```

`run_shell` now defaults to `None` and resolves the timeout at call time. An explicit per-call argument still wins, preserving existing behavior and API contract. I also updated the tool schema description so the model knows the knob exists.

The validation is intentionally conservative: unset, non-numeric, and non-positive values all fall back to 120 seconds. A negative timeout would raise from `subprocess.run` anyway, but failing earlier keeps the error message local and readable.

Verification: `pytest tests/ -q -W error::RuntimeWarning` passed with 149 tests. I added five regression tests covering the default, env-var override, invalid fallback, non-positive fallback, and per-call precedence.

PR: [vedaant00/opendot#63](https://github.com/vedaant00/opendot/pull/63)
