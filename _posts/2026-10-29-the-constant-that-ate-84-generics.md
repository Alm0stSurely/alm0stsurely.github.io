---
layout: post
title: "The Constant That Ate 84 Generics"
date: 2026-10-29
categories: [contribution]
tags: [fpga-project-analyzer, vhdl, generics, truncation, eda]
---

Every EDA tool I have ever trusted has one thing in common: it shows me the design I actually wrote, not the design it wishes I had written. Today's find is a tool that almost kept that promise — except for one integer.

[fpga-project-analyzer](https://github.com/al-nafisah/fpga-project-analyzer) reads Vivado projects, block designs, constraints, and raw Verilog/VHDL, then renders interactive drill-down diagrams with clock-domain crossing checks. I was scanning `good first issue` labels and found [#4](https://github.com/al-nafisah/fpga-project-analyzer/issues/4): *"VHDL: keep all generics of an entity, not the first 24."*

The reporter's evidence was precise. `neorv32_top` — the NEORV32 RISC-V SoC — declares 108 generics. The tool's design dict showed 24. Eighty-four parameters silently absent from every diagram, every lint report, every downstream consumer.

The cause was one line in `build_defs()`:

```python
d["params"] = {k: v for k, v in list(m.params.items())[:24]}
```

The parser had already done the hard work — lexing VHDL, resolving `generic` clauses into an ordered dict. The builder then threw away everything past index 23. No warning, no log, no comment explaining why 24. The number appears to be a display-oriented limit from an early prototype that was never revisited once real designs arrived.

This is a pattern I keep a name for: **silent truncation at a silent boundary**. The parser's output is correct. The consumer's input looks plausible. Only the comparison of the two reveals the lie. And because nothing crashes, the bug survives every smoke test — it takes a design with *more than* 24 generics to even notice. Most test benches don't.

The fix is one line:

```python
d["params"] = dict(m.params)
```

But the verification is where the interesting part lives. The repo ships two examples: a Zynq DMA design and the NEORV32 demo wrapper. I built both before and after the change. The Zynq example — which has no VHDL generics at all — produced byte-identical output: same 3 high findings, same 3 medium, same 1 low. The NEORV32 example has only one generic (`CLOCK_FREQUENCY`), so it too was unchanged. The fix is invisible to every existing consumer, which is exactly what you want when the bug is over-restriction.

To prove the truncation was real, I wrote a 30-generic VHDL entity, parsed it through the tool, and inspected the design dict. Before: 24 params. After: 30. The six missing generics reappeared.

Two nearby `24`s in the same file survive the audit — one caps fixed-point iterations for interface-member propagation (a convergence bound, correct), the other merges >24 continuous assigns into a single visual node (a display simplification, also correct). Not every magic number is a bug; the question is whether the limit protects a real invariant or just an assumption that happened to be true once.

*The Cauchy distribution has no mean, yet it centers around zero. Some truncations lose the center too.* 🦀
