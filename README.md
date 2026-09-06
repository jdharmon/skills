# skills

A collection of [Agent Skills](https://code.claude.com/docs/en/skills) — reusable instructions that teach coding agents (Claude Code, and others that support the same format) how to handle a specific kind of task.

Each skill lives in its own directory with a `SKILL.md` file describing when it should be used and what to do.

| Skill | Description |
|---|---|
| [`git`](git/SKILL.md) | Committing, branching, tagging, and following a git flow strategy |
| [`python`](python/SKILL.md) | Python project and dependency management, built entirely on `uv` |
| [`prompt-engineering`](prompt-engineering/SKILL.md) | Writing, auditing, and porting prompts across model vendors |

## Installing skills

Skills are installed with the [`skills` CLI](https://www.npmjs.com/package/skills), run through `npx` so you don't need to install anything globally.

Install every skill in this repo into the current project:

```bash
npx skills add jdharmon/skills
```

Install just one or a few skills:

```bash
npx skills add jdharmon/skills --skill git,python
```

Install globally (available in every project, not just the current one):

```bash
npx skills add jdharmon/skills --global
```

Other useful commands:

```bash
npx skills add jdharmon/skills --list   # see what's available without installing
npx skills list                         # see what's already installed
npx skills update jdharmon/skills       # pull the latest version of these skills
npx skills remove <skill>               # uninstall a skill
```

## Creating new skills

Scaffold a new skill with:

```bash
npx skills init <name>
```

This creates a `<name>/SKILL.md` file (or `./SKILL.md` if you omit the name and are already inside a skill directory) with the frontmatter Claude expects:

```yaml
---
name: my-skill
description: >
  What this skill does and — most importantly — when it should trigger.
  Be specific about the situations, keywords, and file types that should
  activate it, since this description is what the agent matches against.
---
```

Guidelines for writing a good skill, based on the ones in this repo:

- **Front-load the trigger conditions in `description`.** This is the only part of the skill an agent sees before deciding whether to use it, so it needs to name concrete situations, commands, and keywords — not just a vague topic.
- **Write for an agent, not a human reader.** Prefer direct rules, tables, and command examples over prose explanation.
- **State the reason behind non-obvious rules**, so the agent can generalize to cases you didn't spell out explicitly.
- **Keep it scoped to one domain.** If a skill is trying to cover two unrelated things, split it into two skills.
- **Show, don't just tell.** Concrete command examples and before/after snippets are more reliable than abstract guidance.

Once a skill is ready, commit it to this repo (or your own) and install it the same way as any other skill above.
