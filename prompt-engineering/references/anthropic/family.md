# Prompting Claude (family-level)

Applies to all current Claude models: Fable 5.1, Mythos 5.1, Fable 5, Mythos 5, Opus 5,
Opus 4.8/4.7/4.6, Sonnet 5, Sonnet 4.6, Haiku 4.5. Read this plus the model file.

Source: `platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices`

## Contents

- [Structure and clarity](#structure-and-clarity)
- [Output and formatting](#output-and-formatting)
- [Thinking and effort](#thinking-and-effort)
- [Tool use](#tool-use)
- [Agentic and long-horizon work](#agentic-and-long-horizon-work)
- [Coding-specific patterns](#coding-specific-patterns)
- [Frontend design](#frontend-design)
- [Deprecated mechanics](#deprecated-mechanics)
- [Model selection quick reference](#model-selection-quick-reference)

## Structure and clarity

**XML tags are the native idiom.** Claude parses `<instructions>`, `<context>`,
`<examples>`, `<input>` reliably and they're the recommended way to disambiguate mixed
content. Nest for hierarchy: `<documents>` containing `<document index="n">` containing
`<document_content>` and `<source>`.

**Explain the why.** Claude generalises from motivation. Stating the reason behind a
constraint extends it correctly to cases you didn't list.

**Ask for "above and beyond" explicitly.** "Create an analytics dashboard" produces a
minimum viable dashboard. "Create an analytics dashboard. Include as many relevant
features and interactions as possible. Go beyond the basics to create a fully-featured
implementation." produces what people usually meant. Animations and interactive elements
in particular must be requested.

**Role in the system prompt.** A single sentence of role framing measurably focuses tone
and domain behaviour.

**Long context:** documents at the top, query at the bottom. Anthropic measures up to
~30% response-quality improvement from this ordering on complex multi-document inputs.
For long-document tasks, ask Claude to quote the relevant passages first, then answer.

**Examples:** 3–5, wrapped in `<example>` tags inside an `<examples>` block, varied
enough to cover edge cases. You can ask Claude to critique your example set for relevance
and diversity.

**Model self-knowledge.** If the app needs Claude to identify itself or emit model
strings, state it: "The current model is Claude Opus 5... the exact model string is
`claude-opus-5`." Models do not reliably know their own identifier.

## Output and formatting

Current Claude models are more concise, more conversational, and less machine-like than
earlier generations, and may skip post-tool-call summaries. If you want visible summaries,
ask for them: "After completing a task that involves tool use, provide a quick summary of
the work you've done."

Verbosity direction differs sharply by model — see the model file. Opus 5 runs long,
Sonnet 5 self-calibrates, Fable 5.1 under-narrates during agentic work.

Four levers for format control, in rough order of effectiveness:

1. Positive instruction rather than prohibition ("Write in smoothly flowing prose
   paragraphs" ≫ "Don't use markdown").
2. XML format indicators: "Write the prose sections in `<smoothly_flowing_prose_paragraphs>`
   tags."
3. Match prompt style to desired output style — removing markdown from your prompt
   reduces markdown in the response.
4. An explicit formatting block for fine control. Anthropic's sample
   `<avoid_excessive_markdown_and_bullet_points>` block is the canonical version: prose
   paragraphs, markdown reserved for inline code / code blocks / `##` headings, no
   ordered or unordered lists unless the items are genuinely discrete or the user asked.
   **Do not apply this block to Fable 5.1**, which already under-formats; it will suppress
   structure the content needs.

**LaTeX** is the default for math. To suppress it, say so explicitly and give the plain-text
substitutes: "/" for division, "*" for multiplication, "^" for exponents; no `\( \)`, `$`,
or `\frac{}{}`.

## Thinking and effort

Claude 4.6+ and Mythos Preview use **adaptive thinking** (`thinking: {type: "adaptive"}`),
where the model decides when and how much to think based on the `effort` parameter and
query complexity. On Fable/Mythos 5.x thinking is always on and adaptive is the only mode.
On Opus 5 and Sonnet 5 thinking is on by default when the `thinking` field is omitted — a
change from 4.6/4.8, where omitting it meant off.

Anthropic's internal evals put adaptive thinking ahead of manual extended thinking.

- Prefer **general** thinking instructions over prescriptive step lists. "Think
  thoroughly" typically beats a hand-written plan; Claude's own decomposition usually
  exceeds what a human would prescribe.
- Few-shot examples containing `<thinking>` blocks will shape the style of Claude's own
  reasoning.
- To *reduce* thinking frequency (common with large system prompts): "Thinking adds
  latency and should only be used when it will meaningfully improve answer quality —
  typically for problems that require multistep reasoning. When in doubt, respond
  directly."
- To reduce thrashing on Opus 4.6-era models: "When you're deciding how to approach a
  problem, choose an approach and commit to it. Avoid revisiting decisions unless you
  encounter new information that directly contradicts your reasoning."
- **Self-check instructions are model-dependent.** "Before you finish, verify your answer
  against [criteria]" helps on most models and *hurts* on Opus 5, which already verifies.
- With thinking disabled on Opus 4.5, the word "think" and its variants are unusually
  sensitive — use "consider," "evaluate," or "reason through."

`budget_tokens` is deprecated on 4.6/Sonnet 4.6 and **400-errors on Claude 4.7 and
later**. Control cost with `effort` plus `max_tokens` instead.

## Tool use

Current models follow instructions literally, so **ask for the action you want**.
"Can you suggest some changes to improve this function?" gets suggestions.
"Change this function to improve its performance" gets edits.

To bias toward action, use a `<default_to_action>` block: implement rather than suggest,
infer the most useful likely action when intent is unclear, discover missing details with
tools instead of guessing. To bias away, use `<do_not_act_before_instructions>`: default
to research and recommendations, only edit when explicitly asked.

**Dial back aggressive triggering language.** Opus 4.5/4.6 and later respond strongly to
the system prompt. Prompts that said "CRITICAL: You MUST use this tool when..." to fix
undertriggering on older models now cause overtriggering. Plain "Use this tool when..."
is correct. Same for "If in doubt, use [tool]" and "Default to using [tool]" — replace
with "Use [tool] when it would enhance your understanding of the problem."

**Parallel tool calls** happen by default with a high success rate. The
`<use_parallel_tool_calls>` block pushes it to near 100%: make all independent calls in
parallel, call sequentially only where one call's parameters depend on another's result,
never use placeholders or guess missing parameters. On Fable 5.1 in long agent loops,
resend this as a turn-scoped system message after each round of tool results.

## Agentic and long-horizon work

**Context awareness** (Sonnet 5, Sonnet 4.6, Sonnet 4.5, Haiku 4.5) lets the model track
its remaining token budget. If your harness compacts context or writes state to files,
say so, or the model may wrap up early:

> Your context window will be automatically compacted as it approaches its limit,
> allowing you to continue working indefinitely from where you left off. Therefore, do
> not stop tasks early due to token budget concerns... Never artificially stop any task
> early regardless of the context remaining.

**Multi-window workflows.** Use a different prompt for the first window (write tests,
create setup scripts) than for continuation windows (iterate a todo list). Have the model
keep tests in a structured file (`tests.json`) and tell it removing or editing tests is
unacceptable. Encourage `init.sh`-style quality-of-life scripts. Prefer a fresh context
window over compaction where possible — current models discover state from the filesystem
very effectively — and be prescriptive about how to start ("Call pwd", "Review
progress.txt, tests.json, and the git logs").

**State management:** JSON for structured state, freeform text for progress notes, git for
checkpoints and history. Ask explicitly for incremental progress.

**Reversibility.** Without guidance, models may take hard-to-reverse actions. The
canonical block: encourage local reversible actions (editing files, running tests),
require confirmation for destructive ops (`rm -rf`, dropping tables, deleting branches),
hard-to-reverse ops (`git push --force`, `git reset --hard`, amending published commits),
and externally visible ops (pushing, commenting on PRs, sending messages). Add: don't use
destructive actions as a shortcut around obstacles, don't bypass safety checks with
`--no-verify`.

**Subagents** are orchestrated natively without instruction. The risk is overuse — Opus
4.6 and Opus 5 both delegate readily. Damping language: delegate for parallel,
context-isolated, independent workstreams; work directly for simple tasks, sequential
operations, single-file edits, and anything needing shared context.

**Research tasks:** define success criteria, ask for cross-source verification, and for
complex work ask for competing hypotheses, tracked confidence levels, periodic
self-critique, and a persisted hypothesis tree or notes file.

## Coding-specific patterns

**Overengineering** (notably Opus 4.5/4.6): scope creep, unnecessary abstractions,
speculative flexibility. Counter with explicit minimalism across four axes — scope (no
unrequested features or refactors), documentation (no docstrings on untouched code),
defensive coding (validate at system boundaries only, trust internal code), abstractions
(no helpers for one-time operations, no designing for hypothetical futures).

**Test-gaming:** "Implement a solution that works correctly for all valid inputs, not just
the test cases. Do not hard-code values... Tests are there to verify correctness, not to
define the solution. If the task is unreasonable or infeasible, or if any of the tests are
incorrect, please inform me rather than working around them."

**Hallucination control:** an `<investigate_before_answering>` block — never speculate
about code you haven't opened; if the user references a specific file, read it before
answering.

**Temp file cleanup:** models use scratch files productively, which helps outcomes. If net
new files are a problem: "If you create any temporary new files, scripts, or helper files
for iteration, clean up these files by removing them at the end of the task."

## Frontend design

Left alone, models converge on "AI slop": Inter/Roboto/Arial, purple gradients on white,
predictable layouts. The `<frontend_aesthetics>` block counters this by directing
attention to typography (distinctive fonts), colour (cohesive commitment, dominant colours
with sharp accents, CSS variables), motion (CSS-only for HTML, Motion for React, one
well-orchestrated page load beats scattered micro-interactions), and backgrounds (layered
gradients, geometric patterns, atmosphere over flat fills). Note that models converge even
on the "distinctive" choices — Space Grotesk is the running example — so the block should
explicitly ask for variety across generations.

Full version: the `frontend-design` skill in the `anthropics/claude-code` repo.

## Deprecated mechanics

- **Assistant prefill** on the last turn: unsupported from Claude 4.6 and Mythos Preview
  onward; returns 400. Migrate format control, preamble suppression, and continuations to
  explicit instructions. Assistant messages elsewhere in the conversation are fine.
- **`budget_tokens`:** 400-errors on 4.7+.
- **Sampling params:** `temperature` / `top_p` / `top_k` 400-error on Sonnet 5. Use
  system-prompt instructions for tone and variety instead.
- **History rewriting:** on Fable 5.1, editing earlier messages, rebuilding `system` or
  `tools`, or summarising older turns in place invalidates every later thinking block.
  Keep history append-only; move changes to mid-conversation system messages.

## Model selection quick reference

| Model | Shape | Read |
|---|---|---|
| Fable 5.1 / Mythos 5.1 | Most capable; under-narrates, formats sparsely, always-on thinking | `fable-mythos-5.md` |
| Opus 5 | Agentic coding and enterprise work; verbose, self-verifying, delegation-happy | `opus-5.md` |
| Sonnet 5 | Coding and agentic; literal, self-calibrating length, more tool-eager than 4.6 | `sonnet-5.md` |
| Haiku 4.5 | Fast/cheap with context awareness; needs more explicit structure and examples | this file |
| Opus 4.8 / 4.6 | Prior generation; overeager, overengineers, over-explores at high effort | this file + live docs |
