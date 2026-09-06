# Claude Fable 5.1 / Mythos 5.1 (and Fable 5 / Mythos 5)

Read `../anthropic/family.md` first.

Sources:
- `platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5-1`
- `platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-fable-5`

**Coverage warning.** This file is assembled from the cross-model best-practices page and
the summaries it gives of the Fable-specific guides. It is thinner than the Opus 5 and
Sonnet 5 files. **Fetch the two source pages above before doing substantial Fable/Mythos
prompt work.**

## What these models are

Fable and Mythos share an underlying model; Fable carries additional safety measures for
biology, cybersecurity, and LLM R&D. They sit in the Mythos tier above Opus. Fable 5 and
Mythos 5 target problems previously too complex, long-running, or ambiguous for prior
models, and are especially effective at end-to-end work.

Access note: Claude Mythos Preview is not publicly available (Project Glasswing). Fable
5.x may route some queries to Opus 5 via conservative safeguards, which fire in under 5%
of sessions on average.

## Behavioural differences that change prompts

**Thinking is always on.** Adaptive thinking is the only mode, regardless of whether the
`thinking` parameter is set. There is no thinking-off configuration to prompt around.

**Under-narration during agentic work.** The opposite of Opus 5: Fable 5.1 writes *fewer*
user-facing updates between tool calls. Ask for progress text explicitly, and **remove any
instruction telling it to keep that text brief** — those instructions carried over from
verbose models will silence it almost entirely.

**Sparse formatting by default.** Fable 5.1 already formats less than earlier models.
Applying the standard `<avoid_excessive_markdown_and_bullet_points>` block here will
suppress structure the content genuinely needs. Remove it, or use the much shorter
formatting rule from the model's own guide.

**Denser writing.** Prose density differs from earlier models; re-evaluate style prompts
if your product depends on a particular voice.

**Effort levels** behave differently from Opus 4.8's. Re-baseline rather than carrying a
setting across. At low effort, search triggering in particular changes — verify tool
behaviour at whatever effort you settle on.

**Tool-call batching in agent loops.** Send the `<use_parallel_tool_calls>` instruction as
a **turn-scoped system message after each round of tool results**, not once at the top of
the conversation. A single up-front instruction decays over a long loop.

**Instruction following** is tighter than Opus 4.8's — scoping words, conservatism
instructions, and negative constraints get honoured more literally.

**Long-run progress claims.** Fable 5's guide covers how the model reports progress on
long-horizon runs; check it if you're building anything that reports status back to a user
or an orchestrator.

**Memory systems.** Fable 5's guide has specific guidance on memory-tool scaffolding.
Fetch it if your harness uses a memory tool.

**`reasoning_extraction` refusal category.** New on Fable 5 — requests to extract or
reproduce raw reasoning may refuse. Relevant if your pipeline reads or forwards thinking
content.

## Conversation history must be append-only

The hardest constraint on Fable 5.1. Append each assistant turn **exactly as the API
returned it, thinking blocks included.** Modifying the conversation before a thinking
block errors — or silently drops the block if you opt into that behaviour.

All of these invalidate every later thinking block:

- Editing earlier messages
- Rebuilding `system` or `tools` between requests
- Summarising older turns in place

Move those changes to mid-conversation system messages and server-side context management
instead.

## Prefill

Not supported. Prefilled assistant messages on the final turn return 400 on Claude 4.6+
and Mythos Preview.
