# Gemini 3.x — Flash & Flash-Lite (`gemini-3.8-flash`, `gemini-3.5-flash-lite`)

Read `gemini-family.md` first.

Sources:
- `aistudio.google.com/docs/whats-new-gemini-3.8.md`
- `aistudio.google.com/docs/whats-new-gemini-3.6.md`
- `aistudio.google.com/docs/prompting-strategies.md` ("Gemini 3 Flash strategies")
- `aistudio.google.com/docs/models.md`

**One-line summary:** Gemini 3 Flash needs three explicit system-prompt patches (current
date, knowledge cutoff, and — for closed-book RAG — a forceful grounding block), while
`gemini-3.5-flash-lite` remains Google's current fastest/lowest-cost general tier alongside
the Gemini 4 lineup.

## Lineup status

For new general-purpose or coding projects, Google now recommends **`gemini-4-hydrogen`**
(see `gemini-4.md`), while **`gemini-3.5-flash-lite`** remains an active headline tier for
high-volume, sub-second latency workloads:

| Model | Window / Output | `thinking_level` | Role |
|---|---|---|---|
| `gemini-3.5-flash-lite` | 1M in / 64K out | `minimal` (**default**), `low`, `medium`, `high` | Current fastest, lowest-cost general model (high-volume extraction, translation, subagent execution). |
| `gemini-3.8-flash` | 1M in / 64K out | `low`, `medium` (**default**), `high` (`minimal` errors) | Previous-gen agentic Flash for long-horizon coding and enterprise workflows. |
| `gemini-3.6-flash` | 1M in / 64K out | `minimal`, `low`, `medium` (**default**), `high` | Previous-gen balanced Flash. Runs upfront diagnostic code more readily; needs explicit UI styling guidelines if visual polish matters. |
| `gemini-3.1-flash-lite` | 1M in / 64K out | `minimal` (**default**), `low`, `medium`, `high` | Earlier low-cost Flash-Lite tier (upgrade target: `gemini-3.5-flash-lite`). |

**December 1, 2026 auto-routing:** `gemini-2.5-flash` auto-routes to
`gemini-3.5-flash-lite`, and `gemini-2.5-pro` auto-routes to `gemini-3.8-flash`. Update
pinned model strings and remove 2.5-era parameters before that cutoff.

## Tuning `thinking_level`

- **On `gemini-3.8-flash`:** Defaults to `medium` (`minimal` and `xhigh` return errors).
  By design it spends more tokens on complex runs via smaller reasoning steps, iterative
  tool calls, and self-verification. Drop `thinking_level` to `low` for everyday
  latency-sensitive tasks rather than prompting verification away.
- **On `gemini-3.5-flash-lite`:** Keep `minimal` (default) for high-volume extraction,
  routing, or classification. **Raise to `medium` or `high` when running autonomous
  subagents with tool calls, Code Execution, or MCP** — leaving Flash-Lite at `minimal`
  causes premature tool termination on multi-step tasks.

## The three Flash-specific system instructions

Google publishes three targeted system-instruction blocks in `prompting-strategies` for
Gemini 3 Flash. Include the relevant ones verbatim.

**1. Current-day accuracy.** Flash drifts on the current date, corrupting tool search
queries:

```text
For time-sensitive user queries that require up-to-date information, you
MUST follow the provided current time (date and year) when formulating
search queries in tool calls. Remember it is 2026 this year.
```

Update the year as needed. This is one of the few places where `MUST` is in Google's own
recommended wording.

**2. Knowledge cutoff accuracy:**

```text
Your knowledge cutoff date is January 2025.
```

Verify the cutoff on the DeepMind model card for the specific Flash version you're
targeting before using this line (`January 2025` is the literal example in Google's
`prompting-strategies` guide).

**3. Closed-book grounding performance.** For strict context-only RAG, Google's block is
deliberately absolute — the redundancy here is doing real work, unlike elsewhere in Gemini
prompting:

```text
You are a strictly grounded assistant limited to the information provided in
the User Context. In your answers, rely **only** on the facts that are
directly mentioned in that context. You must **not** access or utilize your
own knowledge or common sense to answer. Do not assume or infer from the
provided facts; simply report them exactly as they appear. Your answer must
be factual and fully truthful to the provided text, leaving absolutely no
room for speculation or interpretation. Treat the provided context as the
absolute limit of truth; any facts or details that are not directly
mentioned in the context must be considered **completely untruthful** and
**completely unsupported**. If the exact answer is not explicitly written in
the context, you must state that the information is not available.
```

Edit lightly for your domain, without softening the constraints.

## API rules and migration (`3.6+` and `3.8-flash` / `3.5-flash-lite`)

- **Remove `temperature`, `top_p`, `top_k`:** Deprecated and ignored on Gemini 3.5
  Flash-Lite, 3.6+, and 3.8 Flash (with deprecation warning headers, and `400` errors in
  future generations). Define determinism rules in `system_instruction`.
- **Remove prefilled `model` turns:** Ending `contents` with a non-empty `model` turn
  returns HTTP `400`. Use `system_instruction` or Structured Outputs instead.
- **Remove `candidate_count`** (`> 1` unsupported across Gemini 3+).
- **Replace `thinking_budget` with `thinking_level`.**
- **Context caching:** Unlike Gemini 4 (implicit caching only), Gemini 3.8 Flash, 3.6
  Flash, and 3.5 Flash-Lite still support explicit Context Caching alongside implicit.
- **Multi-turn & function calling:** Prefer `previous_interaction_id` on the Interactions
  API. On `generateContent`, preserve `thought_signature` across turns, include `call_id`
  and `name` on every `FunctionResponse`, place multimodal assets inside the response
  payload, and separate inline instructions with `\n\n` (see Google's pre-tool-text
  workarounds if `Malformed_Function_Call` appears).

## Gemini 3 Pro (`gemini-3.1-pro-preview`)

`gemini-3-pro-preview` is shut down; `gemini-3.1-pro-preview` (`low` / `medium` / `high`,
default `high`) follows `gemini-family.md` without Flash's date/cutoff caveats. Google
recommends migrating 3.1 Pro workloads to **`gemini-4-neon`** or **`gemini-4-carbon`**
(`gemini-4.md`).
