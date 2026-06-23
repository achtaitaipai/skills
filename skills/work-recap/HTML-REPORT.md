# Work Recap — HTML Report Format

This file is the **editorial brief** — it owns how the recap *reads*: its audience, structure, and voice. The **visual form** lives next to it in [`SKELETON.html`](./SKELETON.html): a single self-contained HTML file (inline CSS, no framework, no external requests, no diagram libraries) with the section markup, the `.theme--*` accent classes, and the design tokens documented inline. Fill the skeleton; don't restructure it. A recap is a narrative, not architecture docs — convey structure in prose, not boxes-and-arrows.

The skeleton's CSS borrows the readability fundamentals from Andy Bell's [_Some simple ways to make content look good_](https://piccalil.li/blog/some-simple-ways-to-make-content-look-good/) — a measured line length, a fluid type scale, generous line-height, and a `.flow` utility for vertical rhythm — and the accent palette from [Open Color](https://yeun.github.io/open-color/).

> This is a starter brief + skeleton. Replace either to control tone, structure, and vocabulary. The skill reads this file as the editorial brief and the skeleton it names as the visual form.

## Audience — precise *and* readable by a non-technical reader

Two readers at once: the engineer who wants the exact mechanism, and a non-technical reader (a manager, a client — anyone outside the codebase) who needs to know **what changed and why it matters**. Serve both, in this order:

- Lead every theme with a plain-language sentence: the effect, the user-visible or business consequence, the "so what". A non-technical reader should understand it without knowing the codebase.
- Then give the precise mechanism for whoever wants it. Don't dumb it down — the technical specifics stay.
- **Gloss jargon, don't strip it.** Pair a term with its consequence on first use: "a _named pipe_ (a channel two processes use to talk to each other)", "single-use tickets (a token that can only be redeemed once)". Identifiers (`ClassificationSuggestionEngine`, `myged://`) stay in monospace (`<code>` or `.mono`) but should never be the only thing a sentence conveys. (Examples here are in English to keep the template language-agnostic; write the actual recap in the requested document language.)

## Scaffold

The markup and CSS live in [`SKELETON.html`](./SKELETON.html). Start from it, fill the `{{placeholders}}`, and replace the example themes — don't rewrite the CSS or restructure the layout. The pieces it gives you, and how the sections below expect you to use them:

- **`<main class="flow">`** wraps a `header`, an `#overview` section, the category sections (`#features`, `#improvements`, `#fixes`, `#misc`), and `#loose-ends`. The `.flow` utility handles all vertical spacing; you don't add margins.
- **Category sections** — each `<section class="work flow">` with an `<h2>` heading holds the theme cards for one category. Omit any category that has nothing in it.
- **Theme cards** — one `<div class="theme theme--MODIFIER flow">` per theme, each with an `<h3>` title and one to three `<p>`. The modifier matches the category and sets the accent colour (see Style guidance). The `#misc` digest uses a single neutral `<div class="theme flow">` with a `<ul>` of one-liners instead.
- **Inline identifiers** go in `<code>` (or `<span class="mono">`); links use a plain `<a>`.

## Header

Repo name, the period covered (resolved dates), and a one-line stat strip (commits, files, ± lines). No throat-clearing — straight into the overview.

## Overview

A short paragraph — two at most. Someone who didn't follow the work reads only this and knows the focus of the period and what shipped. Lead with the headline. Resist letting it grow into a full recap of its own; the substance belongs in Work.

## Work — grouped by category

The substance, split into category sections in this order — **Features** (new capabilities), **Améliorations** (improvements to existing behaviour: perf, UX, refactor), **Fixes** (bug fixes), then an optional **Divers** digest. Omit any category that has nothing in it; never show an empty section. Within a category, order the cards biggest-first (most files / lines touched).

Each card:

- A short title naming the work in human terms.
- One or two sentences of what changed and why (mine the commit bodies for the why) — impact first, mechanism second.

Synthesize. Five commits on one feature are one card, not five. What's notable about a theme — the decision made, the hard problem wrestled with, the silent bug, the surprise — belongs **inside that card**, woven into its prose, not in a separate "notable" section. Commit bodies and scrappy `wip` commits earn their place here. State the consequence before the cause (the user-visible effect first, then the mechanism). Don't add a section that re-summarizes the work; the orientation is already in the overview and the substance is here.

**Divers** keeps the recap digestible: when a category would otherwise fill with small, self-explanatory items (a dependency bump, a typo fix, a one-line tweak), collect them as a single neutral card whose `<ul>` lists them as one-liners, rather than one card each. Use it only when it genuinely earns its place — a period with no such scraps omits it.

## Loose ends

If the history implies unfinished work or open questions, list them. Omit the section if there are none.

## Style guidance

- Lean editorial, not corporate-dashboard. (The skeleton supplies the look — serif body, bold sans headings, generous whitespace; this brief governs what you write into it.)
- **Colour by category, not decoration.** The accent comes from the card's category, applied via a `.theme--*` modifier — it follows the section the card lives in, so don't editorialise scope in prose:
  - **`.theme--feature`** (indigo) — new capabilities (the Features section).
  - **`.theme--improvement`** (green) — improvements to existing behaviour (the Améliorations section).
  - **`.theme--fix`** (neutral gray) — bug fixes (the Fixes section); deliberately low-key, routine.
  - **`.theme--warn`** (orange) — reserved for **Loose ends** / things needing attention, nothing else.
  - The Divers digest uses a bare `.theme` (neutral). Don't introduce a fourth hue; the grays carry the rest.
- **Sober editorial voice** — the register of a sharp engineer writing up the week, not a product changelog. Plain declarative sentences; let the facts carry the weight. No hedging ("it's worth noting that…"), no marketing gloss. If a sentence could be a bullet, make it a bullet; if a bullet could be cut, cut it.
- **Neutral register — describe, don't appraise.** Never rate the work's size, difficulty, importance, or drama. Drop qualifiers like "big", "major", "catastrophic", "elegant", "nightmare", "finally" — in prose, in theme titles, and in the overview alike. Convey magnitude with checkable facts (line/file counts, commit counts, "a full rewrite") and severity with the symptom, not with adjectives. A heading names the work ("Production incident in the rule matcher"), it doesn't characterise it ("The catastrophic incident"). A fixed technical term of art is exempt: name _catastrophic backtracking_ once as the precise phenomenon, but don't echo the qualifier as editorial colour.
- **Bold almost never** in the body prose. Highlighting a phrase in every paragraph stops meaning anything and reads as a template; the theme title carries the emphasis already.
- **No formulaic scaffolding.** Avoid label-led fragments ("Symptom: … Cause: … Fix: …") and the same connective tics ("The through-line…", "Now…", "Finally…") reused theme after theme. Connect ideas in real sentences, and vary their length and openings — not a staccato of em-dashes, colons, and one-line fragment closers.
- Write all prose in the document language requested by the user — natural to a native speaker, not a translation of English phrasing.
- **Don't translate terms of art.** Keep established technical terms in the form practitioners actually use in the target language — which for software is usually the original English: "deep link" (not "lien profond"), "named pipe", "worker", "endpoint", "cache", "commit". A literal calque nobody says reads as *less* readable, not more. Translate the prose around the term, gloss the concept once, but leave the term itself idiomatic.
