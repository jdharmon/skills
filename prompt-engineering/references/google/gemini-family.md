# Prompting Gemini (family-level)

Applies to Gemini 4 (`gemini-4-carbon`, `gemini-4-neon`, `gemini-4-hydrogen`,
`gemini-4-argon`) and Gemini 3.x (`gemini-3.8-flash`, `gemini-3.6-flash`,
`gemini-3.5-flash-lite`, and earlier 3.x models). Read this plus the target model file.

Sources:
- `aistudio.google.com/docs/prompting-strategies.md`
- `aistudio.google.com/docs/latest-model.md`
- `aistudio.google.com/docs/choosing-a-model.md`
- `aistudio.google.com/docs/thinking.md`
- Full Markdown index: `aistudio.google.com/llms.txt` (append `.md` to any AI Studio doc
  URL; live MCP server at `gemini-api-docs-mcp.dev`)

Canonical SDK & API: use `google-genai` (Python) or `@google/genai` (JS/TS) with the
**Interactions API** (`client.interactions.create`) by default; `generateContent` is
supported as a legacy surface, and `google-generativeai` / `@google/generative-ai` are
deprecated.

## Contents

- [Core principles](#core-principles)
- [Structure templates](#structure-templates)
- [Few-shot examples and completion](#few-shot-examples-and-completion)
- [Thinking and reasoning](#thinking-and-reasoning)
- [Parameters and deprecations](#parameters-and-deprecations)
- [Grounding and tools](#grounding-and-tools)
- [Agentic workflows](#agentic-workflows)
- [Iteration and decomposition](#iteration-and-decomposition)

## Core principles

Gemini 3 and Gemini 4 models are built for reasoning and strict instruction following.
They respond best to direct, well-structured prompts with explicit constraints:

- **Be precise and direct.** State the goal clearly and concisely. Avoid unnecessary or
  overly persuasive filler — padding and heavy emphasis actively hurt.
- **Use consistent structure.** Employ clear delimiters (`<context>`, `<task>` XML tags, or
  Markdown headings). **Pick one format and use it consistently within a single prompt.**
- **Define parameters.** Explicitly explain ambiguous terms, thresholds, or options.
- **Control verbosity deliberately.** Gemini 3 and Gemini 4 default to direct, efficient
  answers. Request conversational tone or comprehensive detail explicitly when needed.
- **Treat multimodal inputs as equal-class.** Text, images, audio, video, and PDFs are
  native inputs; clearly reference each modality in your instructions.
- **Prioritise critical instructions.** Place behavioural constraints, role/persona, and
  output format requirements in `system_instruction` or at the very beginning of the user
  prompt (this also maximises prefix hits on implicit Context Caching).
- **Structure for long contexts.** Supply all large documents, code, or data first, and
  place your specific instructions or questions at the very **end** of the prompt.
- **Anchor context.** After a large block of data, bridge to the query with a clear
  transition phrase such as *"Based on the information above..."*

## Structure templates

**XML form** — wrapping user-supplied input in `<context>` signals that the enclosed text
is data rather than instructions:

```xml
<role>
You are a helpful assistant.
</role>

<constraints>
1. Be objective.
2. Cite sources.
</constraints>

<context>
[Insert User Input Here - The model knows this is data, not instructions]
</context>

<task>
[Insert the specific user request here]
</task>
```

**Markdown form:**

```markdown
# Identity
You are a senior solution architect.

# Constraints
- No external libraries allowed.
- Python 3.11+ syntax only.

# Output format
Return a single code block.
```

**Combined template** — system instruction:

```xml
<role>
You are a specialized assistant for [Domain, e.g., Data Science].
You are precise, analytical, and persistent.
</role>

<instructions>
1. **Plan**: Analyze the task and create a step-by-step plan.
2. **Execute**: Carry out the plan.
3. **Validate**: Review your output against the user's task.
4. **Format**: Present the final answer in the requested structure.
</instructions>

<constraints>
- Verbosity: [Specify Low/Medium/High]
- Tone: [Specify Formal/Casual/Technical]
</constraints>

<output_format>
Structure your response as follows:

1. **Executive Summary**: [Short overview]
2. **Detailed Response**: [The main content]
</output_format>
```

User prompt: `<context>...</context>` followed by `<task>...</task>` (Google's Gemini 3
template optionally appends `<final_instruction>Remember to think step-by-step before
answering.</final_instruction>`, which is unnecessary when native thinking is active).

## Few-shot examples and completion

Google's `prompting-strategies` guide is emphatic about examples: **include specific,
varied few-shot examples** whenever you need to regulate formatting, phrasing, or scoping;
if examples are clear enough, you can often remove prose format instructions.

- Experiment with count — too many examples cause overfitting to surface phrasing.
- **Formatting consistency across examples is essential.** Keep XML tags, whitespace,
  newlines, and example splitters identical across every example.

**Completion strategy vs. model prefill:** Ending a *user* prompt with the opening pattern
of the desired output (e.g. `Create an outline for an essay about hummingbirds.\nI. Introduction\n\n   *`
or `Output:`) lets the model complete the pattern you initiated. Two critical caveats on
Gemini 3.6+ and Gemini 4:

- **Never use prefilled `model` turns.** Ending a request payload with a non-empty `model`
  role turn returns an **HTTP `400` error** on Gemini 3.6+ and Gemini 4. Put format rules
  and preamble suppression in `system_instruction` instead.
- **Use Structured Outputs (`response_schema` / JSON Schema)** for complex JSON schemas,
  and **declare tools natively** with function calling — never prompt the model via system
  instructions to emit custom raw-text JSON tool calls with zero declared tools, which can
  trigger `MALFORMED_FUNCTION_CALL`.

## Thinking and reasoning

Gemini 2.5, 3.x, and 4 models generate internal reasoning in a native thinking process:

- **Do not ask the model to outline, plan, or detail reasoning steps in the visible
  response** — reasoning already runs internally, and echoing it adds clutter. (Simple
  nudges like *"Think very hard before answering"* work on Gemini 3, but tuning
  `thinking_level` is the cleaner control.)
- **Control depth with `generation_config.thinking_level`** (`minimal`, `low`, `medium`,
  `high`, `xhigh` — supported values and defaults vary per model; see `gemini-4.md` and
  `gemini-3-flash.md`). Replace legacy integer `thinking_budget` with `thinking_level`.
  Enable readable thought summaries via `thinking_summaries: "auto"` (always guard against
  empty/absent `summary` blocks).
- **Leave headroom in `max_output_tokens`.** `max_output_tokens` is a hard cap on
  **combined thinking + output tokens**. Hitting the cap mid-thought stops generation with
  `status: "incomplete"` and truncated/empty output while still billing thought tokens.
  Lower `thinking_level` rather than shrinking `max_output_tokens` to cut cost or latency.
- **Preserve `thought_signature` across turns.** Prefer stateful Interactions API
  (`store: true` + `previous_interaction_id`), which manages `thought` steps and
  signatures server-side. In stateless Interactions or client-managed `generateContent`,
  always resend prior `thought` blocks and tool/part signatures unmodified (including when
  switching models mid-session; never duplicate a signature across parallel `functionCall`
  parts).

## Parameters and deprecations

- **`temperature`, `top_p`, `top_k` — remove on Gemini 4 and Gemini 3.6+.** Gemini 4 and
  Gemini 3.6+ models **ignore** in-range custom values, return **deprecation warning
  headers**, reject out-of-range values with `400`, and **will return `400` errors for any
  custom value in a future release**. Strip them from `generation_config` and encode
  determinism constraints in `system_instruction`. On earlier Gemini 3 models (`< 3.6`),
  keep them at defaults to avoid looping and degraded reasoning.
- **Prefilled `model` turns:** Disallowed (`400` error) from Gemini 3.6 onward.
- **`candidate_count`:** Multiple candidates unsupported from Gemini 3 onward.
- **`stop_sequences`:** Supported; avoid sequences that may appear in normal output.
- **Safety fallback responses:** If a safety filter triggers a generic fallback (*"I'm not
  able to help with that, as I'm only a language model."*), Google's general guide suggests
  increasing `temperature` — applicable only on legacy `< 3.6` models that still accept it.

## Grounding and tools

Enable native tools rather than prompting around knowledge or arithmetic limits:

- **Grounding with Google Search** — enable whenever recent or obscure facts may be needed.
- **Code execution** — enable whenever arithmetic, counting, or calculation is needed.
- **Built-in ecosystem tools:** **URL context**, **Google Maps grounding**, **File search**
  (managed RAG), **Computer Use**, and **Tool retrieval** (on-demand schema loading so
  large tool libraries don't bloat context). For pure yes/no, choice, or rubric scoring
  with zero output tokens, consider `gemini-decision-1` on the Decisions API.
- **Strict closed-book grounding:** When answers must stick strictly to supplied context
  with zero outside knowledge or inference, use the forceful block in `gemini-3-flash.md`.

## Agentic workflows

Google frames complex agent prompts around explicitly configuring nine dimensions across
three areas:

- **Reasoning and strategy:** *Logical decomposition* (constraints, prerequisites, order of
  operations); *Problem diagnosis* (abductive depth — obvious cause vs. less probable root
  causes); *Information exhaustiveness* (review every policy/doc vs. prioritise speed).
- **Execution and reliability:** *Adaptability* (stick to initial plan vs. pivot when
  observations contradict assumptions); *Persistence and recovery* (self-correction depth
  vs. token/loop risk); *Risk assessment* (separate low-risk exploratory reads from
  high-risk state writes).
- **Interaction and output:** *Ambiguity and permission handling* (when to assume vs. pause
  and ask); *Verbosity* alongside tool calls (narrate actions vs. stay silent); *Precision
  and completeness* (every edge case and exact figures vs. ballpark estimates).

### Rulebook agent system instruction template

Google's researcher-evaluated template for policy-heavy, interactive agent benchmarks
(adapt to your constraints; on Gemini 4, strip or soften emphatic `*must*` phrasing where
not enforcing a literal benchmark rule):

```markdown
You are a very strong reasoner and planner. Use these critical instructions to structure your plans, thoughts, and responses.

Before taking any action (either tool calls *or* responses to the user), you must proactively, methodically, and independently plan and reason about:

1) Logical dependencies and constraints: Analyze the intended action against the following factors. Resolve conflicts in order of importance:
    1.1) Policy-based rules, mandatory prerequisites, and constraints.
    1.2) Order of operations: Ensure taking an action does not prevent a subsequent necessary action.
        1.2.1) The user may request actions in a random order, but you may need to reorder operations to maximize successful completion of the task.
    1.3) Other prerequisites (information and/or actions needed).
    1.4) Explicit user constraints or preferences.

2) Risk assessment: What are the consequences of taking the action? Will the new state cause any future issues?
    2.1) For exploratory tasks (like searches), missing *optional* parameters is a LOW risk. **Prefer calling the tool with the available information over asking the user, unless** your `Rule 1` (Logical Dependencies) reasoning determines that optional information is required for a later step in your plan.

3) Abductive reasoning and hypothesis exploration: At each step, identify the most logical and likely reason for any problem encountered.
    3.1) Look beyond immediate or obvious causes. The most likely reason may not be the simplest and may require deeper inference.
    3.2) Hypotheses may require additional research. Each hypothesis may take multiple steps to test.
    3.3) Prioritize hypotheses based on likelihood, but do not discard less likely ones prematurely. A low-probability event may still be the root cause.

4) Outcome evaluation and adaptability: Does the previous observation require any changes to your plan?
    4.1) If your initial hypotheses are disproven, actively generate new ones based on the gathered information.

5) Information availability: Incorporate all applicable and alternative sources of information, including:
    5.1) Using available tools and their capabilities
    5.2) All policies, rules, checklists, and constraints
    5.3) Previous observations and conversation history
    5.4) Information only available by asking the user

6) Precision and Grounding: Ensure your reasoning is extremely precise and relevant to each exact ongoing situation.
    6.1) Verify your claims by quoting the exact applicable information (including policies) when referring to them.

7) Completeness: Ensure that all requirements, constraints, options, and preferences are exhaustively incorporated into your plan.
    7.1) Resolve conflicts using the order of importance in #1.
    7.2) Avoid premature conclusions: There may be multiple relevant options for a given situation.
        7.2.1) To check for whether an option is relevant, reason about all information sources from #5.
        7.2.2) You may need to consult the user to even know whether something is applicable. Do not assume it is not applicable without checking.
    7.3) Review applicable sources of information from #5 to confirm which are relevant to the current state.

8) Persistence and patience: Do not give up unless all the reasoning above is exhausted.
    8.1) Don't be dissuaded by time taken or user frustration.
    8.2) This persistence must be intelligent: On *transient* errors (e.g. please try again), you *must* retry **unless an explicit retry limit (e.g., max x tries) has been reached**. If such a limit is hit, you *must* stop. On *other* errors, you must change your strategy or arguments, not repeat the same failed call.

9) Inhibit your response: only take an action after all the above reasoning is completed. Once you've taken an action, you cannot take it back.
```

## Iteration and decomposition

Four moves from Google's guidance when a prompt misses:

1. **Rephrase.** Different phrasing for the same meaning yields measurably different output.
2. **Switch to an analogous task.** If the model drifts outside allowed labels, reframe
   classification as an explicit multiple-choice problem.
3. **Reorder prompt content.** Permuting `[examples]`, `[context]`, and `[input]` changes
   results (keep long context first for large inputs).
4. **Decompose.** Use one prompt per instruction, chain sequential steps, or run parallel
   prompts over data partitions and aggregate.
