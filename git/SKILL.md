---
name: git
description: >
  Use this skill for any git-related task: committing changes, pulling or pushing to a remote,
  branching, merging, rebasing, resolving conflicts, viewing history/diffs, stashing, tagging,
  resetting, or following a git flow branching strategy (feature, release, hotfix branches).
  Trigger whenever the user mentions git commands, GitHub/GitLab workflows, "commit", "push",
  "pull", "branch", "merge", "rebase", "stash", "tag", "cherry-pick", "conflict", "git flow",
  "feature branch", "release branch", or "hotfix". Also trigger when the user asks how to undo
  changes, compare versions, or manage a project's version history — even if they don't say "git"
  explicitly.
---

# Git Skill

Comprehensive guidance for common git operations, safe practices, and git flow branching strategy.

---

## Core Principles

- **Always check status first**: `git status` before committing, `git log --oneline -5` to orient.
- **Prefer explicit over implicit**: show full commands, not abbreviations.
- **Safety net**: flag destructive operations (force push, hard reset, rebase on shared branches) with a warning.
- **Context matters**: ask for the remote name (usually `origin`) and branch name if not obvious from context.
- **Tags are the version source of truth**: version numbers are always derived from semver git tags (matching `vMAJOR.MINOR.PATCH`). Never hard-code versions in branch names, commit messages, or files without first reading the current version from tags. Use `git describe` or `git tag` to determine the current version before incrementing.

---

## 1. Stage & Commit

```bash
# See what changed
git status
git diff                    # unstaged changes
git diff --staged           # staged changes

# Stage files
git add <file>              # specific file
git add .                   # everything in current directory
git add -p                  # interactive hunk-by-hunk staging (recommended)

# Commit
git commit -m "short description of what changed"

# Amend last commit (before pushing)
git commit --amend --no-edit          # keep message, just add staged changes
git commit --amend -m "new message"   # change message
```

**Message length rules:**
- **Simple change** (single area, obvious scope — e.g. button style tweak, typo fix): subject line only.
- **Complex change** (multiple areas, non-obvious motivation, or behaviour change): subject + blank line + body explaining *why*.

**Hard rules:**
- **MUST NOT** include Conventional Commit prefixes, e.g. `fix:`, `feat:`, `docs:`, etc. 
- **MUST NOT** reference Claude, Anthropic, Co-Authored-By AI footers, or any AI tool in commit messages.
- **MUST NOT** stage or commit generated files, build artifacts, or `node_modules`. Check `.gitignore` covers them before committing.

---

## 2. Tags

**Tags matching `vMAJOR.MINOR.PATCH` (e.g. `v1.2.3`) are the single source of truth for version numbers.** Always read the current version from tags before creating a release or hotfix — never assume or hard-code it.

### Reading the current version

```bash
# Most recent semver tag reachable from HEAD (includes commit distance + hash if not on a tag)
git describe --tags --match "v*"

# Latest semver tag in the repo (regardless of HEAD position)
git tag --list "v*" --sort=-version:refname | head -1

# Confirm HEAD is exactly on a tag (exit code 0 = yes, non-zero = dirty/ahead)
git describe --tags --exact-match --match "v*" 2>/dev/null
```

### Creating and managing tags

```bash
git tag v1.0.0                       # lightweight tag (preferred — merge commit carries the message)
git push origin v1.0.0               # push single tag
git push origin --tags               # push all tags
git tag -d v1.0.0                    # delete local tag
git push origin --delete v1.0.0      # delete remote tag
```

> **Note:** `git describe` requires the `--tags` flag to find lightweight tags — the version-reading commands above already include it.

---

## 3. Git Flow Branching Strategy

Use the `git-flow` CLI (`git-flow-next` installed at `/usr/local/bin/git-flow`) for all git flow operations.

### Initialization

```bash
git flow init            # interactive setup
git flow init -d --tag v # accept defaults (master/develop, feature/release/hotfix/support prefixes, v tag prefix)
```

If a command fails because git flow is not initialized, run `git flow init` first, then re-run the original command.

### Branch Structure
| Branch | Purpose | Merges into |
|--------|---------|-------------|
| `master` | Production code | — |
| `develop` | Integration branch | `master` (via release) |
| `feature/*` | New features | `develop` |
| `release/*` | Release prep & bugfixes | `master` + `develop` |
| `hotfix/*` | Emergency prod fixes | `master` + `develop` |

### Common Commands

```bash
# Features
git flow feature start <name>        # branch off develop
git flow feature finish <name>       # merge back to develop, delete branch
git flow feature publish <name>      # push to remote
git flow feature update              # pull changes from develop into current feature

# Releases (version = semver without "v", e.g. 1.2.0)
git flow release start <version>     # branch off develop
git flow release finish <version>    # merge to master + develop, tag v<version>
git flow release publish <version>   # push to remote

# Hotfixes
git flow hotfix start <name>         # branch off master
git flow hotfix finish <name>        # merge to master + develop, tag

# Convenience (on current branch)
git flow finish                      # finish whichever branch you're on
git flow publish                     # publish current branch to remote
git flow update                      # update current branch from its parent
git flow overview                    # show all git-flow branches at a glance
```

**Tags are the version source of truth.** Always read the current version before starting a release or hotfix:

```bash
git tag --list "v*" --sort=-version:refname | head -1
```

---

## 11. Common Troubleshooting

| Problem | Fix |
|--------|-----|
| Merge conflict | Edit conflicted files, `git add <file>`, then `git commit` |
| Accidentally committed to wrong branch | `git cherry-pick <hash>` onto correct branch, then `git reset --hard HEAD~1` on wrong branch |
| Need to move uncommitted changes to new branch | `git stash` → `git switch -c new-branch` → `git stash pop` |
| Detached HEAD | `git switch -c recovery-branch` to save work, or `git switch master` to discard |
| Diverged from remote | `git pull --rebase origin <branch>` usually resolves it cleanly |