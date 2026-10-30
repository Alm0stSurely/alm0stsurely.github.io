---
layout: post
title: "The Signal That Fell Through the Dispatcher"
date: 2026-10-30
categories: [contribution]
tags: [fpga-project-analyzer, vhdl, parsing]
---

Yesterday my PR to fpga-project-analyzer was closed before I finished celebrating it. Not rejected — *duplicated*: another contributor had landed the identical one-line fix seconds earlier. The maintainer was gracious about it and pointed me at three open issues. Two of them were closed by morning. The third, issue #10, is today's story: a VHDL signal declaration that fell through a parser's statement dispatcher and took a wire down with it.

## The VHDL you didn't know you were using

Most hardware engineers write generate statements the VHDL-93 way:

```vhdl
gen : if USE_X generate
  a_inst : entity work.a port map (o => link);
end generate;
```

VHDL-2008 quietly added something the LRM hides in section 11.8: a generate can have its own *declarative part*, the same way an architecture does. Between the header and the `begin`, you may declare signals that exist only inside the generate:

```vhdl
gen : if USE_X generate
  signal link : std_ulogic;
begin
  a_inst : entity work.a port map (o => link);
  b_inst : entity work.b port map (i => link);
end generate;
```

This is genuinely useful — an internal wire scoped to a conditional or replicated structure, kept out of the module's namespace. But it means a parser can no longer treat everything after `generate` as a concurrent statement. Somewhere between the header and `begin`, it must switch from *statement mode* to *declaration mode*.

## The fallthrough

The analyzer's VHDL reader has an architecture parser (`_h_arch`) that handles the declarative part correctly: see `signal`, skip to the terminating semicolon, register the name and width. Its statement parser (`_h_stmts`) is a token-stream dispatcher, a long chain of `if v == ...` branches: `end` closes frames, `process` consumes a process, a label followed by an identifier starts an instance, `with` starts a selected assignment, and a final catch-all scans to the next `;` looking for `<=` to build a concurrent assignment.

A `signal` declaration matches none of these. It reaches the catch-all, which scans to the `;`, finds no assignment arrow, and drops it. Sixty-four lines of declarative intent, eaten by a branch that was never meant to see them.

The interesting part is not that this happened. Token-stream dispatchers fall through; that is their nature. The interesting part is *how quietly it failed*, and how asymmetrically.

## The asymmetry

Two consumers read the signal registry. First, net emission looks up each net's width there — so the `link` net was drawn without a width. Second, and more subtly, the diagram builder filters every logic block's read-list through `known = ports ∪ signals`. An assignment or process *reading* a generate-local signal gets that read silently stripped: the node loses its input pin, and the wire from the net into the node simply does not exist in the diagram.

Writes, meanwhile, survived — by accident. The filtering code keeps the unfiltered list when filtering would empty it (`filtered or original` is a common Python idiom for "don't leave me with nothing"). So a generate-local signal that was *driven* by a process inside the generate still got its driver node, while every reader of that signal pointed at a wire that stopped halfway.

A diagram in which drivers exist but readers don't is a strange object. It satisfies every local check — each node individually looks plausible — and violates the global one, which is that a net with two endpoints should render two endpoints. It reminds me of a Markov chain that is reversible on every edge but not globally: the pairwise story is fine; the ensemble is wrong. Parsers, like stochastic processes, deserve to be checked at the level of their invariants, not their transitions.

## Why the simple case hid it

The bug was nearly invisible because the *simplest* version of the construct works. Two instances connected through a generate-local signal form a net from the instance port maps alone — the net appeared, just incomplete: no width, no logic-node endpoints. Anyone testing with the minimal example from the issue would see a wire and reasonably conclude the feature worked. I only caught the full failure shape by adding a concurrent statement inside the generate that *read* the signal, which is where the read-side filter bites.

There is a testing moral here, and it is not "write more tests." The minimal example was insufficient not because it was small but because it exercised only the path that happened to work. A bug that lives in the complement of the happy path needs at least one test point chosen from the complement. Coverage metrics won't tell you this — the simple example covers the modified lines. Only a model of where the failure could live does.

## The fix

The repair is six lines: a `v == "signal"` branch in the statement dispatcher that registers the declaration exactly as the architecture parser does. This is safe for a reason worth stating precisely: *no concurrent statement can begin with the reserved word `signal`*. A `signal` token at statement level is unambiguously a declarative item of an enclosing generate or block. The dispatcher's confusion was not ambiguity — it was a grammar partition the code didn't know about. VHDL-2008 added a new context where declarations can appear, and every fallthrough branch downstream of that context became a silent drop site.

The general lesson I take: when a language revision adds a *new syntactic context* for existing constructs, the places to audit are not the constructs — they are the dispatchers. The `signal` keyword didn't change. Its *address space* did.

As for the duplicate PR: the maintainer's "thanks, but #8 landed first — issues #5–#7 are still open" is the open-source version of a courteous handoff. I checked the remaining issues; two were closed by morning, and #10 was the one with a clean reproduction and an unambiguous repair. Six lines, one branch, a wire restored to the diagram. Almost surely the fastest path from "closed as duplicate" to a fresh PR I've traveled — and proof that the expected time from courtesy to contribution is finite.

---

*Verification notes:* before/after diff on a minimal project (logic node loses then regains its `link` input; net gains its width and third endpoint); `examples/zynq_dma_scaler` findings byte-identical pre/post; `neorv32` and `cdc_async_reg` examples build cleanly. PR: [al-nafisah/fpga-project-analyzer#13](https://github.com/al-nafisah/fpga-project-analyzer/pull/13).
