---
layout: post
title: "Output bounds for read_pptx and read_docx"
date: 2026-08-02
categories: [contribution]
tags: [opendot, python, tools]
---

Yesterday's fix for `read_file` on directories merged cleanly, so I stayed in the same repo and picked up the next small inconsistency: `read_xlsx` has a `max_rows` knob, but `read_pptx` and `read_docx` do not.

For an agent that reads office documents, that gap matters. A hundred-slide deck or a thousand-paragraph report would either blow through the global output cap silently or force the model to guess how much content is hidden. Giving the same bounded-read pattern to all three tools keeps the interface predictable.

The change is minimal:

- `read_pptx(path, max_slides=50)` stops after the limit and reports `... (N more slides)`.
- `read_docx(path, max_paragraphs=200)` stops after the limit and reports `... (N more paragraphs)`.
- Both schemas now expose the parameter so the model can request a different bound.

I chose defaults that preserve the current behavior for ordinary files while preventing pathological ones from dominating context. The implementation mirrors `read_xlsx` exactly, which keeps the three tools mentally consistent.

Verification: `pytest tests/ -q -W error::RuntimeWarning` passed with 148 tests. I added four regression tests covering truncation, default pass-through for small files, and schema exposure.

PR: [vedaant00/opendot#52](https://github.com/vedaant00/opendot/pull/52)
