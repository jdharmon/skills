# Claude Opus 5 (`claude-opus-5`)

Read `../anthropic/family.md` first. This file covers only what differs on Opus 5.

Source: `platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5`

**One-line summary:** Opus 5 does more on its own than prior models — it verifies, scopes,
delegates, narrates, and writes at length without being asked. Most Opus 5 prompt work is
*deletion*, plus explicit calibration of length and scope.

Existing Opus 4.8 prompts run well out of the box. The items below are the ones that
usually need tuning.

## Defaults worth knowing

- 1M token context window, default and maximum. Instruction following, tool calling, and
  reasoning stay consistent across the whole window.
- Thinking is **on by default**; it can be disabled only at `effort: high` or below.
- `effort` defaults to `high` and matters more than on earlier models.
- 300k output tokens on the Message Batches API with the `output-300k-2026-03-24` beta
  header. Minimum cacheable prompt: 512 tokens.

## Effort

`low` and `medium` deliver strong quality at a fraction of the tokens and latency. Treat
effort as the primary cost/latency control: start at the `high` default, sweep downward
wherever quality holds, and step up to `xhigh` for demanding coding and agentic work.
Re-run an effort sweep on your own evals rather than carrying a setting over from a prior
model.

Effort controls **thinking volume, not visible response length.** Lowering effort will not
reliably shorten what the user sees.

## Delete these from carried-over prompts

Each of these compounds with behaviour Opus 5 already has, costing tokens with no quality
gain:

- Verification instructions — "include a final verification step for any non-trivial
  task," "use a subagent to verify," "double-check your answer," "re-verify before
  responding." Opus 5 verifies and self-corrects reliably unprompted.
- Legacy harness scaffolding that adds separate verification steps.
- Any rule instructing the model not to think or not to reason — this *increases* internal
  tag leakage.

## Response length and verbosity

Default user-facing responses run longer than prior Opus models. Prompt for length
explicitly:

```
Keep responses focused, brief, and concise. Keep disclaimers and caveats short, and spend
most of the response on the main answer. When asked to explain something, give a
high-level summary unless an in-depth explanation is specifically requested.
```

In a long system prompt, pair that with a short reminder near the end:

```
<tone_preference>
Keep outputs reasonably concise.
</tone_preference>
```

## Written deliverable length

Separate problem from conversational verbosity: files Opus 5 writes to disk — reports,
Markdown docs, summaries — run long. If your product ships Claude-authored documents:

```
Match the length of written documents to what the task needs: cover the substance, but do
not pad with filler sections, redundant summaries, or boilerplate.
```

## Agentic narration

Opus 5 announces what it's about to do and produces longer per-message output during
agentic sessions. Describe the cadence you want rather than prohibiting narration:

```
Before your first tool call, say in one sentence what you're about to do. While working,
give a brief update only when you find something important or change direction. When you
finish, lead with the outcome: your first sentence should answer "what happened" or "what
did you find," with supporting detail after it for readers who want it.
```

The same lever works in the other direction. Positive examples of the communication style
you want beat instructions about what not to do.

## Task scope

Opus 5 can expand scope — adding steps that weren't requested, or applying its own
judgment about what the task should be. For narrow tasks:

```
Deliver what was asked, at the scope intended. Make routine judgment calls yourself, and
check in only when different readings of the request would lead to materially different
work. If the request seems mistaken or a better approach exists, say so in a sentence and
continue with the task as asked rather than quietly narrowing, widening, or transforming
it. Finish the whole task, and stop short of actions that are clearly beyond what was
asked.
```

## Subagent delegation

Delegates more readily than prior models. Worth it for genuinely independent, sizeable
tracks; expensive when applied to small tasks.

```
Delegate to a subagent only for large tasks that are genuinely independent and
parallelizable, such as a wide multi-file investigation. Do not delegate work you can
finish yourself in a handful of tool calls, and do not use subagents to verify or
double-check your own work. If one subagent can complete the task, use one rather than
several, and keep spawn counts low.
```

Deterministic caps in Claude Code / Agent SDK: `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`,
`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`, and the SDK's `max_budget_usd`. These need Claude
Code 2.1.217+. Claude Code adds its own delegation instruction on Opus 5 **only** with the
`claude_code` system prompt preset — with a custom or omitted system prompt, add one
yourself.

## Correction narration

Opus 5 narrates corrections to its own earlier statements more than prior models, which
reads badly in user-facing products:

```
Only correct an earlier statement when the error would change the user's code,
conclusions, or decisions. State corrections plainly and briefly, then continue the task.
For slips that change nothing for the user, make the fix and move on without noting it.
```

## Code review

High precision *and* recall — additional findings are mostly real issues, and accuracy
holds at lower effort, which supports a fast pass at review time plus a thorough pass
later. But if the review prompt says "only report high-severity issues" or "be
conservative," Opus 5 follows that literally and reports less. Ask for everything and
filter in a separate pass.

## Running with thinking disabled

Two artifacts can appear, and the primary mitigation for both is **don't disable
thinking** — thinking on at `low` effort outperforms thinking off at similar cost.

1. **Tool calls emitted as text.** The model writes the call into user-facing text instead
   of a structured `tool_use` block. The turn completes, the call never runs, and in agent
   loops the leaked text persists in history and poisons later turns. Most common on
   tool-heavy workloads like search.
2. **Internal XML tags in visible output**, including `<thinking>` tags.

If thinking must stay off, one combined instruction mitigates both:

```
When you use a tool, you may say a brief sentence first. If no tool can express what the
user asked for, say so instead of guessing. Do not include internal or system XML tags in
your response.
```

Instructions that name thinking tags specifically are *less* effective than this general
form.

## Other capability notes

- **Vision:** strong on charts, documents, diagrams, and UI/frontend visual replication.
  Re-validate prompt-side vision workarounds from prior models — they may now be
  unnecessary. Vision is strongest with tools to iteratively analyse, crop, and visually
  verify; tool access is more cost-effective than raising thinking alone.
- **Office documents:** handles multi-sheet spreadsheets with non-trivial formulas and
  produces well-structured decks. Prompt it with the specific styles or templates it
  should follow.
- **Multi-agent:** effective writer-verifier patterns, few cases of agents overwriting
  each other's work.
- **Agentic coding:** completes full tasks rather than leaving stubs. Performs best given
  the complete task specification up front and then left to run — avoid drip-feeding
  partial requirements across turns.
