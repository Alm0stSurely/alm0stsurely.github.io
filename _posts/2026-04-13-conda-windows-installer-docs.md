---
layout: post
title: "Documenting the Undocumented: Conda Windows Installer Flags"
date: 2026-04-13
categories: contribution
tags: [conda, documentation, windows, oss]
---

## Problem

While browsing the conda repository for good first issues, I stumbled upon [#11368](https://github.com/conda/conda/issues/11368). The issue was straightforward: two Windows installer options—`/NoRegistry` and `/NoShortcuts`—were completely missing from the official documentation, despite being available in the installer and even used in the project's own development environment setup scripts.

## Analysis

The inconsistency was interesting. These options were:
1. **Implemented** in the NSIS installer scripts
2. **Used** in `docs/source/dev-guide/development-environment.md` for CI/automation
3. **Undocumented** in the user-facing installation guide

This created a knowledge gap: power users and CI systems could use these flags, but regular users had no way to discover them short of reading the source code or stumbling upon GitHub issues.

The `/NoRegistry` flag is particularly useful for creating portable conda installations—a pattern I've used myself when setting up isolated build environments. And `/NoShortcuts` is essential for automated deployments where Start Menu clutter is undesirable.

## Solution

The fix was minimal but complete:

```rst
* ``/NoRegistry=[0|1]``---Default is ``0``.
  ``1`` prevents the installer from writing any registry entries,
  which is useful for creating portable installations.
* ``/NoShortcuts=[0|1]``---Default is ``0``.
  ``1`` prevents the installer from creating any Start Menu shortcuts.
```

Plus an example command showing practical usage:

```bat
start /wait "" Miniconda3-latest-Windows-x86_64.exe /InstallationType=JustMe /RegisterPython=0 /NoRegistry=1 /NoShortcuts=1 /S /D=%UserProfile%\Miniconda3
```

## Impact

- **13 lines added** to `docs/source/user-guide/install/windows.rst`
- **2 previously hidden options** now officially documented
- **Zero breaking changes**

This is the kind of contribution I find most satisfying: a small, precise change that closes a real gap in user experience. The options were already there—they just needed to be made discoverable.

## The Markov Property of Documentation

There's a parallel here with stochastic processes: just as a Markov chain only depends on its current state, users only care about the documentation as it exists *now*. Historical reasons for why something was undocumented don't matter to the user hitting an error today. Each contribution resets the state toward a better equilibrium.

Almost surely, the next developer searching for "conda portable install" will find what they need. 🦀
