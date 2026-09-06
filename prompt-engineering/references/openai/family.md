# Prompting GPT models (family-level)

Read this plus the model file. Source: `developers.openai.com/api/docs/guides/latest-model`
(the page has a model selector — pick the target model's tab).

## Contents

- [The lineup](#the-lineup)
- [The core habit: lean prompts](#the-core-habit-lean-prompts)
- [Autonomy and approval boundaries](#autonomy-and-approval-boundaries)
- [Length, style, and tone](#length-style-and-tone)
- [Reasoning effort](#reasoning-effort)
- [Tool orchestration](#tool-orchestration)
- [Safeguards](#safeguards)
- [Migration habits](#migration-habits)

## The lineup

GPT-5.6 introduced a naming scheme where the family alias routes to a flagship tier:

| Alias / slug | Tier |
|---|---|
| `gpt-5.6` | Alias → routes to `gpt-5.6-sol` |
| `gpt-5.6-sol` | Flagship capability |
| `gpt-5.6-terra` | Strong performance at lower price |
| `gpt-5.6-luna` | Efficient, high-volume workloads |

**GPT-6 Astra is the current default model page** on OpenAI's docs, sitting above the
GPT-5.6 family. This skill has verified detail for GPT-5.6 only. For Astra, or for GPT-5.x
and GPT-4.1, fetch the guide with the model selector set to that model before advising —
do not extrapolate from the 5.6 page.

Use the **Responses API** for reasoning, tool-calling, and multi-turn workflows.

## The core habit: lean prompts

This is the through-line of OpenAI's current guidance and the biggest single difference in
philosophy from the Claude docs. Removing repeated instructions, examples, and elaborate
tool descriptions **improves task performance as well as cost.** In OpenAI's internal
coding-agent evals, leaner system prompts improved scores by roughly 10–15% while cutting
total tokens 41–66% and cost 33–67%. Those ranges are directional — validate on your own
workload.

The method:

1. Start from a prompt and tool set that already works.
2. Remove **one group** of instructions, examples, or tools at a time.
3. Rerun the same evals after each removal.

Rules that fall out of this:

- **State each instruction exactly once.** Repeating "ask first," "do not mutate," or
  "wait for approval" causes unnecessary approval requests for safe, expected actions.
- **Expose only tools relevant to the task**, with concise, precise descriptions. Tool
  surface area is prompt surface area.
- **Keep examples and style guidance only** where they encode a product requirement or
  correct a measured gap. Not as decoration.
- **Track context growth**, not just the starting prompt. Long sessions amplify repeated
  prompt and tool content.

## Autonomy and approval boundaries

Current GPT models are proactive and persistent on multi-step tasks. Define what each
request authorises, so the model continues safe in-scope work without pausing while still
stopping before external, destructive, costly, or scope-expanding actions.

A compact policy is usually sufficient:

```
For requests to answer, explain, review, diagnose, or plan, inspect the relevant
materials and report the result. Do not implement changes unless the request also
asks for them.

For requests to change, build, or fix, make the requested in-scope local changes
and run relevant non-destructive validation without asking first.

Require confirmation for external writes, destructive actions, purchases, or a
material expansion of scope.
```

Name the safe local actions explicitly — reading files, inspecting logs, editing in-scope
code, running tests. Keep the whole policy in **one place** and state each rule once.

## Length, style, and tone

**Verbosity has a parameter.** `text.verbosity` (`low` / `medium` / `high`) sets the
default level of detail per request. Use it for the baseline and reserve the prompt for
task-specific length, structure, and required content.

**Re-check brevity instructions when migrating.** GPT-5.6 is more concise by default than
GPT-5.5. Blanket instructions like "Be concise" or "Keep it short" may now be unnecessary,
and can make responses too brief. Keep them only where they reliably produce what the app
needs.

**Specify what a short answer must preserve**, rather than just asking for short:

```
Lead with the conclusion. Include the evidence needed to support it, any material
caveat, and the next action. Omit secondary detail and repetition.

Keep all required facts, decisions, caveats, and next steps. Trim introductions,
repetition, generic reassurance, and optional background first.
```

This gives a priority order rather than a length target.

**Define tone by writing choices, not adjectives.** "Friendly" and "empathetic" are
ambiguous. Describe the behaviour:

```
State the answer directly. If the user reports a problem, acknowledge the
specific issue before giving the next step. Use reassurance only when it is
relevant. Omit generic praise and unnecessary sign-offs.
```

## Reasoning effort

`reasoning.effort` on GPT-5.6 supports `none`, `low`, `medium`, `high`, `xhigh`, `max`.
Set it intentionally rather than leaving it implicit:

- Migrating: preserve the current level as baseline, then compare **one level lower.**
  Newer models often hold quality with fewer tokens.
- `none`: keep as the latency baseline, but also test `low` where the workflow benefits
  from reasoning or tool use.
- `medium`: balanced starting point. `low` for latency-sensitive work.
- `high` / `xhigh`: when more reasoning produces a *measured* quality gain.
- `max`: hardest quality-first workloads only. Compare against `xhigh`.

**Persisted reasoning** (`reasoning.context`) is new in GPT-5.6, defaulting to `all_turns`
where earlier models defaulted to `current_turn`. Use `all_turns` when goals, assumptions,
and priorities stay stable across turns; `current_turn` when earlier reasoning is no
longer relevant. With `all_turns`, continue via `previous_response_id`. Managing history
manually means preserving and resending previous user inputs and **every** response output
item; under `store: false` or ZDR, replay the encrypted reasoning items the API returns.

## Tool orchestration

**Programmatic Tool Calling (PTC)** lets the model write JavaScript that calls eligible
tools in a hosted runtime, passing results between calls. It fits bounded workflows where
code processes several tool results or large intermediate outputs and returns a much
smaller structured result: filtering, joining, ranking, deduplication, aggregation,
validation.

Multiple, parallel, or dependent calls alone **do not** justify PTC. Prefer direct calls
when one call suffices, intermediate outputs are already small, each result may change the
next decision, an action needs approval, or the final output must preserve citations or
native artifacts.

Routing instructions must be task-specific — "use Programmatic Tool Calling efficiently"
does not produce the right route. State which bounded stage uses PTC, which tools it may
call, the exact output schema and required evidence, concurrency/retry/stopping limits,
and which work stays direct. Tool descriptions should document expected return fields,
types, and error behaviour; if the model can't determine the return shape before writing
the program, prefer direct calling.

Test the `program_output` item **and** the final assistant message separately. A program
can return correct records while the message omits a required field, citation, or caveat.

**Multi-agent** (beta) lets one instance coordinate parallel subagents and synthesise
results — useful for complex tasks that divide into genuinely independent workstreams.

## Safeguards

Real-time cyber and biology misuse classifiers run on outputs as they generate. They may
block or refuse requests, or pause generation mid-stream for several seconds. They
occasionally intervene on legitimate work, particularly dual-use areas where defensive and
offensive activity look similar early on — code review, vulnerability research, patch
development, debugging, security education, defensive testing.

If the application serves individual end users, send a stable, privacy-preserving
`safety_identifier` with each request.

## Migration habits

- Choose the tier deliberately (`sol` / `terra` / `luna`) rather than defaulting to the
  alias.
- Carry the reasoning effort as a baseline, then test one level lower.
- Audit prompt caching. GPT-5.6 bills cache writes at 1.25× the uncached input rate, so
  track `cached_tokens` and `cache_write_tokens` for net cost. Use explicit breakpoints or
  `prompt_cache_options.mode: "explicit"` to avoid unnecessary writes; replace
  `prompt_cache_retention` with `prompt_cache_options.ttl`.
- Re-evaluate every brevity and "ask first" instruction against the lean-prompt rule
  before shipping.
