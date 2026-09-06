# Prompting Gemini (family-level)

Applies to Gemini 3.x models (Pro and Flash lines). Read this plus the model file.

Sources:
- `ai.google.dev/gemini-api/docs/prompting-strategies` (see the "Gemini 3" section)
- `ai.google.dev/gemini-api/docs/latest-model`

## Contents

- [Core principles for Gemini 3](#core-principles-for-gemini-3)
- [Structure templates](#structure-templates)
- [Few-shot examples](#few-shot-examples)
- [Thinking and reasoning](#thinking-and-reasoning)
- [Parameters — mostly leave alone](#parameters--mostly-leave-alone)
- [Grounding and tools](#grounding-and-tools)
- [Agentic workflows](#agentic-workflows)
- [Iteration techniques](#iteration-techniques)

## Core principles for Gemini 3

Gemini 3 models are built for advanced reasoning and instruction following, and respond
best to prompts that are direct, well-structured, and explicit about the task and its
constraints.

- **Be precise and direct.** State the goal clearly and concisely. Avoid unnecessary or
  overly persuasive language — padding and emphasis actively hurt here.
- **Use consistent structure.** Clear delimiters separating parts of the prompt. XML-style
  tags (`<context>`, `<task>`) or Markdown headings both work; **pick one and use it
  consistently within a single prompt.**
- **Define parameters.** Explicitly explain any ambiguous term.
- **Control verbosity deliberately.** Gemini 3 defaults to direct, efficient answers. A
  conversational or detailed response must be requested explicitly — this is the opposite
  default from most earlier models, and the most common source of "why is it so terse."
- **Treat multimodal inputs as equal-class.** Text, images, audio, and video all count;
  reference each modality explicitly in the instructions.
- **Prioritise critical instructions.** Behavioural constraints, persona, and output format
  go in the System Instruction or at the very beginning of the user prompt.
- **Structure for long contexts.** All context first, the specific instruction or question
  at the very **end**.
- **Anchor context.** After a large block of data, bridge with a transition phrase:
  "Based on the information above..."

## Structure templates

**XML form** — the `<context>` tag also signals to the model that the enclosed material is
data, not instructions, which matters when embedding user input:

```
<role>
You are a helpful assistant.
</role>

<constraints>
1. Be objective.
2. Cite sources.
</constraints>

<context>
[User input here - the model knows this is data, not instructions]
</context>

<task>
[The specific user request]
</task>
```

**Markdown form:**

```
# Identity
You are a senior solution architect.

# Constraints
- No external libraries allowed.
- Python 3.11+ syntax only.

# Output format
Return a single code block.
```

**Full template** combining Google's recommended practices — system instruction:

```
<role>
You are Gemini 3, a specialized assistant for [Domain].
You are precise, analytical, and persistent.
</role>

<instructions>
1. **Plan**: Analyze the task and create a step-by-step plan.
2. **Execute**: Carry out the plan.
3. **Validate**: Review your output against the user's task.
4. **Format**: Present the final answer in the requested structure.
</instructions>

<constraints>
- Verbosity: [Low/Medium/High]
- Tone: [Formal/Casual/Technical]
</constraints>

<output_format>
1. **Executive Summary**: [Short overview]
2. **Detailed Response**: [Main content]
</output_format>
```

User prompt: `<context>` … `<task>` … `<final_instruction>`.

## Few-shot examples

Google's guidance is more emphatic about examples than Anthropic's or OpenAI's: **always
include few-shot examples.** Prompts without them are likely to be less effective, and if
the examples are clear enough you can often remove instruction text entirely.

- Use specific and varied examples to narrow focus.
- Experiment with the count — too many causes overfitting to the examples' surface form.
- **Formatting consistency across examples is essential**, since showing the response
  format is a primary purpose. Watch XML tags, whitespace, newlines, and example
  splitters.

**The completion strategy** is a distinctively Gemini technique: provide the beginning of
the desired output and let the model continue the pattern. "Create an outline for an essay
about hummingbirds.\n\nI. Introduction\n *" produces the outline in your format rather than
one the model chose. Same trick for JSON shape, though for complex schemas use the
structured output feature instead of prompting for it.

## Thinking and reasoning

Gemini 2.5 and 3.x automatically generate internal thinking text. Consequences:

- **Don't ask the model to outline, plan, or show reasoning steps in the response itself**
  — that work already happens internally, and requesting it in the output just adds
  visible clutter.
- For heavy-reasoning problems, a simple "Think very hard before answering" improves
  performance at the cost of extra thinking tokens.
- Depth is controlled by `thinking_level` (`low` / `medium` / `high`), not by prompt
  wording. See the model file for per-model defaults and which levels are supported.

## Parameters — mostly leave alone

**Strongly recommended: keep `temperature`, `top_p`, and `top_k` at defaults for Gemini
3.x.** Changing them — particularly temperature below 1.0 — causes looping and degraded
performance, especially on complex mathematical and reasoning tasks. On Gemini 3.8 Flash
these should be stripped from generation configs entirely, along with `candidate_count`
(unsupported from Gemini 3 onward).

`thinking_budget` is replaced by the string enum `thinking_level`.

`stop_sequences` still applies; choose sequences unlikely to appear in generated content.

If the model returns a safety fallback response ("I'm not able to help with that, as I'm
only a language model"), Google's suggested remedy is to **increase** temperature — one of
the few cases where touching it is advised.

## Grounding and tools

Two tools exist specifically to prevent hallucination, and Google's guidance is to enable
them by default rather than prompt around the weakness:

- **Grounding with Google Search** — enable whenever the model may need obscure or recent
  facts.
- **Code execution** — enable whenever the model needs arithmetic, counting, or any
  calculation.

For strict grounding in supplied context only, Google publishes an explicit instruction
block: rely only on facts directly in the context, don't use own knowledge or common
sense, don't assume or infer, treat the context as the absolute limit of truth, and state
when the answer isn't available. It is deliberately heavy-handed — see
`gemini-3-flash.md` for the full text.

## Agentic workflows

Google frames agent prompting as configuring trade-offs along explicit dimensions rather
than writing prose. When designing an agent prompt, decide each of these and state it:

**Reasoning and strategy** — logical decomposition depth; problem diagnosis depth (accept
the obvious cause vs. explore less probable ones); information exhaustiveness (analyse
every policy and document vs. prioritise speed).

**Execution and reliability** — adaptability (adhere to the initial plan vs. pivot when
observations contradict assumptions); persistence and recovery (self-correction depth,
trading success rate against token cost and loop risk); risk assessment (explicitly
separate low-risk reads from high-risk writes).

**Interaction and output** — ambiguity and permission handling (when to assume vs. when to
pause and ask); verbosity alongside tool calls (explain actions vs. stay silent);
precision and completeness (every edge case and exact figures vs. ballpark estimates).

Google publishes a long researcher-evaluated system instruction template covering all nine
of these as numbered rules — logical dependencies and constraints, risk assessment,
abductive reasoning and hypothesis exploration, outcome evaluation and adaptability,
information availability, precision and grounding, completeness, persistence and patience,
and response inhibition. It is on the prompting-strategies page; fetch it verbatim rather
than paraphrasing if you're building a rulebook-following agent.

Two details from it worth carrying into any Gemini agent prompt:

- For exploratory actions like searches, missing *optional* parameters is low risk —
  prefer calling the tool with available information over asking the user, unless a later
  step needs that information.
- On *transient* errors, retry unless an explicit retry limit is reached. On other errors,
  change strategy or arguments — never repeat the same failed call.

## Iteration techniques

Three moves specific to Google's guidance when a prompt isn't working:

1. **Rephrase.** Different wording for the same meaning yields different responses.
   "How do I bake a pie?" / "Suggest a recipe for a pie." / "What's a good pie recipe?"
2. **Switch to an analogous task.** If the model won't stay within your options, restate
   the task in a form that constrains it — reframing a classification as an explicit
   multiple-choice problem is the canonical example.
3. **Reorder prompt content.** Examples / context / input can be permuted, and the order
   measurably affects output.

Also consider decomposition: one prompt per instruction, chained prompts for sequential
steps, or parallel prompts over data partitions with aggregation.
