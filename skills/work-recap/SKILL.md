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

Invocation is a slash command (`/work-recap`) with flags or natural language ("recap the last sprint in French") — map both to the same parameters.

- **Period** — relative (`last week`, `last sprint` ≈ 14 days, `last N days`) or absolute (`--from`/`--to`). Resolve to concrete `--after`/`--before` dates against today before running git. Default: **last 7 days** (say so in the output).
- **Language** — `--lang <code>`, else infer from the conversation, else English. Only the generated prose changes language; the template stays as-is.
- **Author** — all authors by default; `--author <name-or-email>` restricts to one, passed through to `git log --author=`.
- **Template** — defaults to `HTML-REPORT.md` in this skill's directory (the editorial brief, which names its skeleton). Override with `--template <path>`; if missing, see error handling.
- **Output** — defaults to the OS temp dir, filename `<repo>-<from>_<to>.html`. Override the dir with `--output <path>`. Open it in the browser after writing.

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

The `==END==` sentinel separates commits cleanly. `--no-merges` drops automatic merge commits, which carry no narrative content — but every other commit stays in, including `wip:`, `fixup!`, and `chore:` ones. Don't filter by prefix; judge relevance by _content_ when you synthesize (step 5).

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

If the brief prescribes a specific vocabulary or forbids certain words, obey it exactly. If it says "no introduction paragraph, straight into X", do that. When the brief and this skill could overlap on how to phrase or emphasise something, the brief wins.

---

## 4. Ground the vocabulary in the project's own docs

Commit subjects are terse and inconsistent; the project's docs are where the real names live. Before writing, take a light pass over whatever the repo offers to learn how the project names its own concepts — and then use those names:

- The top-level `README` (and any per-package `README`), to learn what the project _is_ and its component names.
- A glossary / lexicon file if one exists (often `docs/lexique.md`, `GLOSSARY.md`, or similar), and architecture decision records (`docs/adr/`) — these are the canonical source of domain terms.
- The rest of `docs/` — any doc that touches the themes you're about to write up, when a commit references a concept you can't name confidently from the log alone.
- As a last resort, the wording the app shows its users (UI labels, i18n / translation files) — the user-facing name for a feature is often the clearest one.

This is a light grounding pass, not a full read of `docs/` — use `Glob`/`Grep` to locate the glossary and the few docs relevant to the period, not to ingest everything. Adopt the project's vocabulary verbatim (entity names, feature names, the domain terms it has chosen) rather than inventing your own paraphrase. If a recent commit changed a name, prefer the new one (see Gotchas). When the repo has no docs at all, fall back to the names used in the commit bodies and code paths.

---

## 5. Generate the recap

Synthesize the history with judgment — don't transcribe the log. Five commits fixing one feature are one accomplishment, not five bullets; a typo-fix rarely earns a line, a commit body explaining a tricky decision usually does. Group related commits, name the work in human terms, keep the scrappy-but-meaningful `wip` commits that reveal real effort, and let trivia fall away. Judge relevance by commit _content_, not prefix. This judgment over *what to include and how to group it* is yours.

Everything about how the recap *reads* — section order, grouping, audience, tone, what to bold, jargon, the ban on evaluative language — is owned by the template's editorial brief. Read those rules out of the resolved template and obey them; this skill deliberately doesn't restate them, so a swapped-in template fully controls voice and structure. If the template is silent on a point, fall back to a sober editorial register that reports facts without appraising them and serves both a technical and a non-technical reader.

---

## 6. Write and open

Write the HTML to the resolved output path. Then open it:

- macOS: `open <path>`
- Linux: `xdg-open <path>`
- Windows: `start <path>`

Tell the user where the file is and give a one-line summary of what the recap covers (period, number of commits synthesized).

---

## Gotchas

- **Inventing names instead of using the project's.** If the repo's README/glossary/ADRs name a concept a certain way, use that name verbatim — don't paraphrase. This is what the grounding pass (step 4) is for.
- **Old names during a rename.** If the window includes a rename, use the final names throughout and state the rename itself as a one-line fact, rather than describing the architecture in the old names and forcing the reader to re-map.

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
