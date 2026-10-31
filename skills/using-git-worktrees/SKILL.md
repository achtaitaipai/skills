---
name: using-git-worktrees
description: Use when starting feature work that needs isolation from the current workspace, or before executing an implementation plan - ensures an isolated, installed, test-verified workspace exists via native tools or a git worktree fallback
---

# Using Git Worktrees

## Overview

Ensure work happens in an isolated workspace that is installed and has a known test baseline. Prefer the platform's native worktree tools. Fall back to manual git worktrees only when no native tool fits.

**Core principle:** Detect existing isolation first. Then use native tools. Then fall back to git. Never fight the harness, and never change the user's current branch without asking.

### How to run the shell blocks

- **Run each block whole, in one shell call.** Shell state (variables, `cd`) does not survive between tool calls, so every block defines what it needs and runs in a subshell `( … )`.
- **Fill in the input lines at the top of a block** before running it; don't split it up.
- **Carry results forward as absolute paths.** Each block prints `KEY=value` lines. Use the printed paths literally in later commands (`cd "/abs/worktree" && npm ci`), never a variable from an earlier call.
- The blocks work in bash and zsh. Don't rename their `wt_*` variables: in zsh, `path` is tied to `PATH` and `status` is read-only, and `GIT_DIR` is read by git itself.

## Step 0: Detect Existing Isolation

**Before creating anything, run the detection block.**

```bash
(
  if [ "$(git rev-parse --is-inside-work-tree 2>/dev/null)" != "true" ]; then
    if [ "$(git rev-parse --is-bare-repository 2>/dev/null)" = "true" ]; then
      echo "STATE=bare-repo"
    else
      echo "STATE=no-work-tree"
    fi
    exit 0
  fi
  wt_gitdir=$(cd "$(git rev-parse --git-dir)" >/dev/null 2>&1 && pwd -P)
  wt_common=$(cd "$(git rev-parse --git-common-dir)" >/dev/null 2>&1 && pwd -P)
  wt_super=$(git rev-parse --show-superproject-working-tree 2>/dev/null)
  if [ "$wt_gitdir" != "$wt_common" ] && [ -z "$wt_super" ]; then
    echo "STATE=linked-worktree"
    # The first listed worktree, unless it is a bare repo or a submodule's git dir.
    wt_main=$(git worktree list --porcelain | sed -n '1s/^worktree //p')
    [ -e "$wt_main/.git" ] || wt_main=""
  else
    echo "STATE=main-checkout"
    wt_main=$(git rev-parse --show-toplevel)
  fi
  echo "TOP=$(git rev-parse --show-toplevel)"
  echo "MAIN=$wt_main"
  echo "BRANCH=$(git branch --show-current)"
  echo "HEAD=$(git rev-parse --short HEAD 2>/dev/null)"
  echo "SUBMODULE_OF=$wt_super"
  echo "CHANGES=$(git status --porcelain | wc -l | tr -d ' ')"
)
```

`TOP` is the workspace you are in. `MAIN` is the original checkout, even from a linked worktree. Keep `MAIN`: Step 2 needs it.

- **`STATE=no-work-tree`** (not a repo, or inside a `.git` directory) or **`STATE=bare-repo`**: stop. Tell the user what you found and ask how to proceed.
- **`HEAD=` is empty:** the repo has no commits yet, so there is nothing to branch from. Stop and ask.
- **`STATE=linked-worktree`:** you are already isolated. Do NOT create another worktree. Report, then go to Step 2:
  - `BRANCH` set: "Already in isolated workspace at `<TOP>` on branch `<BRANCH>`."
  - `BRANCH` empty: "Already in isolated workspace at `<TOP>` (detached HEAD, externally managed). A branch will be needed before committing."
- **`STATE=main-checkout`** (including inside a submodule, where `SUBMODULE_OF` is set): continue below. `BRANCH` empty means a detached HEAD; that is fine, the new worktree branches from `HEAD`.

### Uncommitted changes

If `CHANGES` is not `0`, tell the user: **a new worktree starts from a commit, so these changes will NOT be in it.** Ask whether to:

- (a) proceed anyway → continue to consent;
- (b) let them commit or stash first → wait, then run Step 0 again;
- (c) work in place → skip Step 1 and go straight to Step 2 in `TOP`.

### Ask for consent

If the user has not already declared a worktree preference in your instructions, ask:

> "Would you like me to set up an isolated worktree? It keeps your current branch untouched."

Honor any declared preference without asking. If the user declines, work in place and go to Step 2.

## Step 1: Create the Isolated Workspace

### Decide the branch and base first

Whatever mechanism you use, settle these before creating anything:

- **Name:** the user's or plan's name if given; otherwise `feature/<short-kebab-slug>` derived from the task.
- **Base:** the current `HEAD` unless the user or plan names another ref (e.g. `main`). Local commits that aren't pushed only exist on a local base.

### 1a. Native worktree tools (preferred)

**Look for one properly.** It may be a tool like `EnterWorktree` or a `/worktree` command. Tools can be deferred and missing from your visible list, so search your tool registry (e.g. a tool search for "worktree") before concluding there is none. Two things don't count: a `--worktree` launch flag can't be applied to a running session, and `WorktreeCreate` is a settings hook, not something you call.

**Check where the native tool branches from.** Read its description. For example, Claude Code's `EnterWorktree` with a `name` creates the worktree under `.claude/worktrees/` from the ref set by the `worktree.baseRef` setting: `fresh` (the default) means `origin/<default-branch>`, `head` means the local `HEAD`. With the default, unpushed commits and any other base you chose are silently left out.

- **The native base is the base you want** (e.g. a fresh feature from the remote default branch) → create it with the native tool and skip to Step 2.
- **It isn't, but the tool can enter an existing worktree** (`EnterWorktree` takes a `path`) → create the worktree with the 1b block using `wt_location=".claude/worktrees"`, then enter the printed `WORKTREE` path with the native tool. The harness then tracks the session, and the base is right. Its exit tool won't delete a worktree entered this way; see "When the work is done".
- **Neither** → tell the user the native tool would start from `<its base>` instead of `<your base>`, and ask which they want.

**After creating, report what actually exists**, not what you intended:

```bash
git -C "/abs/worktree" branch --show-current; git -C "/abs/worktree" log -1 --format='%h %s'
```

Native tools own placement, branching and cleanup. Running `git worktree add` behind a native tool's back without entering the result creates phantom state the harness can't see.

### 1b. Git worktree fallback

Use when no native tool fits (see 1a). Fill in the inputs, then run the block **whole, in one call**. It works from any subdirectory: paths are resolved from the repo top. It also makes sure the worktree directory is ignored, via `.git/info/exclude` (no commit, the user's branch is untouched), without duplicating the rule.

```bash
(
  # --- Inputs: edit these four lines, then run the whole block in ONE shell call ---
  wt_branch="feature/my-task"  # branch to create (or reuse)
  wt_base=""                   # empty = current HEAD (works on a detached HEAD too)
  wt_location=""               # empty = .worktrees/ or worktrees/ if present, else .worktrees/
  wt_reuse="no"                # "yes" only after the user agreed to reuse an existing branch
  # ----------------------------------------------------------------------------------

  wt_top=$(git rev-parse --show-toplevel) || exit 1
  cd "$wt_top" >/dev/null || exit 1

  if [ -z "$wt_location" ]; then
    if [ -d .worktrees ]; then wt_location=.worktrees
    elif [ -d worktrees ]; then wt_location=worktrees
    else wt_location=.worktrees
    fi
  fi
  case "$wt_location" in
    /*) wt_root="$wt_location" ;;
    *)  wt_root="$wt_top/$wt_location" ;;
  esac
  wt_root="${wt_root%/}"
  wt_path="$wt_root/$wt_branch"

  [ -z "$wt_base" ] && wt_base=HEAD
  if ! git rev-parse --verify --quiet "$wt_base^{commit}" >/dev/null; then
    echo "RESULT=bad-base ($wt_base is not a commit; is the repo empty?)"; exit 1
  fi

  wt_exists=no
  if git show-ref --verify --quiet "refs/heads/$wt_branch"; then
    wt_exists=yes
    wt_where=$(git worktree list --porcelain | awk -v b="branch refs/heads/$wt_branch" \
      '/^worktree /{p=substr($0,10)} $0==b{print p}')
    if [ -n "$wt_where" ]; then
      echo "RESULT=branch-checked-out-elsewhere ($wt_where)"; exit 1
    fi
    if [ "$wt_reuse" != "yes" ]; then
      echo "RESULT=branch-exists (ask: reuse it, or pick another name)"; exit 1
    fi
  fi

  # Ignore check, only when the worktree directory is inside the repo.
  # The probe path works even before the directory exists.
  case "$wt_root/" in
    "$wt_top"/*)
      wt_rel="${wt_root#"$wt_top"/}"
      if ! git check-ignore -q "$wt_rel/__probe__"; then
        wt_exclude="$(git rev-parse --git-common-dir)/info/exclude"
        mkdir -p "$(dirname "$wt_exclude")"
        grep -qxF "/$wt_rel/" "$wt_exclude" 2>/dev/null || echo "/$wt_rel/" >> "$wt_exclude"
        echo "IGNORE_RULE_ADDED=/$wt_rel/ (in $wt_exclude)"
      fi
      ;;
  esac

  if [ "$wt_exists" = "yes" ]; then
    git worktree add "$wt_path" "$wt_branch" || { echo "RESULT=create-failed"; exit 1; }
    wt_base="(existing branch)"
  else
    git worktree add -b "$wt_branch" "$wt_path" "$wt_base" || { echo "RESULT=create-failed"; exit 1; }
  fi

  wt_abs=$(cd "$wt_path" >/dev/null 2>&1 && pwd -P)
  echo "RESULT=created"
  echo "WORKTREE=$wt_abs"
  echo "BRANCH=$(git -C "$wt_abs" branch --show-current)"
  echo "BASE=$wt_base"
  echo "HEAD=$(git -C "$wt_abs" log -1 --format='%h %s')"
)
```

**Location priority** (sets `wt_location`): a directory declared in your instructions, then an existing `.worktrees/` (preferred) or `worktrees/`, then the default `.worktrees/`. An absolute path outside the repo needs no ignore rule. If the user wants a shared, versioned rule instead of `info/exclude`, add it to `.gitignore` only with their agreement.

**Why the ignore rule matters:** an unignored worktree directory shows up as untracked, and `git add -A` records it as an _embedded repository_: a gitlink pointing at the worktree's commit, not its files. That commit then references something clones can't fetch.

**Act on `RESULT`:**

| `RESULT`                                | Meaning                                                                                          | Do                                                                         |
| --------------------------------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| `created`                               | Worktree ready                                                                                   | Note `WORKTREE`, go to Step 2                                              |
| `branch-exists`                         | The branch exists and isn't checked out                                                          | Ask: reuse it (rerun with `wt_reuse="yes"`) or pick another name           |
| `branch-checked-out-elsewhere (<path>)` | Git allows a branch in only one worktree                                                         | Offer to work in `<path>`, or pick another name                            |
| `bad-base`                              | The base isn't a commit (typo, or empty repo)                                                    | Ask for a valid base                                                       |
| `create-failed`                         | Git refused (permission/sandbox denial, path already exists…); the reason is in the output above | Explain, and **ask** before working in place: the user asked for isolation |

## Step 2: Project Setup

Run every command from here on in the worktree, by absolute path (`cd "/abs/worktree" && …` in each call), unless a native tool already switched your session into it.

### Submodules

If the worktree has a `.gitmodules` file, its submodule directories start out **empty**. Initialize them before installing or testing, or their absence will look like baseline failures:

```bash
cd "/abs/worktree" && git submodule update --init --recursive
```

This may fetch from the network.

### Local, untracked config

A worktree contains only tracked files, so `.env` and similar files are missing. List the candidates in the original checkout (`MAIN` from Step 0). The block finds them at any depth, whether ignored or not, and skips files inside other worktrees and dependency directories. Skip this section if you are working in place in `MAIN`. If `MAIN` is empty (a linked worktree of a bare repo or of a submodule), ask the user where the config lives.

```bash
(
  wt_src="/absolute/path/to/original/checkout"   # the MAIN printed in Step 0
  cd "$wt_src" >/dev/null || exit 1
  # Untracked config files, ignored or not, at any depth,
  # minus anything inside a registered worktree (other worktrees' secrets).
  WT_HERE=$(pwd -P); export WT_HERE
  WT_OTHERS=$(git worktree list --porcelain | sed -n 's/^worktree //p'); export WT_OTHERS
  git ls-files --others -- ':(glob)**/.env*' ':(glob)**/*.local' ':(glob)**/*.local.*' \
                           ':(exclude,glob)**/node_modules/**' \
                           ':(exclude,glob)**/.venv/**' ':(exclude,glob)**/venv/**' |
  awk 'BEGIN {
         n = split(ENVIRON["WT_OTHERS"], w, "\n")
         for (i = 1; i <= n; i++)
           if (w[i] != ENVIRON["WT_HERE"] && index(w[i] "/", ENVIRON["WT_HERE"] "/") == 1)
             ex[++m] = substr(w[i], length(ENVIRON["WT_HERE"]) + 2) "/"
       }
       { for (i = 1; i <= m; i++) if (index($0, ex[i]) == 1) next; print }'
)
```

Show the list and **ask before copying anything**: these files often hold secrets. Copy only what the user approves, keeping each relative path.

### Install dependencies

First look for project-specific instructions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, `Makefile`/`justfile` setup targets) and follow them. Otherwise detect by lockfile, which names the package manager. Python always gets a virtualenv **inside the worktree**, so installs never touch a shared or system environment (Homebrew's Python refuses them anyway with `externally-managed-environment`).

| Found                    | Run in the worktree                                                    |
| ------------------------ | ---------------------------------------------------------------------- |
| `pnpm-lock.yaml`         | `pnpm install --frozen-lockfile`                                       |
| `yarn.lock`              | `yarn install --frozen-lockfile` (Yarn 2+: `yarn install --immutable`) |
| `bun.lockb` / `bun.lock` | `bun install --frozen-lockfile`                                        |
| `package-lock.json`      | `npm ci`                                                               |
| `package.json` only      | `npm install` (creates a lockfile; see below)                          |
| `uv.lock`                | `uv sync` (creates `.venv` in the worktree)                            |
| `poetry.lock`            | `poetry install`                                                       |
| `requirements.txt`       | `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`   |
| `pyproject.toml` only    | `python3 -m venv .venv && .venv/bin/pip install -e .`                  |
| `Cargo.toml`             | `cargo build`                                                          |
| `go.mod`                 | `go mod download`                                                      |

A repo may need several of these (monorepos). If a tool is missing, report it rather than substituting another package manager. Run Python tools through the venv (`.venv/bin/python -m pytest`).

**Then check what setup changed:**

```bash
cd "/abs/worktree" && git status --porcelain
```

A modified lockfile, a new lockfile or an unignored `.venv/` is a side effect of setup, not part of the feature. Report it, and don't commit it unless the user asks.

## Step 3: Verify a Clean Baseline

Use the project's own test command (from the docs or task runner found above). Otherwise: `npm test`, `cargo test`, `.venv/bin/python -m pytest`, `go test ./...`.

- **Long suites:** if the full suite is slow, run it in the background or run the fast subset the project defines, and say which you ran.
- **Tests fail:** report the failures (names and a short reason) and ask whether to proceed or investigate. Those become the known baseline failures.
- **No tests found:** say so explicitly; don't claim a clean baseline.

### Report

```
Worktree ready at <WORKTREE>
Branch <BRANCH> from <BASE> (<HEAD>)
Setup: <commands run>; side effects: <none | files>
Baseline: <N> passed, <M> failed (<names if any>) | no tests found
Ready to implement <feature-name>
```

## When the work is done

Don't leave worktrees behind. When the feature is finished or abandoned, ask the user whether to merge, open a PR, keep the branch, or remove the worktree.

If a native tool created the worktree in this session, use its exit or cleanup action. Otherwise, including for a native worktree from an earlier session, run this from outside the worktree:

```bash
git -C "/abs/original/checkout" worktree remove "/abs/worktree"
```

Git refuses in two common cases:

- **`contains modified or untracked files`:** a copied `.env` or an unignored `.venv/` is enough.
- **`working trees containing submodules cannot be moved or removed`:** any worktree whose submodules were initialized in Step 2, even a clean one.

**Don't reach for `--force` on your own: it deletes uncommitted work.** First show what would be lost, including changes inside submodules:

```bash
cd "/abs/worktree" && git status --porcelain && git submodule foreach --quiet --recursive 'git status --porcelain'
```

Show the output, and add `--force` only once the user agrees.

Removing a worktree keeps its branch. Delete the branch only if the user asks, with `git branch -d <branch>`, which refuses unmerged work.

## Quick Reference

| Situation                                 | Action                                          |
| ----------------------------------------- | ----------------------------------------------- |
| Not a repo / bare repo                    | Stop and ask                                    |
| No commits yet                            | Stop and ask                                    |
| Already in a linked worktree              | Skip creation                                   |
| In a submodule                            | Treat as a normal checkout                      |
| Detached HEAD in main checkout            | Fine; base is `HEAD`                            |
| Uncommitted changes                       | Warn they won't carry over; ask                 |
| Native tool, right base                   | Use it                                          |
| Native tool, wrong base, can enter a path | Git block in `.claude/worktrees`, then enter it |
| No native tool                            | Git block                                       |
| Branch exists                             | Ask: reuse or rename                            |
| Branch checked out elsewhere              | Offer that worktree, or rename                  |
| Creation fails                            | Explain, ask before working in place            |
| `.gitmodules` present                     | `git submodule update --init --recursive`       |
| Untracked config in original              | List, ask before copying                        |
| Python project                            | venv inside the worktree                        |
| Setup modified files                      | Report, don't commit                            |
| Baseline tests fail                       | Report and ask                                  |
| No tests                                  | Say so, don't claim clean                       |
| `git worktree remove` refused             | Show what would be lost, ask before `--force`   |

## Common Rationalizations

| Excuse                                         | Reality                                                                            |
| ---------------------------------------------- | ---------------------------------------------------------------------------------- |
| "I'm obviously not in a worktree"              | Run Step 0. Harness-created isolation and submodules both fool eyeballing.         |
| "I set that variable in the previous call"     | Shell state doesn't persist between calls. Use the printed absolute paths.         |
| "I'll split the block up to read it better"    | Each block is self-contained on purpose. Run it whole.                             |
| "`path` is a fine variable name"               | In zsh it overwrites `PATH` and git disappears. Keep the `wt_*` names.             |
| "I don't see a worktree tool, so there's none" | Tools can be deferred. Search before falling back.                                 |
| "The native tool knows the right base"         | Check. It may branch from the remote default and drop local commits.               |
| "My uncommitted changes will be there"         | A worktree starts from a commit. Uncommitted work stays behind.                    |
| "I'll just commit the .gitignore change"       | That modifies the user's branch without asking. The block uses `info/exclude`.     |
| "`pip install` is fine here"                   | Without a venv in the worktree it rewires a shared environment, or fails outright. |
| "The submodule failures are pre-existing"      | Uninitialized submodules are empty. Init them before calling it a baseline.        |
| "Creation failed, I'll just work in place"     | The user asked for isolation. Ask first.                                           |
| "Baseline tests can wait"                      | A dirty baseline makes every later failure ambiguous.                              |
| "`--force` will clear the removal error"       | It also deletes uncommitted work. Show what would be lost and ask.                 |
