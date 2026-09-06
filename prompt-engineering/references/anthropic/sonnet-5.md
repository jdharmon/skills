# Claude Sonnet 5

Read `../anthropic/family.md` first. This file covers only what differs on Sonnet 5.

Source: `platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-sonnet-5`

**One-line summary:** Sonnet 5 does exactly what you say — no more, no less. Vagueness
gets vague results, scoping words get honoured strictly, and instructions don't silently
generalise. Precision in, precision out.

Existing Sonnet 4.6 prompts run well out of the box. The items below are what usually
needs tuning.

## Breaking changes from Sonnet 4.6

- **Adaptive thinking is on by default.** Requests omitting `thinking` now run *with*
  thinking; on 4.6 the same requests ran without it. Disable with
  `thinking: {type: "disabled"}`.
- **Manual extended thinking removed.** `thinking: {type: "enabled", budget_tokens: N}`
  returns 400.
- **Sampling parameters rejected.** Non-default `temperature`, `top_p`, or `top_k` returns
  400 — new for Sonnet-class models. Steer tone and variety through the system prompt.
- **New tokenizer** producing roughly 30% more tokens for the same text. `max_tokens`
  limits tuned on 4.6 may truncate equivalent output.

## Effort and thinking depth

Defaults to `high`, same as Sonnet 4.6.

| Level | Use |
|---|---|
| `max` | Absolute maximum capability, no token constraint |
| `xhigh` | Hardest coding and agentic use cases |
| `high` | Default; balances tokens and intelligence |
| `medium` | Cost-sensitive work, trading intelligence for tokens |
| `low` | Short scoped tasks and latency-sensitive, non-intelligence-sensitive work |

Rough migration mapping: Sonnet 5 at `medium` ≈ Sonnet 4.6 at `high`; Sonnet 5 at `high` ≈
Sonnet 4.6 at `max`. When benchmarking, match by observed thinking length rather than
effort name.

Sonnet 5 respects effort strictly, especially at the low end — at `low` and `medium` it
scopes work to exactly what was asked rather than going above and beyond. Good for latency
and cost; on moderately complex tasks at `low` there's real risk of under-thinking.

If you see shallow reasoning on complex problems, **raise effort rather than prompting
around it.** If latency forces `low`, add targeted guidance: "This task involves multistep
reasoning. Think carefully through the problem before responding."

If thinking blocks appear more often than wanted (common with large system prompts):
"Thinking adds latency and should only be used when it will meaningfully improve answer
quality, typically for problems that require multistep reasoning. When in doubt, respond
directly."

**Leave `max_tokens` headroom** at `high`/`xhigh`/`max`. `max_tokens` caps thinking *plus*
response text; on long tasks adaptive thinking can consume most of the budget, producing a
response that's nearly all thinking followed by a truncated answer and
`stop_reason: "max_tokens"`. Raise `max_tokens` or drop to `medium`.

## Response length

Sonnet 5 calibrates length to task complexity rather than defaulting to a fixed verbosity
— shorter on lookups, longer on open-ended analysis. If your product depends on a
particular style:

```
Provide concise, focused responses. Skip non-essential context, and keep examples minimal.
```

For specific verbosity patterns (over-explaining, etc.), add targeted instructions.
Positive examples showing the concision you want beat negative instructions.

## Literal instruction following

The defining behaviour. Sonnet 5 interprets prompts literally and explicitly, especially
at lower effort. It does not silently generalise an instruction from one item to another,
and it does not infer requests you didn't make.

This is an advantage for API use cases with carefully tuned prompts, structured
extraction, and pipelines wanting predictable behaviour. The cost is that under-specified
prompts fail visibly. **State scope explicitly**: "Apply this formatting to every section,
not just the first one."

## Tool use triggering

More agentic than Sonnet 4.6 — reaches for tools and runs self-verification loops more
readily. Levers:

- **With thinking disabled**, it's *less* likely to reach for tools or consider searching.
  If you depend on tool calls with thinking off, add an explicit nudge in the system
  prompt.
- **Effort:** `high` and `xhigh` show substantially more tool use in agentic search and
  coding.
- If a specific tool is being ignored (web search is the common case), describe clearly
  why and how it should be used.

## Progress updates

Sonnet 5 gives regular, high-quality updates through long agentic traces on its own.
**Remove scaffolding** like "After every 3 tool calls, summarize progress." If the length
or content of updates isn't calibrated for your use case, describe what updates should look
like and give examples.

## Code review harnesses

If a review harness tuned for an earlier model shows lower recall on Sonnet 5, that's
almost certainly a harness effect, not a capability regression. Given "only report
high-severity issues," "be conservative," or "don't nitpick," Sonnet 5 investigates just as
thoroughly, finds the bugs, and then declines to report findings below your stated bar.
Precision rises; measured recall falls.

Fix by separating finding from filtering:

```
Report every issue you find, including ones you are uncertain about or consider
low-severity. Do not filter for importance or confidence at this stage - a separate
verification step will do that. Your goal here is coverage: it is better to surface a
finding that later gets filtered out than to silently drop a real bug. For each finding,
include your confidence level and an estimated severity so a downstream filter can rank
them.
```

This works even without an actual second stage. If you do want single-pass self-filtering,
be concrete about where the bar sits rather than using qualitative terms like "important":
"report any bugs that could cause incorrect behavior, a test failure, or a misleading
result; only omit nits like pure style or naming preferences."

## Tone and writing style

Prose style on long-form writing shifts from 4.6. If your product relies on a specific
voice, re-evaluate style prompts against the new baseline. Example:
"Use a warm, collaborative tone. Acknowledge the user's framing before answering."

Since `temperature` is rejected, all stylistic variety must come from the prompt.

## Design and frontend defaults

Sonnet 5 settles into a consistent default visual style on open-ended briefs. It reads
fine for some work and wrong for dashboards, dev tools, fintech, healthcare, or enterprise
apps. Generic redirection ("don't use that color," "make it clean and minimal") just moves
it to a different fixed palette.

Two approaches work:

1. **Specify a concrete alternative.** Sonnet 5 follows explicit specs precisely — name
   the palette hexes, the type treatment, the corner radius, the section structure, the
   transition timing. The more concrete, the better the result.
2. **Have it propose options first** — the recommended way to get variety across runs now
   that `temperature` is unavailable:

```
Before building, propose 4 distinct visual directions tailored to this brief (each as: bg
hex / accent hex / typeface, plus a one-line rationale). Ask the user to pick one, then
implement only that direction.
```

A short anti-slop directive works alongside either: no overused font families (Inter,
Roboto, Arial, system fonts), no clichéd colour schemes (especially purple gradients), no
predictable layouts or cookie-cutter component patterns.

## Interactive coding products

Use `xhigh` or `high` effort, add autonomous features like an auto mode, and reduce the
number of required human interactions. Specify task, intent, and constraints fully in the
**first** user turn — ambiguous prompts conveyed progressively across turns reduce token
efficiency and sometimes performance.

## Computer use

Supports `computer_toolset_20260801` (Claude API and Google Cloud) and `computer_20251124`,
plus the browser use tool (`browser_toolset_20260801`). Works up to 2576px / 3.75MP. 1080p
screenshots balance performance and cost; 720p or 1366×768 are lower-cost options with
strong performance.
