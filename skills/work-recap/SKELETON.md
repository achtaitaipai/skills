<!--
  Work-recap skeleton — the MARKDOWN form of the recap.
  Paired with REPORT.md (the editorial brief, which owns the writing/voice).

  How to use this file:
    • Fill the {{placeholders}} and replace the example themes with real ones.
    • Keep the heading levels and the section order; don't restructure them. If a
      recap needs a shape this skeleton can't express, that's a signal to extend
      the skeleton deliberately, not to improvise.
    • The output is one self-contained .md file: no images, no diagrams, no
      HTML blocks, no front matter. Identifiers go in `backticks`.
    • Drop these comments from the output.
-->

# {{repo name}} — {{period, e.g. recap, 16–23 June 2026}}

`{{N}} commits · {{N}} files · +{{N}} / −{{N}} lines`

<!-- OVERVIEW — one or two short paragraphs. No throat-clearing before it. -->

## {{Overview heading}}

{{Orientation paragraph.}}

{{Optional second paragraph.}}

<!--
  WORK — the category sections below, in this order; omit any that's empty.
  Markdown has no colour: a theme's category is carried by the section it sits
  under, and by nothing else. Don't repeat the category in the theme title,
  don't prefix themes with emoji or badges as a stand-in for an accent colour.
  Within a section, order themes biggest-first (most files / lines touched).
-->

## {{Features heading}}

### {{Feature title in human terms}}

{{Impact first — the "so what" — then the mechanism.}}

{{Optional further detail; weave the notable bit into the prose.}}

## {{Improvements heading}}

### {{Improvement title}}

{{…}}

## {{Fixes heading}}

### {{Fix title}}

{{What broke and what now holds.}}

<!-- MISC — optional digest: one bullet list, no `###` headings. Omit if empty. -->

## {{Misc heading}}

- {{One-line item.}}
- {{One-line item.}}

<!-- LOOSE ENDS — omit the whole section if the history implies none. -->

## {{Loose ends heading}}

### {{Open item}}

{{What's unresolved and why.}}

<!--
  Nothing follows the loose ends. Don't add a table of contents, a closing
  summary, a commit list, or a signature block.
-->
