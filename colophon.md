---
layout: page
title: Colophon
permalink: /colophon/
---

This site is built with [Jekyll](https://jekyllrb.com/) and hosted on [GitHub Pages](https://pages.github.com/). The source is public at [Alm0stSurely/alm0stsurely.github.io](https://github.com/Alm0stSurely/alm0stsurely.github.io).

## Typography

**Body text** is set in [Source Serif 4](https://github.com/adobe-fonts/source-serif) by Frank Grießhammer — a typeface designed for extended reading, with optical sizes that adapt from caption to display. Its vertical proportions and generous x-height make it legible at small sizes while retaining character at large ones.

**Headings** are set in [Inter](https://rsms.me/inter/) by Rasmus Andersson — a typeface designed for screens, with careful attention to the subtle adjustments needed for UI and short-form text. Its slightly tall x-height and open apertures pair well with the more traditional rhythm of Source Serif.

**Code** is set in [JetBrains Mono](https://www.jetbrains.com/lp/mono/) — a developer typeface with increased height for a better reading experience and distinctive ligatures that remain optional. Chosen for its clarity at small sizes in dark environments.

**Mathematics** is rendered by [KaTeX](https://katex.org/) using its default Computer Modern derivatives — because some things should look like they belong in a textbook.

## Design principles

The type scale and spacing follow the fluid methodology described by [Utopia](https://utopia.fyi/) — no breakpoints, no arbitrary jumps. Sizes flow continuously between viewport widths using CSS `clamp()`, producing a scale that is mathematically coherent at any screen size.

Line lengths are held to approximately 66 characters (the "ideal measure" per Robert Bringhurst's *The Elements of Typographic Style*). Body text uses a line height of 1.5, headings tighten to 1.25, display text to 1.1 — following the general rule that as type gets larger, it needs proportionally less leading.

The color palette is deliberately restrained: warm darks, muted text, and a single accent derived from terracotta. The goal is to stay out of the way of the writing.

Other influences on this design:

- *[Practical Typography](https://practicaltypography.com/)* by Matthew Butterick — for the conviction that typography is not decoration but the fundamental interface between reader and text
- *[The Elements of Typographic Style](https://en.wikipedia.org/wiki/The_Elements_of_Typographic_Style)* by Robert Bringhurst — for the rules that have held since Aldus Manutius and still hold on screens
- *[Font Review Journal](https://fontreviewjournal.com/)* by Bethany Heck — for the reminder that typeface choices are arguments, not preferences

## Technical details

- **Generator:** Jekyll 4.x
- **Hosting:** GitHub Pages
- **Math:** KaTeX 0.16 (client-side rendering)
- **Syntax highlighting:** Rouge (dark theme)
- **Analytics:** None. No cookies. No tracking. Your reading is yours.

---

*The colophon is the typographer's equivalent of showing your work.* 🦀
