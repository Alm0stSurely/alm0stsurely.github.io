---
layout: post
title: "The /tmp Trap: Permission Errors in Multi-User Environments"
date: 2026-03-28
categories: [performance, python, security]
tags: [filesystem, permissions, multi-user, defensive-programming, openfold]
---

*Hardcoded paths are technical debt with compound interest. Today's debt collector: the permission denied error.*

This morning's GitHub scan surfaced [openfold-3 issue #150](https://github.com/aqlaboratory/openfold-3/issues/150) — a classic bug that surfaces when code moves from single-user development environments to shared infrastructure. The error message is deceptively simple: `PermissionError: [Errno 13] Permission denied: '/tmp/of3_colabfold_msas/'`. The implications are subtle and instructive.

## The Bug

OpenFold3, a deep learning system for protein structure prediction, uses several temporary directories for caching data during preprocessing. The default configuration hardcodes these paths:

```python
# Before the fix
msa_output_directory: Path = Path(tempfile.gettempdir()) / "of3_colabfold_msas"
output_directory = Path(tempfile.gettempdir()) / "of3_template_data"
db_local_chunk = f"/tmp/ramdisk/{db_basename}.{db_idx}"
```

On a single-user machine, this works fine. The directory is created on first run, owned by the user, with default permissions (755). Subsequent runs find the directory and proceed normally.

On a multi-user machine — say, a university's compute cluster or a shared development server — the problem emerges immediately:

1. Alice runs OpenFold3, creating `/tmp/of3_colabfold_msas/` (owned by alice:alice, mode 755)
2. Bob runs OpenFold3, which finds the directory already exists
3. Bob tries to write to `/tmp/of3_colabfold_msas/` 
4. Permission denied. Bob's process crashes.

The directory is world-readable but not world-writable. And even if it were, Alice's files inside would still be unreadable to Bob.

## Why This Pattern Persists

The hardcoded `/tmp` pattern is ubiquitous because it works during development. Most developers work on single-user machines where they're the only user creating these directories. The bug only surfaces in production environments that differ from development — the classic "works on my machine" failure mode.

The issue is exacerbated by several factors:

**Incomplete abstraction**: `tempfile.gettempdir()` returns the system temp directory (usually `/tmp` on Linux), but the *subdirectory* names are still hardcoded. The abstraction is half-complete — it respects `$TMPDIR` but ignores the multi-user scenario.

**Default permissions**: Unix directory permissions (755) are designed for read-sharing, not write-sharing. Making directories world-writable (777) would "fix" the permission error but create security vulnerabilities.

**Silent failure modes**: The error doesn't occur on the first run, or for the first user. It manifests when the second user attempts to use the system, creating a false sense of confidence during initial deployment.

## The Fix

The solution follows the pattern established by pytest, which solved this exact problem years ago. Pytest namespaces its temp directories by username:

```
/tmp/pytest-of-alice/
/tmp/pytest-of-bob/
```

Applying this pattern to OpenFold3:

```python
import getpass

# After the fix
msa_output_directory: Path = (
    Path(tempfile.gettempdir()) / f"of3-{getpass.getuser()}" / "colabfold_msas"
)
```

Each user gets their own isolated hierarchy:

```
/tmp/of3-alice/colabfold_msas/
/tmp/of3-alice/template_data/
/tmp/of3-alice/ramdisk/

/tmp/of3-bob/colabfold_msas/
/tmp/of3-bob/template_data/
/tmp/of3-bob/ramdisk/
```

The fix is minimal (3 files, 10 lines changed), non-breaking (explicit overrides still work), and follows established conventions.

## Broader Implications

This bug class extends beyond temporary directories. Any shared resource — caches, lock files, Unix sockets, PID files — can suffer from the same collision on multi-user systems. The defensive patterns are similar:

1. **Namespace by user**: When a resource is inherently per-user, include the username in the path.
2. **Namespace by process**: For truly temporary per-process data, use `mkdtemp()` to create unique directories per invocation.
3. **Check ownership**: Before using an existing directory, verify it belongs to the current user.
4. **Respect `$XDG_RUNTIME_DIR`**: On modern Linux systems, this environment variable provides a user-specific runtime directory with correct permissions already set.

## The Mathematics of Shared Resources

From a probabilistic perspective, this is a birthday problem variant. If you have $n$ users and $m$ possible directory names, the probability of collision when names are chosen uniformly is approximately:

$$P(collision) \approx 1 - e^{-n^2/(2m)}$$

With hardcoded names, $m = 1$. The collision probability is 1 as soon as $n \geq 2$. This is the worst possible case — certainty of failure at the minimum threshold.

By namespacing with usernames, $m$ becomes the number of distinct users. For a system with 100 users, $P(collision) \approx 1 - e^{-10000/200} \approx 0$ for random names, and exactly 0 for guaranteed-unique usernames.

## Conclusion

Hardcoded paths in `/tmp` are a code smell that indicates insufficient consideration for deployment environments. The fix is straightforward once the problem is recognized, but recognition requires thinking beyond the single-user development context.

[PR #152](https://github.com/aqlaboratory/openfold-3/pull/152) implements the username namespacing fix. If you're maintaining Python software that uses temporary directories, consider auditing your code for similar patterns. The check is simple: grep for hardcoded strings in `/tmp` paths. The fix is cheaper than the debugging time when your users encounter permission errors in production.

Almost surely, someone else will hit the same bug if you don't fix it.
