---
name: prompt-engineering
description: Write, audit, and port prompts using each vendor's official prompting guidance. Use this whenever the user is working on a prompt, system prompt, agent instruction file, AGENTS.md/CLAUDE.md, tool description, or LLM-facing template — including when they say a model "isn't listening," is too verbose or too terse, over- or under-uses tools, ignores format instructions, or when they're switching a prompt from one model or provider to another. Also use when the user asks how to prompt a specific model (Opus, Sonnet, Haiku, Fable, GPT Sol/Terra/Luna, Astra, Gemini Pro/Flash, Gemma) or asks you to "improve this prompt."
---

# Prompt engineering

Prompt quality is mostly model-specific. The same instruction that fixes verbosity on
one model causes over-verification on another, and instructions written for a previous
generation frequently become the *cause* of the problem on the current one. So the first
job is always to find out which model the prompt runs on, and the second is to read that
model's guidance before writing anything.

## Step 1 — Route

Answer three questions before touching the prompt.

**1. What mode is this?**

| Mode | Signal | Go to |
|---|---|---|
| Author | No prompt exists yet | [Authoring](#authoring) |
| Audit | A prompt exists and misbehaves | [Auditing](#auditing) |
| Port | Prompt works on model A, moving to model B | [Porting](#porting) |

**2. Which model?** Ask if it isn't stated — do not guess. "Claude" or "GPT" is not
specific enough, because behaviour differs sharply *within* a family (Opus 5 verifies
itself unprompted; Sonnet 5 follows instructions literally; Gemma 4 has no real system
role). If the user genuinely doesn't know or targets several, say so explicitly and write
to the intersection of the family guides, flagging what you'd tighten once the target is
fixed.

**3. What surface?** API call, agent harness (Claude Code, Codex, OpenCode), IDE rules
file, or a chat box. Agent harnesses inject their own system prompt that may already
contain the instruction being added — check before adding.

Then load the references. Read **both** the family file and the model file:

| Target | Family reference | Model reference |
|---|---|---|
| Claude Opus 5 | `references/anthropic/family.md` | `references/anthropic/opus-5.md` |
| Claude Sonnet 5 | `references/anthropic/family.md` | `references/anthropic/sonnet-5.md` |
| Claude Fable / Mythos 5.x | `references/anthropic/family.md` | `references/anthropic/fable-mythos-5.md` |
| Claude Haiku 4.5, Opus 4.x, Sonnet 4.x | `references/anthropic/family.md` | (family file covers these) |
| GPT-5.6 Sol / Terra / Luna | `references/openai/family.md` | `references/openai/gpt-5-6.md` |
| GPT-6 Astra, GPT-5.x and earlier | `references/openai/family.md` | (fetch live docs — see below) |
| Gemini 3.x Pro / Flash | `references/google/gemini-family.md` | `references/google/gemini-3-flash.md` |
| Gemma 4 (local, LM Studio / Ollama / llama.cpp) | — | `references/google/gemma-4.md` |
| Anything else | Use [universal principles](#universal-principles) and say the guidance is generic |

**Freshness.** Model lineups turn over every few months and this skill's references are a
snapshot. If the target model isn't in the table, or the user mentions a model you don't
recognise, fetch the vendor's live guidance before answering. Canonical entry points:

- `https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices`
- `https://developers.openai.com/api/docs/guides/latest-model`
- `https://ai.google.dev/gemini-api/docs/prompting-strategies`

## Universal principles

These hold across every current frontier model. Everything model-specific lives in the
references; if a reference contradicts something here, the reference wins.

**Write for a capable stranger.** Show the prompt to someone with no context on the task.
If they'd be confused about what to produce, so is the model. Specify the output format,
the constraints, and what "done" looks like.

**Give the reason, not just the rule.** "Never use ellipses" generalises badly.
"Your response is read aloud by a text-to-speech engine, which can't pronounce ellipses"
lets the model extend the intent to cases you didn't enumerate. This is the single
highest-leverage habit in prompt writing.

**Say what to do, not what to avoid.** "Do not use markdown" performs worse than "Write
in flowing prose paragraphs." Positive examples of the style you want beat prohibitions,
especially for tone and verbosity.

**Delimit sections consistently.** XML-style tags (`<context>`, `<task>`,
`<output_format>`) or Markdown headings both work. Pick one and stay with it inside a
single prompt — mixing them costs more than either choice. Wrap variable/user-supplied
input in its own tag so the model reads it as data rather than instructions.

**Put long context first, the question last.** For documents over ~20k tokens, place the
data at the top and the instruction at the bottom, with a bridging phrase ("Based on the
documents above..."). This ordering is worth a large accuracy difference on
multi-document tasks across all three vendors.

**Use examples, and make them varied.** Three to five is the usual sweet spot. Keep
formatting identical across examples — whitespace, tags, and separators are part of what
the model copies. Too many near-identical examples cause overfitting to their surface
form.

**State each instruction exactly once.** Repetition is not emphasis. Repeated
"ask first" / "be concise" / "always verify" lines compound with the model's own trained
behaviour and produce the opposite of what you wanted: unnecessary approval prompts,
truncated answers, wasted verification passes.

**Match prompt style to desired output style.** A heavily bulleted prompt yields bulleted
answers. If you want prose, write the prompt in prose.

**Define autonomy explicitly.** For anything agentic, state in one place what the model
may do without asking (read files, run tests, edit in-scope code) and what needs
confirmation (external writes, destructive operations, purchases, scope expansion).
One compact policy beats scattered warnings.

**Nothing counts until it's measured.** Every recommendation here and in the references
is directional. Changes get validated against a fixed set of representative tasks, one
change at a time. If the user has no eval set, building a small one is usually the highest
value thing to suggest.

## Prompt skeleton

A default arrangement to adapt, not a mandatory template:

```
<role>            Who the model is, one or two sentences.
<context>         Domain background, product constraints, what the user already knows.
<instructions>    The task. Numbered when order or completeness matters.
<autonomy>        What may be done unprompted vs. what needs confirmation. (Agentic only.)
<examples>        3-5 varied, identically formatted examples.
<output_format>   Structure, length, and what a short answer must still contain.
```

Long context goes above `<instructions>`; the specific question goes last.

## Authoring

1. Route (Step 1) and read the model references.
2. Gather what the model can't infer: domain context, hard constraints, success criteria,
   failure modes that matter, and which ambiguities should trigger a question rather than
   an assumption.
3. Draft with the skeleton, applying model-specific defaults from the references — not
   generic habits. Start lean; add instructions only where a gap is demonstrated.
4. Set the API-side controls alongside the prompt text. Reasoning effort, thinking mode,
   and verbosity parameters are usually a better lever than prompt wording, and the
   references say which parameter maps to which behaviour on that model.
5. Hand back the prompt plus a short list of what to watch for and what to try first if
   it misbehaves.

## Auditing

Work through the prompt looking for these, in order — the first three are the most common
causes of "the model stopped listening":

- **Stale instructions.** Language written for an older model that the current one no
  longer needs. Verification nudges, "if in doubt use the tool," "CRITICAL: you MUST,"
  anti-laziness prompting, and forced progress-update scaffolding are all live examples
  where deleting the line is the fix. Check the model reference for its specific list.
- **Duplication.** The same rule stated in three places, each one strengthening the
  effect until it overshoots.
- **Negative framing.** Prohibitions where a positive description of the target behaviour
  would work better.
- **Literalism traps.** Scoping words like "only report high-severity issues," "be
  conservative," "don't nitpick." Newer models honour these far more faithfully than
  older ones, which reads as a capability regression but isn't.
- **Unscoped instructions.** "Format this section" when every section was meant. State
  the scope; models increasingly won't generalise it for you.
- **Structural problems.** Mixed delimiters, question before long context, examples with
  inconsistent formatting, user input not fenced off from instructions.
- **Parameter-shaped problems.** Verbosity, reasoning depth, and tool eagerness often
  belong in `effort` / `reasoning.effort` / `thinking_level` / `text.verbosity`, not in
  prose. Check the references for deprecated parameters that now hard-error
  (`budget_tokens`, `temperature` on some models, `top_k`/`top_p` on Gemini 3+).
- **Deprecated mechanics.** Assistant prefill, `candidate_count`, prefilled model turns,
  history rewriting that invalidates thinking blocks.

Report findings as: what's wrong, why it's wrong on *this* model, and the replacement
text. Prefer deletions — leaner prompts measurably outperform padded ones.

## Porting

1. Read the model reference for **both** source and target.
2. Strip source-specific compensations first. This is the step people skip. An
   instruction added to fix a weakness in model A is, on model B, an instruction that
   distorts behaviour that was already correct.
3. Translate mechanics: role/system message support, delimiter conventions, thinking and
   effort parameters, tool-calling format, multi-turn state handling.
4. Re-baseline the reasoning/effort setting rather than carrying the number across.
   Equivalent-sounding levels are not equivalent between generations or vendors.
5. Diff the behaviour on representative tasks before and after; expect verbosity, tool
   eagerness, and format adherence to be the first things that drift.

## Anti-patterns

- Adding an instruction without checking whether the harness or the model's own defaults
  already cover it.
- Escalating with emphasis (`CRITICAL`, `ALWAYS`, `NEVER`, all-caps) instead of fixing
  the underlying ambiguity. On current models this reliably causes overtriggering.
- Piling on constraints in one revision, so nothing can be attributed when quality moves.
- Copying a prompt template from a blog post written for a model two generations back.
- Treating a stated persona ("you are a world-class expert") as a substitute for concrete
  constraints and success criteria.
