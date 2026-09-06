# GPT-5.6 — Sol, Terra, Luna

Read `../openai/family.md` first. That file carries most of the GPT-5.6 guidance, because
OpenAI publishes prompting advice per *release* rather than per tier. This file covers
tier selection, pro mode, and the 5.6-specific mechanics.

Source: `developers.openai.com/api/docs/guides/latest-model?model=gpt-5.6`

**One-line summary:** GPT-5.6 infers intent well and is token-efficient, so the winning
move is to remove instructions rather than add them — state the goal, constraints,
approval boundaries, and success criteria, then stop.

## Choosing a tier

| Model | Use for |
|---|---|
| `gpt-5.6-sol` | Flagship capability. The `gpt-5.6` alias routes here. |
| `gpt-5.6-terra` | Balance of intelligence and cost. |
| `gpt-5.6-luna` | Efficient, high-volume workloads. |

Prompting guidance is shared across the three. Where a prompt is tuned for `sol` and moves
to `terra` or `luna`, the usual regressions are in multi-step planning and instruction
adherence under long context — the fix is usually tighter, more explicit structure rather
than more instructions.

## What changed that affects prompts

**Intent understanding.** GPT-5.6 infers the user's underlying goal and intended level of
work from context, so you often don't need to prescribe every step. Keep providing domain
context, hard constraints, approval boundaries, and success criteria. **Tell the model
explicitly when an important ambiguity should trigger a question** rather than an
assumption — it will otherwise proceed.

**Token efficiency.** Flagship-level performance at fewer output tokens. Combined with the
lean-prompt effect, prompts ported from GPT-5.5 or 5.4 are usually carrying dead weight.

**Frontend design.** Stronger layout, visual hierarchy, and design judgment out of the box.
Design-scaffolding instructions written for earlier models may now be redundant.

**Image detail.** With `original` or `auto` detail, GPT-5.6 preserves image dimensions,
except that images over 65,535 px on a side are scaled to fit. Images still exceeding the
30,000-patch limit are **rejected rather than resized.** Large images raise input tokens
and latency.

## Pro mode

`reasoning.mode: "pro"` applies more model work before returning a single final answer.
It's an execution mode on the same model slug — do **not** switch to a separate Pro model.

- Reasoning mode and reasoning effort are **independent.** Choose `reasoning.effort`
  separately; omitting it defaults to `medium` in both standard and pro mode.
- Use pro mode when a marginal quality improvement materially affects the outcome and the
  task is hard enough to benefit: complex optimisation, high-value coding or review, deep
  analysis with clear evaluation criteria.
- Prefer standard mode for routine, latency-sensitive, or high-volume work, and wherever
  evals don't show a meaningful gain.
- Tokens from the extra work aggregate into reported usage and bill at standard rates.

**Keep the same prompt you use in standard mode.** State the goal, context, constraints,
required evidence, success criteria, and output format. Do **not** add "use pro mode,"
"think harder," or "generate several candidate answers" — those are the standard-mode
workarounds pro mode replaces.

Example of the right prompt shape:

```
Review this database migration plan for failure modes that could cause data loss
or extended downtime. For each finding, cite the relevant step, estimate impact
and likelihood, and recommend a specific mitigation. Return the five most
important risks in severity order.
```

Compare standard and pro on the same representative tasks, measuring task success, answer
completeness, required evidence, total tokens, latency, and cost. Start from the same
model and effort as your standard-mode baseline rather than assuming highest effort wins.

## Programmatic Tool Calling template

When both direct and programmatic routes are available, define one clear handoff and tell
the model not to switch routes or repeat completed work:

```
<tool_orchestration>
Use Programmatic Tool Calling for [bounded stage] using only [eligible tools].
Run independent calls concurrently when safe. Use only documented tool input
and output fields.

Process and reduce the intermediate results, then emit exactly [output schema],
including the evidence needed for the final answer.

Stop when [condition] is met. Retry transient failures at most [R] times.
Do not repeat completed calls or perform side-effecting actions. If a required
result is still missing, return a clear structured failure.

Use direct tool calls for [semantic judgment, approval, or final validation].
</tool_orchestration>
```

Implementation notes: add the `programmatic_tool_calling` tool, opt eligible tools in with
`allowed_callers`, and handle `program` items, program-issued function calls, and
`program_output` items while preserving each call's `call_id` and `caller` linkage. PTC is
ZDR-compatible with no additional container costs.

Benchmark before adopting. Fewer calls, turns, or intermediate outputs count as
improvements **only** when the final answer still meets the quality bar.

## Migration checklist from GPT-5.5 / 5.4

1. Pick the tier; update the model slug.
2. Move to the Responses API if not already there.
3. Set `reasoning.effort` explicitly; test the current level and one level lower.
4. Configure `reasoning.context` — GPT-5.6 defaults to `all_turns`. Check the response's
   `reasoning.context` field to confirm the effective mode.
5. Strip brevity instructions and re-measure; 5.6 is already more concise than 5.5.
6. Consolidate approval language into one policy block; delete every duplicate "ask
   first."
7. Audit prompt caching for the 1.25× cache-write rate.
8. Prune tools and tool descriptions down to what the task actually needs.

OpenAI ships an `openai-docs` skill that can apply the mechanical parts of this migration
automatically, available from `github.com/openai/skills`.
