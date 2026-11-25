# Gemini 4 — Carbon, Neon, Hydrogen, Argon

Read `gemini-family.md` first.

Sources:
- `aistudio.google.com/docs/latest-model.md`
- `aistudio.google.com/docs/choosing-a-model.md`
- `aistudio.google.com/docs/whats-new-gemini-4-carbon.md`
- `aistudio.google.com/docs/whats-new-gemini-4-neon.md`
- `aistudio.google.com/docs/whats-new-gemini-4-hydrogen.md`
- `aistudio.google.com/docs/whats-new-gemini-4-argon.md`

**One-line summary:** All four Gemini 4 models share a **unified 1M-token window** (input +
thinking + output), long decode up to 1M, implicit-only caching, and native tool/verification
discipline — so most work when porting to Gemini 4 is picking the right tier + `thinking_level`,
raising low `max_output_tokens` caps, and deleting legacy sampling/caching/scaffolding.

## Choosing a Gemini 4 tier

Every Gemini 4 model accepts Text, Image, Audio, Video, and PDF, outputs Text, and supports
Search/Maps grounding, Code Execution, Function Calling, Computer Use (agentic video
understanding + UI), File Search, Tool Retrieval, URL Context, Structured Outputs, and
Managed Agents (Antigravity).

| Model | Speed | `thinking_level` | Best for |
|---|---|---|---|
| `gemini-4-hydrogen` | Fast (Free + Paid) | `low`, `medium`, `high` (**default**), `xhigh` | Default general-purpose model & default for Antigravity Managed Agent. Everyday coding, interactive apps, multimodal pipelines, parallel worker subagents. |
| `gemini-4-neon` | Moderate (Paid) | `low`, `medium`, `high` (**default**), `xhigh` | Interactive complex software engineering, multi-file refactors, repo-wide debugging, dense legal/finance/science synthesis with fast end-to-end completion. |
| `gemini-4-carbon` | Deliberate (Paid) | `low`, `medium` (**default**), `high` | Deepest reasoning & verification: long-horizon async runs, hard root-cause bugs, formal math/science proofs, dense multimodal precision, security audits. |
| `gemini-4-argon` | Efficient frontier (Paid) | `low`, `medium` (**default**), `high` | Same schema and `low`/`medium`/`high` profile as Carbon, spending fewer thinking tokens per turn for faster interactive engineering loops. |

For ultra-low-cost high-throughput extraction/translation, use `gemini-3.5-flash-lite`
(`gemini-3-flash.md`); for pure yes/no, choice, or rubric scoring with zero output tokens,
use `gemini-decision-1` (Decisions API).

## Behavioural shifts that affect prompts

- **Self-verification and grounded claims out of the box.** Carbon runs deep build-and-verify
  loops; Neon cuts redundant file reads, task shortcuts, and agentic overreach across long
  sessions; Hydrogen cuts fabricated claims by **>5x** compared to `gemini-3.8-flash` by
  flagging missing context instead of guessing, and batches read/search calls in parallel.
  **Delete carried-over anti-laziness prompts, repeated verification nudges, and "always
  read again" rules** — they compound with built-in verification and waste thinking tokens.
- **Strict schema and tool-declaration expectations.** Gemini 4 adheres tightly to
  Structured Outputs (`response_schema`). **Never** instruct Gemini 4 in `system_instruction`
  to emit custom raw-text JSON tool calls when zero tools are declared — that triggers
  `MALFORMED_FUNCTION_CALL`. Always declare tools natively via Function Calling or constrain
  JSON via Structured Outputs.
- **Implicit-only Context Caching.** Explicit caching (`cachedContents` / `caches.create` /
  `cached_content`) is **not supported** on any Gemini 4 model — remove `cached_content`
  fields. Implicit caching runs automatically on prompts `>= 4,096` tokens (95% input
  discount on hits), so **front-load static `system_instruction`, tool schemas, and shared
  reference docs** before dynamic per-turn content.

## Thinking levels — watch the per-tier defaults

Note the split across the family — **Hydrogen and Neon default to `high` and support
`xhigh`**, whereas **Carbon and Argon default to `medium` and cap at `high`**. Passing
`minimal` returns an error on **all** Gemini 4 models; passing `xhigh` is valid only on
Hydrogen and Neon.

| Level | Supported on | Guidance |
|---|---|---|
| `low` | All Gemini 4 | Cost- and latency-sensitive chat, summarization, straightforward tool calls. |
| `medium` | All Gemini 4 (**default on Carbon & Argon**) | Recommended baseline on Carbon/Argon; good step-down on Neon/Hydrogen for interactive coding and live UI prototyping. |
| `high` | All Gemini 4 (**default on Neon & Hydrogen**) | Standard setting on Neon/Hydrogen for coding, agents, and multimodal work; maximum depth on Carbon/Argon. |
| `xhigh` | **Neon & Hydrogen only** | Reserve for hard multi-step problems where `high` falls short. **Avoid on simple extraction/JSON prompts** — causes overthinking and can exhaust `max_output_tokens`. |

Sweep all supported levels on your eval suite with `max_output_tokens >= 65536`, then pick
the lowest `thinking_level` that clears your quality bar.

## Unified 1M window, output caps, and Long Decode

Unlike Gemini 3's separate 1M input + 64K output budgets, Gemini 4 shares **one
`1,048,576`-token window across input, thinking, and output**:

- **Raise legacy `max_output_tokens` caps.** Because `max_output_tokens` caps **thinking +
  output combined**, hardcoded 4K/8K caps from older models starve the final answer. Keep
  the `65,536` default (or omit `max_output_tokens`), and never set below `32,768` on
  reasoning workloads.
- **Non-streaming hop difference:** Foreground calls (`background=False`) generate up to
  `65,536` tokens per hop on Hydrogen and Neon (streaming or non-streaming) and on
  streaming Carbon/Argon calls, but **non-streaming Carbon and Argon calls cap at `32,768`
  tokens per foreground hop**.
- **Long decode (> 64K up to 1M):** To generate past `65,536` tokens (up to `1,048,576`
  minus input tokens), set **`background=True` AND `max_output_tokens > 65536`** (e.g.
  `131072` or `262144` — `background=True` alone still stops at `65,536`). Server-side
  continuation runs in 64K hops, appending a `model_output` step per hop to
  `interaction.steps`; concatenate text across all `model_output` steps when complete.
  (If running long foreground requests without `background=True`, set HTTP timeouts >= 15
  minutes.)

## Multi-model pairing

Because Carbon, Neon, Argon, and Hydrogen share the exact same request schema, 1M window,
and built-in tools, Google recommends pairing them across turns in one session:

1. **Turn 1 (`gemini-4-carbon` or `gemini-4-neon`):** Diagnose a concurrency/architecture
   bug, design interfaces, or resolve a hard roadblock.
2. **Turn 2+ (`gemini-4-hydrogen`):** Pass `previous_interaction_id` (or full unmodified
   `thought` steps in stateless mode) to Hydrogen to implement edits, fan out parallel
   worker subagents, and iterate on tests at lower latency and cost.

## Upgrade paths & migration checklist from Gemini 3 / 2.5

| Current model | Recommended target |
|---|---|
| `gemini-3.8-flash`, `gemini-3.6-flash`, `gemini-3-flash-preview` | `gemini-4-hydrogen` |
| `gemini-3.1-pro-preview` | `gemini-4-neon` (or `gemini-4-carbon` for hardest problems) |
| `gemini-2.5-pro` *(auto-routes to `3.8-flash` Dec 1, 2026)* | `gemini-4-hydrogen` or `gemini-4-carbon` |
| `gemini-2.5-flash`, `gemini-2.5-flash-lite` *(2.5-flash auto-routes to `3.5-flash-lite` Dec 1, 2026)* | `gemini-4-hydrogen` or `gemini-3.5-flash-lite` |

Checklist when moving an existing integration to Gemini 4:

1. Update `model` to `gemini-4-hydrogen`, `gemini-4-neon`, `gemini-4-carbon`, or
   `gemini-4-argon`.
2. **Remove `temperature`, `top_p`, `top_k`, and `candidate_count`** (in-range sampling
   values are ignored with deprecation headers; out-of-range values return `400`).
3. **Remove prefilled `model` turns** (`400` error) — move format/preamble rules to
   `system_instruction` or Structured Outputs.
4. **Remove explicit caching (`cached_content` / `caches.create`)** — rely on automatic
   implicit caching (`>= 4,096` tokens) by placing static instructions/tools/docs first.
5. **Set `thinking_level` explicitly** (`minimal` and `thinking_budget` fail; watch `high`
   default on Hydrogen/Neon vs `medium` on Carbon/Argon).
6. **Remove low `max_output_tokens` caps** (use default `65,536`, or `background=True` +
   `> 65536` for long decode).
7. **Declare native tools or `response_schema`** instead of raw-text JSON tool prompts.
8. **Use `previous_interaction_id`** (or preserve `thought` blocks / `thought_signature`
   unchanged; never copy a signature across parallel `functionCall` parts).
9. Account for billing on **every processed token**, including thinking tokens and internal
   grounding system instructions / function declarations. (Google's `gemini-api-dev` skill
   from `google-gemini/gemini-skills` automates SDK migrations.)
