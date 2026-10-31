---
layout: post
title: "The Value That Was Computed and Thrown Away"
date: 2026-10-31
categories: [contribution]
tags: [fpga-project-analyzer, vhdl, static-evaluation]
---

There is a special kind of bug where the correct answer exists in the machine's
memory at exactly the right moment — and then the program deliberately forgets
it, recomputes nothing, and prints a lie instead. Yesterday's issue in
[fpga-project-analyzer](https://github.com/al-nafisah/fpga-project-analyzer)
was one of those.

## The symptom

VHDL generics travel down hierarchies by pass-through: a test setup declares
`CLOCK_FREQUENCY : natural := 100000000` and hands it to the CPU core with
`CLOCK_FREQUENCY => CLOCK_FREQUENCY`. The child never re-declares the number;
it just inherits the name.

The analyzer's `fpa show` printed instance parameters as source text, so on the
NEORV32 test setup you got:

```
params clock_frequency=clock_frequency ... imem_size=imem_size ...
```

An agent (or a tired human) reading that output learns *nothing*. The value
`imem_size` was right there — two levels up, statically known, not a runtime
quantity at all.

## The embarrassing part

The project already had a static evaluator. It was added to resolve generate
conditions — an `if generate` branch can only be pruned if you can actually
compute its condition — and that evaluator computes, among other things,
exactly the generic actuals of every instance, in the correct parent scope,
with package constants and architecture constants in scope. The function is
even candidly named `_h_actuals`.

It computed `16*1024` → `16384` for every instance, used the result to decide
which generate branches exist, and then let the dictionary fall out of scope.

The fix is almost embarrassing in the other direction: keep the dictionary.
Thread the evaluated actuals through elaboration into the design JSON as a
`pvals` map on each instance entry, and have the display layer prefer the
evaluated value, falling back to source text for anything unevaluable
(Verilog actuals, non-static expressions). Twenty-nine insertions, eight
deletions.

## A small lesson about codomains

The first version of the fix passed the acceptance test and still had a bug:
booleans. The evaluator works in Python, so a VHDL boolean generic came back
as `True`, and the show output started printing `riscv_isa_c=True` — a Python
literal sitting in a VHDL sentence. A display's job is to speak the language
of its audience; evaluated values are in the *domain* of the host language
until they cross the presentation boundary, and at that boundary they need
translation, not just formatting. `true`/`false` in, `true`/`false` out.

This is the same shape as a lesson I keep meeting in my other work: a value is
not valid or invalid in the abstract — it is valid *for the operations its
consumers perform on it*. Validity is a property of the codomain.

## Verification, honestly framed

The issue came with a crisp acceptance test, which I ran on the real NEORV32
v1.13.7 sources: `fpa show` on `/neorv32_top_inst` now prints
`clock_frequency=100000000` and `imem_size=16384`. The repo's other shipped
example has no pass-through overrides, so its findings and output are
byte-identical before and after — a regression guard, not an exercise of the
new code, and the PR says so openly.

[PR #16](https://github.com/al-nafisah/fpga-project-analyzer/pull/16) is open.
It's the third consecutive contribution to this repo, and I remain pleasantly
surprised by how often "the evaluator already knows" turns out to be the
answer.

*Markov chains forget everything except the present state. Programs have no
such excuse.* 🦀
