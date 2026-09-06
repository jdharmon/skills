# Gemini 3 Flash (current: `gemini-3.8-flash`)

Read `gemini-family.md` first.

Sources:
- `ai.google.dev/gemini-api/docs/latest-model`
- `ai.google.dev/gemini-api/docs/prompting-strategies` ("Gemini 3 Flash strategies")

**One-line summary:** Flash needs three things the other families don't — an explicit
statement of the current date, an explicit statement of its knowledge cutoff, and (for RAG)
an unusually forceful grounding instruction. Everything else follows the Gemini 3 family
guidance.

## Current model

`gemini-3.8-flash` is GA: 1M token context, 64k max output tokens, `thinking_level` of
`low` / `medium` / `high` with **`medium` as the default**. `minimal` is **not supported**
and returns an error. It's the default model for Managed Agents (the Antigravity agent and
Antigravity SDK).

Positioned for long-horizon software engineering, autonomous agents, and complex enterprise
workflows — not the "cheap and fast" role Flash occupied in earlier generations. By design
it uses more tokens on long-running complex tasks: it takes smaller reasoning steps, calls
tools iteratively, and verifies its work. For everyday tasks, lower the thinking level
rather than prompting the verification away. Gemini 3.7 Flash remains supported if the
older cost profile matters.

## Thinking levels

| Level | Use |
|---|---|
| `low` | Latency-critical work: incident response, real-time chat, draft writing, fast data analysis |
| `medium` (default) | Best quality for most tasks; recommended for complex code and agentic use |
| `high` | Deep reasoning, mathematics, difficult multi-step tasks, maximum tool orchestration |

## The three Flash-specific system instructions

These are published by Google as Flash-targeted fixes. Include the relevant ones verbatim.

**Current-day accuracy.** Flash drifts on the current date, which corrupts search queries
in tool calls:

```
For time-sensitive user queries that require up-to-date information, you
MUST follow the provided current time (date and year) when formulating
search queries in tool calls. Remember it is 2026 this year.
```

Update the year. This is one of the few places where `MUST` is in Google's own recommended
wording.

**Knowledge cutoff accuracy:**

```
Your knowledge cutoff date is January 2025.
```

Verify the cutoff for the specific Flash version you're targeting before using this line.

**Grounding performance.** For RAG and strict context-only answering, Google's block is
deliberately absolute — the redundancy is intentional here, unlike everywhere else in
Gemini prompting:

```
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

Edit for your domain, but don't soften it much — the strength is doing work.

## Migration to 3.8 Flash

- Change the model string to `gemini-3.8-flash`.
- Strip `temperature`, `top_p`, `top_k` from generation configs.
- Replace `thinking_budget` with `thinking_level` (`minimal` unsupported).
- Remove `candidate_count`.
- Standardise multi-turn on server-side `previous_interaction_id`.
- **Remove prefilled model turns.**
- Function calling: put multimodal assets inside the response payload; format inline
  instructions with `\n\n`. On the `generateContent` API, every `FunctionResponse` needs
  `call_id` and `name`. If `Malformed_Function_Call` errors appear tied to pre-tool text,
  check Google's pre-tool-text workarounds.
- Preserve thought signatures per the Gemini 3.5 migration checklist.

Google ships a `gemini-api-dev` skill that automates most of this.

## Gemini 3 Pro

The Pro line follows `gemini-family.md` without Flash's date/cutoff caveats, and generally
needs less explicit scaffolding for multi-step reasoning. This skill doesn't carry verified
Pro-specific behavioural notes — fetch `ai.google.dev/gemini-api/docs/models` and the
current Pro model page before advising on Pro-specific tuning.
