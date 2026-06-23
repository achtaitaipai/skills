---
name: hello-world
description: Example skill. Replace this with your own. Use when the user wants a demo of the skills repo.
---

# Hello World

This is an example skill shipped with the repo so `skills add hello-world`
works out of the box. Delete it once you've added your own.

## What a skill is

A skill is a directory under `skills/` containing a `SKILL.md` with YAML
frontmatter (`name`, `description`) and a markdown body of instructions.
Claude Code loads it from `.claude/skills/<name>/SKILL.md`.

You can bundle extra files alongside `SKILL.md` (scripts, templates,
references) — the whole directory is copied on install.
