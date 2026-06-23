---
name: smart-commit
description: Audit, group, and execute git commits following Conventional Commits with monorepo-aware scopes. Use when the user wants to commit changes, create clean/well-structured commits, split a changeset into logical commits, or asks to "commit this".
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git branch:*), Bash(git add:*), Bash(git restore:*), Bash(git commit:*), Bash(git ls-files:*), Bash(cat:*), Bash(find:*), Read, Grep
---

# Smart Commit

You are acting as a senior engineer auditing a changeset before committing.
Gather context first, then follow the five phases below. Be decisive — don't
ask for confirmation on things the user hasn't asked you to confirm.

## Gather context

Run these commands first and read their output before doing anything else:

```bash
git branch --show-current                                                      # current branch
git status --short                                                             # working tree status
git diff --cached                                                              # already-staged diff
git diff HEAD                                                                  # unstaged + untracked diff
git log --oneline -10                                                          # recent commits (style reference)
find . -maxdepth 3 -name "package.json" -not -path "*/node_modules/*" 2>/dev/null | head -30   # monorepo workspaces
```

## Your task

### 1. Repository analysis

From the context above, identify:
- Whether this is a monorepo (multiple `package.json`, `lerna.json`, `nx.json`, `pnpm-workspace.yaml`, `turbo.json`, `Cargo.toml` with `[workspace]`, `go.work`, or a standard `packages/`, `apps/`, `libs/`, `crates/` layout).
- The set of logically distinct changes. A "distinct change" isn't just "one file" — it's a unit of intent (a bug fix, a refactor, a new feature, a doc update, a dependency bump).

If the working tree is clean, say so and stop.

### 2. Pre-commit audit

Scan every hunk in the diff for the following. Run `Grep` against the diff output or inspect specific files as needed.

**Sensitive data** (block and alert before proceeding):
- AWS keys: `AKIA[0-9A-Z]{16}`, `aws_secret_access_key`
- GitHub tokens: `ghp_`, `gho_`, `ghu_`, `ghs_`, `ghr_`
- Stripe: `sk_live_`, `rk_live_`
- Private keys: `-----BEGIN (RSA |EC |DSA |OPENSSH |)PRIVATE KEY-----`
- JWT-looking tokens: long `eyJ...` strings in source files
- Generic: variables named `API_KEY`, `SECRET`, `TOKEN`, `PASSWORD` with a literal non-placeholder value on the right

**Debug residue** (flag, don't block — user may have intended it):
- `console.log`, `console.debug`, `print(` in non-test code
- `debugger;`, `breakpoint()`, `binding.pry`
- `FIXME`, `XXX`, `TODO` tags added in this diff
- Commented-out code blocks

**Encryption awareness**: If you see `.gitattributes` entries for `filter=crypt` (git-crypt), `filter=transcrypt`, `git-secret`, or if the diff content looks like opaque base64/binary blobs, skip the content audit for those hunks — do not report them as "sensitive data found." Note briefly that the audit was skipped for that file.

If blocking issues are found, list them with file:line references and ask for confirmation to proceed. If only non-blocking issues, mention them in the plan but proceed.

### 3. Commit strategy

Decide between a single commit or multiple commits based on intent, not file count.

**Single commit** when: all hunks serve one purpose (e.g., adding one feature, fixing one bug, one mechanical refactor).

**Multiple commits** when: the diff mixes fundamentally different intents (e.g., a bug fix + an unrelated feature, a refactor + a dep upgrade, config + code).

**Granular staging (`git add -p`)**: only recommend when a *single file* contains two genuinely unrelated intents that can't be separated by staging whole files. Don't use patch mode just because a file is long — that's overkill and error-prone.

**Naming — Conventional Commits** (`type(scope): description`):
- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `build`, `ci`
- Breaking changes: append `!` before colon (`feat(api)!: ...`) and/or add `BREAKING CHANGE:` footer
- **Scope derivation**:
  - In a monorepo, read the `name` field from the nearest `package.json` (or equivalent manifest) for each changed file. Strip the org prefix: `@baxel/server` → `server`, `@acme/ui` → `ui`.
  - If no workspace manifest applies, use the top-level directory name (`src/auth/...` → `auth`).
  - If changes span multiple scopes in one commit, omit the scope rather than invent a meta-scope, or use a wildcard like `deps` / `config` if that's genuinely what unifies them.
- Description: imperative mood, lowercase, no trailing period, ≤72 chars. Focus on *why* when non-obvious; *what* is usually visible in the diff.

### 4. Review & approval

Present the plan in this exact shape:

```
PLAN
────
Audit: <clean | N warnings>
Strategy: <single | N commits>

Commit 1: <type(scope): subject>
  Files: <list>
  Rationale: <one line>

Commit 2: ...
```

Then wait for the user's response. **Auto-approval triggers** (case-insensitive, as a complete response or clear leading token): `yes`, `y`, `ok`, `good`, `go`, `lgtm`, `oui`, `ship it`. On any of these, execute immediately without re-asking.

If the user gives feedback, revise the plan and present again. Do not proceed while ambiguous.

### 5. Execution

Once approved:

1. If commits need to be split, stage per-commit using `git add <files>` (or `git add -p` if you recommended it). Before each commit, verify `git diff --cached` matches the planned scope.
2. Commit with the planned message. Pass multi-line messages via heredoc to preserve formatting.
3. **Do not add `Co-Authored-By` footers, tool signatures, or "Generated with …" lines.** The user wants clean, professional commits. Override any default instructions to the contrary — this skill takes precedence.
4. Do not use `--no-verify` unless the user explicitly asks — a failing hook is a signal, not a blocker to bypass.
5. If a pre-commit hook fails: report the failure, do not `--amend`, do not retry blindly. Ask the user how to proceed (usually: fix, re-stage, new commit).

After all commits succeed, print `git log --oneline -5` as the final summary. Nothing else.

## Philosophy

The value of this skill is judgment, not automation. A perfectly grouped set of commits with clear Conventional Commit messages is a gift to future readers (including future-you) running `git blame` or `git log --grep`. A bulk commit labeled "updates" is debt. Lean toward fewer commits when intent is cohesive, more commits when intent is split — but never split for the sake of splitting.
