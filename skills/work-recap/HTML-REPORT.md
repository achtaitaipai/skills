# Work Recap — HTML Report Format

The recap is rendered as a single self-contained HTML file. Tailwind comes from a CDN; everything else is hand-built divs — that's where the editorial voice lives. No diagram libraries: a recap is a narrative, not architecture docs. Boxes-and-arrows look generic and rarely earn their space here; convey structure in prose plus a compact "touched areas" line.

> This is a starter template. Replace it with your own to control structure, tone, and vocabulary. The skill reads whatever lives here as the editorial brief.

## Audience — precise *and* readable by a non-technical reader

Two readers at once: the engineer who wants the exact mechanism, and a non-technical reader (a manager, a client — anyone outside the codebase) who needs to know **what changed and why it matters**. Serve both, in this order:

- Lead every theme with a plain-language sentence: the effect, the user-visible or business consequence, the "so what". A non-technical reader should understand it without knowing the codebase.
- Then give the precise mechanism for whoever wants it. Don't dumb it down — the technical specifics stay.
- **Gloss jargon, don't strip it.** Pair a term with its consequence on first use: "_catastrophic backtracking_ (une expression régulière qui saturait le CPU)", "tickets à usage unique (un jeton qu'on ne peut utiliser qu'une fois)". Identifiers (`ClassificationSuggestionEngine`, `myged://`) stay in `font-mono` but should never be the only thing a sentence conveys.

## Scaffold

```html
<!doctype html>
<html lang="{{document language}}">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Recap — {{repo name}} — {{period}}</title>
    <script src="https://cdn.tailwindcss.com"></script>
  </head>
  <body class="bg-stone-50 text-slate-900 font-sans antialiased">
    <main class="max-w-4xl mx-auto px-6 py-12 space-y-14">
      <header>...</header>
      <section id="overview">...</section>
      <section id="work">...</section>
      <section id="loose-ends">...</section>
    </main>
  </body>
</html>
```

## Header

Repo name, the period covered (resolved dates), and a one-line stat strip (commits, files, ± lines, contributors). No throat-clearing — straight into the overview. If there's a single contributor, name them here once and **don't** repeat it in a footer note.

## Overview

A short paragraph — two at most. Someone who didn't follow the work reads only this and knows the focus of the period and what shipped. Lead with the headline. Resist letting it grow into a full recap of its own; the substance belongs in Work.

## Work

The substance, grouped by theme or area — not a flat list of commits. Each theme:

- A short title naming the work in human terms.
- One or two sentences of what changed and why (mine the commit bodies for the why) — impact first, mechanism second.
- The touched areas as a compact `font-mono text-sm` line where useful.

Synthesize. Five commits on one feature are one entry, not five.

What's notable about a theme — the decision made, the hard problem wrestled with, the silent bug, the surprise — belongs **inside that theme**, woven into its prose, not in a separate "notable" section. Commit bodies and scrappy `wip` commits earn their place here. State the consequence before the cause ("ça figeait un worker entier" → puis le mécanisme). Don't add a section that re-summarizes the work; the orientation is already in the overview and the substance is here.

## Loose ends

If the history implies unfinished work or open questions, list them. Omit the section if there are none.

## Style guidance

- Lean editorial, not corporate-dashboard. Generous whitespace. `font-serif` headings work well with stone/slate.
- **Colour as hierarchy, not decoration.** One accent family, used to rank the work:
  - **Indigo** — flagship/headline work (the one or two big chantiers of the period).
  - **Emerald** — supporting work (fixes, finishing touches, infra).
  - **Amber** — warnings and loose ends only.
  - Don't introduce a fourth accent; let stone/slate carry everything else.
- **Sober editorial voice** — the register of a sharp engineer writing up the week, not a product changelog. Plain declarative sentences; let the facts carry the weight. No hedging ("it's worth noting that…"), no marketing gloss. If a sentence could be a bullet, make it a bullet; if a bullet could be cut, cut it.
- **Bold almost never** in the body prose. Highlighting a phrase in every paragraph stops meaning anything and reads as a template; the theme title and the `font-mono` touched-areas line carry the emphasis already.
- **No formulaic scaffolding.** Avoid label-led fragments ("Symptôme : … Cause : … Correctif : …") and the same connective tics ("Le fil rouge…", "Désormais", "Enfin") reused theme after theme. Connect ideas in real sentences, and vary their length and openings — not a staccato of em-dashes, colons, and one-line fragment closers.
- Write all prose in the document language requested by the user — natural to a native speaker, not a translation of English phrasing.
- **Don't translate terms of art.** Keep established technical terms in the form practitioners actually use in the target language — which for software is usually the original English: "deep link" (not "lien profond"), "named pipe", "worker", "endpoint", "cache", "commit". A literal calque nobody says reads as *less* readable, not more. Translate the prose around the term, gloss the concept once, but leave the term itself idiomatic.
- The document is fully static — Tailwind CDN only, no scripts, no diagram libraries.
