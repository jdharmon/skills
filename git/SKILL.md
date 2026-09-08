---
name: git
description: >
  Use this skill for project-specific git workflows: formatting commits, executing git flow operations (feature, release, hotfix), and determining version tags. Trigger on requests to commit changes, summarize branch changes, tag releases, or manage git-flow branches. Do NOT trigger for general git help or basic troubleshooting.
---

# Git Skill

Project-specific guidance for git operations, safe practices, and git flow branching strategy.

---

## Core Principles

- **Always check status first**: Run `git status` and `git diff` before committing.
- **Prefer explicit over implicit**: Use full commands, not abbreviations.
- **Tags are the version source of truth**: Version numbers are derived from semver git tags (`vMAJOR.MINOR.PATCH`). Never hard-code versions. Use `git describe` or `git tag` to read the current version.

---

## 1. Stage & Commit

When committing work, base your commit message **only** on the staged and unstaged changes. 

### Commit Message Conventions

- **Simple change** (single area, obvious scope): Subject line only.
- **Complex change** (multiple areas, non-obvious motivation): Subject line + blank line + a bulleted list summarizing each significant change.

**Examples:**
*Simple:* `Fixed grid styling. #123`
*Complex:*
```text
Update signature field behavior. #123

* Removed Sign button and moved onClick to grid
* Updated grid text styling
```

### Azure DevOps (ADO) Integration

To associate a commit with an ADO work item without automatically closing it, include the work item number with a pound sign in the subject line (e.g., `#123`).
- Do **NOT** use keywords like `Fixes`, `Fix`, or `Fixed`, as this bypasses the "Resolved" state and closes the ticket prematurely.
- **Multiple Items**: Separate with commas (e.g., `#123, #124`).

### Hard rules
- **No Prefixes**: Do NOT use Conventional Commit prefixes (e.g., `fix:`, `feat:`).
- **No AI References**: Do NOT reference AI tools (GPT, Co-Authored-By) in messages.
- **No Junk Files**: Do NOT stage generated files, build artifacts, or dependency directories (e.g., `node_modules`, `bin/`, `obj/`).

---

## 2. Tags

**Tags matching `vMAJOR.MINOR.PATCH` (e.g. `v1.2.3`) are the single source of truth for version numbers.** Always read the current version from tags before creating a release or hotfix — never assume or hard-code it.

### Reading the current version
```bash
# Most recent semver tag reachable from HEAD
git describe --tags --match "v*"

# Latest semver tag in the repo (top result is the latest)
git tag --list "v*" --sort=-version:refname
```

### Creating and pushing tags

Version tags are created **automatically** when finishing a release or hotfix via `git flow`. Do NOT create version tags manually with `git tag`.

```bash
# Push tags to remote after finishing a release/hotfix
git push origin --tags
```

---

## 3. Git Flow Branching Strategy

Use the `git flow` CLI extension for all git flow operations. If a command fails because git flow is not initialized, run `git flow init` first.

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

# Releases (version = semver without "v", e.g. 1.2.0)
git flow release start <version>     # branch off develop
git flow release finish <version>    # merge to master + develop, tag v<version>

# Hotfixes
git flow hotfix start <name>         # branch off master
git flow hotfix finish <name>        # merge to master + develop, tag
```
