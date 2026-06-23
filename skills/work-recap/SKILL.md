---
name: work-recap
description: Generate a polished, self-contained HTML recap of git history over a time period, in any language. Use whenever the user asks to summarize or report what was shipped or changed over a window (last week, sprint review, "what changed since X", standup notes from git) — even without the word "recap".
allowed-tools: Read, Write, Glob, Grep, Bash(git log:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(open:*), Bash(xdg-open:*), Bash(start:*)
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

**Template** — defaults to `skills/work-recap/HTML-REPORT.md` (resolve relative to this skill's directory), the editorial brief, which names its visual skeleton (default `SKELETON.html` alongside it). Override the brief with `--template <path>`. If either file is missing, see the error handling section.

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

---

## 3. Read the template (brief + skeleton)

The template comes in two paired files; read both into context before writing.

- **The editorial brief** at the resolved template path is **not** a mechanical placeholder file. It describes the document's structure, audience, tone, and vocabulary, and it names its skeleton. Treat every instruction in it as binding — section order, colour discipline, forbidden phrasings, glossary terms, all of it.
- **The skeleton** it names (default `SKELETON.html` alongside the brief) is the visual form: the HTML markup, the inline CSS, the design tokens, and the class system, documented in HTML comments. Start the output from this file — fill the `{{placeholders}}`, reuse its classes and tokens, and **don't** restructure the layout or rewrite the CSS. The output is a single self-contained HTML file unless the brief says otherwise.

If the brief prescribes a specific vocabulary or forbids certain words, obey it exactly. If it says "no introduction paragraph, straight into X", do that.

**Division of labour.** The template (brief + skeleton) owns the **writing** — structure, section order, voice, register, what to bold, how to handle jargon and evaluative language — and the **visual** form. This skill owns the **process**: resolving parameters, collecting the history, grounding the vocabulary, and reading the log with judgment. When the two could overlap (how to phrase, what to emphasise, what tone), the template wins; the baseline narrative intent below applies only as a floor when the template is silent.

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
- **The substance** — the actual changes, grouped rather than dumped as a flat chronological list (the template prescribes how to group — e.g. by category, area, or theme). A reader should see the shape of the work, not re-read the git log. Within each item, surface what's notable about it — the decision made, the hard problem wrestled with, the silent bug, the surprise — as part of its story. This is where commit bodies and `wip` commits earn their place. Do **not** add a separate "notable/highlights" section that re-summarizes the work: orientation is already covered by the overview, and the substance lives here.
- **Loose ends** — if the history implies unfinished work or open questions, surface them.

These are intents, not mandatory headings. Let the template's structure carry them.

### Judgment over completeness

Do not transcribe the git log. Synthesize. Five commits fixing one feature are one accomplishment, not five bullet points. A typo-fix commit usually doesn't deserve a line; a commit body explaining a tricky architectural decision usually does. Group related commits, name the work in human terms, and let trivia fall away — while keeping the scrappy-but-meaningful commits that reveal real effort. This judgment over *what to include and how to group it* is yours; it shapes the content, not the tone.

### Voice, register, and audience — follow the template

The template is the editorial brief and owns how the recap reads: the tone and register, who it's written for, what to bold, how to gloss jargon, how to handle terms of art, and the ban on evaluative language. Read those rules out of the resolved template and obey them. This skill deliberately does **not** restate them, so a swapped-in template fully controls the voice. If the template is silent on some point, fall back to a sober, editorial register that reports facts without appraising them and serves both a technical and a non-technical reader.

---

## 6. Write and open

Write the HTML to the resolved output path. Then open it:

- macOS: `open <path>`
- Linux: `xdg-open <path>`
- Windows: `start <path>`

Tell the user where the file is and give a one-line summary of what the recap covers (period, number of commits synthesized).

---

## Gotchas

Failure modes seen while iterating — check the draft against each:

- **Inventing names instead of using the project's.** If the repo's README/glossary/ADRs name a concept a certain way, use that name verbatim — don't paraphrase it into your own wording. This is what the grounding pass (step 4) is for.
- **Old names during a rename.** If the window includes a rename, describing the architecture in the _old_ names and then announcing the rename forces the reader to re-map everything. Use the final names throughout; state the rename itself as a one-line fact.
- **Transcribing the log.** Five commits on one feature are one entry, not five. Judge relevance by commit _content_, not prefix — a scrappy `wip:` body often reveals more than a tidy `feat:`.
- **Voice drift.** Bold overuse, repeated scaffolding labels and connective tics, evaluative qualifiers ("big push", "catastrophic", "elegant"), calqued jargon, a redundant "highlights" section, an overview that balloons into a second recap — these are all governed by the template's editorial brief. Re-read it and check the draft against it rather than trusting recall.

## Error handling

Aim for a useful partial result over a hard failure. Specifically:

- **Not a git repository** — stop and tell the user plainly they need to run this inside a git repo.
- **Template missing** — stop and tell the user the expected path (`skills/work-recap/HTML-REPORT.md`, plus the `SKELETON.html` it names) and that they can point elsewhere with `--template`. Don't invent a template or skeleton silently.
- **No commits in the period** — don't produce an empty document. Tell the user no commits matched, echo back the resolved date range and any author filter, and suggest the likely causes (wrong dates, wrong branch, wrong author spelling).
- **Invalid period** — `from` after `to`, or a future start date: flag it and ask for a corrected range rather than guessing.
- **A git command fails mid-collection** — proceed with the data you have, generate the recap, and add a visible note in the output stating which data was unavailable (e.g. "file-level stats unavailable").
- **Detached HEAD / no branch** — proceed; commit history is still reachable. Note it only if it affects results.

## Example invocations

- `/work-recap last week` → English (inferred), all authors, default template, last 7 days.
- `/work-recap --from 2026-06-01 --to 2026-06-23 --lang fr` → French recap for an explicit range.
- "summarize what the team shipped last sprint, in Spanish" → Spanish, all authors, ~14 days.
- `/work-recap last month --author alice@corp.com --lang en` → Alice's personal monthly recap.
