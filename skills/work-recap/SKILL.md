---
name: work-recap
description: Generate a polished, self-contained HTML recap of git history over a time period, in any language. Use whenever the user asks to summarize or report what was shipped or changed over a window (last week, sprint review, "what changed since X", standup notes from git) — even without the word "recap".
allowed-tools: Read, Write, Glob, Grep, Bash(git log:*), Bash(git diff:*), Bash(git shortlog:*), Bash(git rev-parse:*), Bash(open:*), Bash(xdg-open:*), Bash(start:*)
---

# Git Recap

Build a narrative recap of work done in a git repository over a time period, using the commit history as source material, and render it as a self-contained HTML document following a user-supplied template.

The commit history is the raw material. The template is the editorial brief. Your job is to read the history with judgment, decide what actually mattered, and tell the story in the shape the template prescribes — in the language the user asked for.

## Workflow at a glance

1. Resolve parameters (period, language, author filter, template path, output path).
2. Collect git data with the prescribed commands. Handle failures gracefully.
3. Read the template as an editorial brief.
4. Ground the vocabulary in the project's own docs.
5. Generate the recap, exercising judgment over what's relevant.
6. Write the HTML file and open it.

---

## 1. Resolve parameters

Accept invocation either as a slash command (`/work-recap`) with flags, or as natural language ("recap the last sprint in French"). Map natural language to the same parameters.

**Period** — the time window. Accept both:

- Relative names: `last week`, `this week`, `last month`, `this month`, `last sprint` (≈ last 14 days unless the user defines a sprint length), `last N days`, `today`, `yesterday`.
- Absolute dates: `--from 2026-06-01 --to 2026-06-23`.

Resolve relative names to concrete `--after` / `--before` dates against today's date before running git. If only `--from` is given, `--to` defaults to now. If the user gives neither, default to the **last 7 days** and say so in the output.

**Language** — `--lang <code>` takes priority (e.g. `--lang fr`, `--lang en`, `--lang es`). If absent, infer the language from the user's message / conversation. If still ambiguous, default to English. The template stays language-agnostic; only the generated prose changes language. Translate the _content_, not the template's structural HTML/labels unless the template's labels are themselves content.

**Author filter** — all authors by default. `--author <name-or-email>` restricts to one contributor (the "my recap for my 1-on-1" case). Pass it through to `git log --author=`.

**Template** — defaults to `skills/work-recap/HTML-REPORT.md` (resolve relative to this skill's directory). Override with `--template <path>`. If the template is missing, see the error handling section.

**Output** — defaults to the OS temp directory, filename `<repo>-<from>_<to>.html` (e.g. `myrepo-2026-06-01_2026-06-23.html`). Override the directory with `--output <path>`. After writing, open it in the default browser.

---

## 2. Collect git data

Run from the current repository only (no submodules, no multi-repo). First confirm you're in a git repo: `git rev-parse --is-inside-work-tree`. If that fails, see error handling.

Run these commands. If any single command fails, keep going with what succeeded and note the gap in the output (partial recap is better than no recap).

**Full commit log with bodies** — subjects alone are too thin; the commit body is where the _why_, the decisions, and ticket references live:

```bash
git log --no-merges --after="<from>" --before="<to>" [--author="<author>"] \
  --date=short \
  --format="%H%n%an%n%ad%n%s%n%b%n==END=="
```

The `==END==` sentinel separates commits cleanly. `--no-merges` drops automatic merge commits, which carry no narrative content — but every other commit stays in, including `wip:`, `fixup!`, and `chore:` ones. Do not filter by prefix: a scrappy `wip: fought the auth refresh for two days` tells you something real about the effort. Judge relevance by _content_, not by label.

**Changed files per area** — to understand scope without exploding context:

```bash
git diff --stat <from-rev>..<to-rev>
```

If you can't resolve revisions cleanly from dates, fall back to per-commit `--stat` on the log, or skip stats and note it.

**Per-author rollup** — useful for team recaps:

```bash
git shortlog -sne --no-merges --after="<from>" --before="<to>"
```

**Merge commits, separately** — if you want to surface integration work distinct from feature work, list them on their own:

```bash
git log --merges --after="<from>" --before="<to>" --format="%s"
```

These are listed separately, not mixed into the main narrative, since they describe integration rather than feature work.

---

## 3. Read the template as an editorial brief

The template at the resolved path is **not** a mechanical placeholder file. It is a brief: it describes the document's structure, its visual patterns, its tone, and its vocabulary. Read the whole thing into context and treat every instruction in it as binding — section order, diagram patterns, colour discipline, forbidden phrasings, glossary terms, all of it.

If the template prescribes a specific vocabulary or forbids certain words, obey it exactly. If it says "no introduction paragraph, straight into X", do that. If it specifies CDN libraries (Tailwind, Mermaid, etc.), use exactly those and nothing more. The output is a single self-contained HTML file unless the template says otherwise.

The template owns the **visual and structural** form. This skill owns the **narrative intent** below.

---

## 4. Ground the vocabulary in the project's own docs

Commit subjects are terse and inconsistent; the project's docs are where the real names live. Before writing, take a light pass over whatever the repo offers to learn how the project names its own concepts — and then use those names:

- The top-level `README` (and any per-package `README`), to learn what the project _is_ and its component names.
- A glossary / lexicon file if one exists (often `docs/lexique.md`, `GLOSSARY.md`, or similar), and architecture decision records (`docs/adr/`) — these are the canonical source of domain terms.
- Any doc that touches the themes you're about to write up, when a commit references a concept you can't name confidently from the log alone.

This is a light grounding pass, not a full read of `docs/` — use `Glob`/`Grep` to locate the glossary and the few docs relevant to the period, not to ingest everything. Adopt the project's vocabulary verbatim (entity names, feature names, the domain terms it has chosen) rather than inventing your own paraphrase. If a recent commit changed a name, prefer the new one (see Gotchas). When the repo has no docs, fall back to the names used in the commit bodies and code paths.

---

## 5. Generate the recap

### Narrative intent (what each part must achieve)

The template names the sections; you ensure they _work_. Regardless of how the template labels them, the recap must achieve these:

- **Orientation** — someone who did not follow the work should grasp the essentials of the period from the top of the document: what was worked on and what shipped. State it; don't size it up — "the week covered two areas: X and Y", not "two big pushes filled the week".
- **The substance** — the actual changes, grouped by theme or area rather than dumped as a flat chronological list. A reader should see the shape of the work, not re-read the git log. Within each theme, surface what's notable about it — the decision made, the hard problem wrestled with, the silent bug, the surprise — as part of that theme's story. This is where commit bodies and `wip` commits earn their place. Do **not** add a separate "notable/highlights" section that re-summarizes the work: orientation is already covered by the overview, and the substance lives here.
- **Loose ends** — if the history implies unfinished work or open questions, surface them.

These are intents, not mandatory headings. Let the template's structure carry them.

### Judgment over completeness

Do not transcribe the git log. Synthesize. Five commits fixing one feature are one accomplishment, not five bullet points. A typo-fix commit usually doesn't deserve a line; a commit body explaining a tricky architectural decision usually does. Group related commits, name the work in human terms, and let trivia fall away — while keeping the scrappy-but-meaningful commits that reveal real effort.

Write in the requested language naturally — not a translation of English phrasing, but prose a native speaker of that language would write. Translate the prose, not the jargon: established technical terms of art stay in the form practitioners actually use (see Gotchas).

### Write for two readers at once

The recap must serve both the engineer who wants the exact mechanism and a non-technical reader (a manager, a client — anyone outside the codebase) who needs to know what changed and why it matters. Lead each item with the plain-language impact — the "so what" a non-technical reader grasps without knowing the codebase — then give the precise mechanism for whoever wants it. Keep the technical detail; don't dumb it down. Gloss jargon on first use by pairing it with its consequence (e.g. "a _named pipe_ — a channel two processes use to talk to each other") rather than dropping the term.

### Voice

Sober and editorial — the register of a sharp engineer writing up the week, not a product changelog. Plain declarative sentences carry the facts. Reserve bold for almost nothing: highlighting a phrase in every paragraph stops signalling anything and reads as a template. Avoid formulaic scaffolding — label-led fragments and the same connective tics repeated theme after theme. Vary sentence length and openings instead of a staccato of em-dashes and colons. The template may add its own voice rules; they win.

### Neutral register — describe, don't appraise

Report what changed and why; never rate how big, hard, important, or dramatic it was. The reader forms their own judgment from the facts — your job is to give them accurate facts, not a verdict. Cut evaluative qualifiers (translate these into the document language; they're listed in English only to name the categories):

- **Size / importance** — "big", "huge", "major", "massive", "the headline", "flagship", "the big push". Convey magnitude with checkable facts instead: "a ~900-line rewrite across 40 files", "5 commits", not "a big push".
- **Praise / aesthetics** — "solid", "elegant", "clean", "impressive", "nice", "finally".
- **Drama / severity** — "catastrophic", "nightmare", "painful", "critical", "a struggle". A production incident is just "a production incident", described by its symptom and cause, not its emotional weight.

This applies to **headings and the overview too**: name the work, don't characterise it — a heading like "Production incident in the naming-rule matcher", not "The catastrophic incident". The overview opens on the facts of the period (what was worked on, what shipped), not on a framing sentence that sizes it up.

Note the boundary: judgment over *what to include and how to group it* (see Judgment over completeness) is still yours — that shapes the **content**, never the **tone**. And a fixed technical term of art is not a value judgment: if the precise name of a phenomenon is _catastrophic backtracking_, you may name it once as that term where it adds precision — but keep it as the bare technical label, don't bend it into editorial colour in a heading and don't let the qualifier leak into the surrounding prose.

---

## 6. Write and open

Write the HTML to the resolved output path. Then open it:

- macOS: `open <path>`
- Linux: `xdg-open <path>`
- Windows: `start <path>`

Tell the user where the file is and give a one-line summary of what the recap covers (period, number of commits synthesized, authors).

---

## Gotchas

Failure modes seen while iterating — check the draft against each:

- **Calqued jargon.** Translating a term of art literally — "lien profond" for _deep link_, "tube nommé" for _named pipe_ — reads as less natural to a practitioner, not more. Keep the term as people actually say it (usually the original English); translate only the prose around it.
- **Inventing names instead of using the project's.** If the repo's README/glossary/ADRs name a concept a certain way, use that name verbatim — don't paraphrase it into your own wording. This is what the grounding pass (step 4) is for.
- **Old names during a rename.** If the window includes a rename, describing the architecture in the _old_ names and then announcing the rename forces the reader to re-map everything. Use the final names throughout; state the rename itself as a one-line fact.
- **A "highlights / notable" section that re-summarizes.** Orientation is the overview's job, the substance is Work's. A third section re-listing the same items is pure duplication — fold any genuinely notable detail (a silent bug, a hard call) into its theme instead.
- **Overview creep.** Two short paragraphs at most. If it starts listing every theme, it's becoming a second recap — push the detail down into Work.
- **Transcribing the log.** Five commits on one feature are one entry, not five. Judge relevance by commit _content_, not prefix — a scrappy `wip:` body often reveals more than a tidy `feat:`.
- **Highlighter prose.** Bolding a phrase in every paragraph, or reusing the same scaffolding labels and connective tics theme after theme, reads templated and machine-written. Bold almost nothing; connect ideas in plain, varied sentences (see Voice).
- **Evaluative language.** Qualifiers that rate the work — "big push", "catastrophic", "major", "elegant", "finally" — are appraisals, not facts. Strip them from prose, headings, and the overview; state magnitude with numbers and severity with the symptom (see Neutral register). The one exception is a fixed technical term of art whose name happens to contain such a word.

## Error handling

Aim for a useful partial result over a hard failure. Specifically:

- **Not a git repository** — stop and tell the user plainly they need to run this inside a git repo.
- **Template missing** — stop and tell the user the expected path (`skills/work-recap/HTML-REPORT.md`) and that they can point elsewhere with `--template`. Don't invent a template silently.
- **No commits in the period** — don't produce an empty document. Tell the user no commits matched, echo back the resolved date range and any author filter, and suggest the likely causes (wrong dates, wrong branch, wrong author spelling).
- **Invalid period** — `from` after `to`, or a future start date: flag it and ask for a corrected range rather than guessing.
- **A git command fails mid-collection** — proceed with the data you have, generate the recap, and add a visible note in the output stating which data was unavailable (e.g. "file-level stats unavailable").
- **Detached HEAD / no branch** — proceed; commit history is still reachable. Note it only if it affects results.

## Example invocations

- `/work-recap last week` → English (inferred), all authors, default template, last 7 days.
- `/work-recap --from 2026-06-01 --to 2026-06-23 --lang fr` → French recap for an explicit range.
- "summarize what the team shipped last sprint, in Spanish" → Spanish, all authors, ~14 days.
- `/work-recap last month --author alice@corp.com --lang en` → Alice's personal monthly recap.
