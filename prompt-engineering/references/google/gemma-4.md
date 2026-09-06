# Gemma 4 (local / open-weight)

Source: `ai.google.dev/gemma/docs/core/prompt-formatting-gemma4`

**One-line summary:** Gemma 4 is the one target here where *prompt formatting* — literal
control tokens, turn structure, and thought-stripping between turns — matters as much as
prompt content. Get the template right first; only then tune wording.

Sizes seen in the wild: `gemma-4-e2b` / `e4b` (edge), `gemma-4-12B-it`,
`gemma-4-26B-A4B-it` (MoE), `gemma-4-31B-it`. Text, audio, and image input; context up to
256K.

## Control tokens

Gemma 4 introduced a new token scheme; Gemma 1–3 templates do **not** transfer.

| Token | Purpose |
|---|---|
| `<\|turn>` | Begins a dialogue turn |
| `<turn\|>` | Ends a dialogue turn |
| `system` / `user` / `model` | Role, written immediately after `<\|turn>` |

```
<|turn>system
You are a helpful assistant.<turn|>
<|turn>user
Hello.<turn|>
```

Multimodal placeholders: `<|image|>` and `<|audio|>` mark where embeddings get inserted;
they're replaced by soft embeddings after tokenization.

```
<|turn>user
Describe this image: <|image|>

And translate these audio:

a. <|audio|>
b. <|audio|><turn|>
<|turn>model
```

Tool lifecycle tokens: `<|tool>`/`<tool|>` (declaration), `<|tool_call>`/`<tool_call|>`,
`<|tool_response>`/`<tool_response|>`. Note that `<|tool_response>` acts as an **additional
stop sequence** for the inference engine.

String values inside structured blocks use the delimiter token `<|"|>` — for example
`location:<|"|>London<|"|>`. This is what keeps braces, commas, and quotes inside a string
from being parsed as structure. All string literals in declarations, calls, and responses
must be wrapped this way.

Thinking mode: `<|think|>` placed in the system instruction activates it;
`<|channel>`/`<channel|>` delimit the model's internal process, and `<|channel>` is always
followed by the word "thought" when thinking is active.

```
<|turn>system
<|think|><turn|>
<|turn>user
What is the water formula?<turn|>
<|turn>model
<|channel>thought
...
<channel|>The most common interpretation of "the water formula" refers...<turn|>
```

Thinking is a conversation-level setting; consolidate it into a single system turn
alongside tool definitions and other system instructions.

## System instructions: check your runtime

Legacy Gemma (1–3) had **no separate system role** — system-level instructions went inside
the initial user prompt. Gemma 4 does have a `system` turn. Which one applies depends on
the runtime's chat template, not the model alone.

**Practical rule for LM Studio, Ollama, and llama.cpp:** verify what the loaded template
actually emits before assuming a system message is being honoured. If system instructions
appear to be ignored, prepend them to the first user turn instead — this is the single most
common cause of "the local model ignores my system prompt." When serving through an
OpenAI-compatible endpoint, the shim's template is what decides, not the API shape.

## Thought context between turns

This is the part most people get wrong, and it degrades multi-turn quality badly.

- **Standard multi-turn:** you must **strip** the model's generated thoughts from the
  previous turn before passing history back. To disable thinking mid-conversation, remove
  the `<|think|>` token when you strip.
- **Function calling is the exception:** within a single model turn involving tool calls,
  thoughts must **not** be removed between the calls.
- **Long-running agents:** because raw thoughts are stripped between turns, agents can fall
  into cyclical reasoning loops. The recommended technique is to extract, summarise, and
  feed previous thoughts back as **standard text**. Gemma 4 wasn't trained with raw
  thoughts in the prompt, so there's no expected format — use whatever suits your
  architecture.

## Steering thinking depth

Thinking is officially a boolean, but Gemma 4's instruction following is strong enough to
modulate depth with a system instruction instead of a framework parameter. Google's testing
found a "LOW thinking" system instruction cuts thinking tokens by roughly 20%.

This is explicitly a proof of concept, not a tuned feature — there's no canonical prompt.
Write your own instruction telling the model to reason efficiently at reduced depth, and
tune the depth, length, and style against your own latency/cost/quality balance.

**Known edge case:** larger models (`gemma-4-26B-A4B-it`, `gemma-4-31B-it`) sometimes
generate a thought channel even with thinking explicitly off. Stabilise by adding an empty
thinking channel to the prompt:

```
<|turn>model
<|channel>thought
<channel|>
```

The same empty-channel trick improves results when fine-tuning those two models on datasets
that contain no thinking.

## Prompting content, as opposed to format

Apply `gemini-family.md`'s core principles — Gemma shares research and technology with
Gemini — with these adjustments for a much smaller model:

- **Few-shot examples matter more, not less.** Where a frontier model can infer format from
  a description, Gemma generally needs to be shown. Keep example formatting byte-consistent.
- **One task per prompt.** Decompose aggressively; chain prompts rather than stacking
  instructions. Multi-constraint prompts degrade faster here than on hosted models.
- **Prefer structured output enforcement over asking politely.** Grammar/JSON-schema
  constrained decoding (available in llama.cpp and most local runtimes) is more reliable
  than a format instruction.
- **Expect no grounding.** There's no built-in search or code execution. Arithmetic,
  counting, and recent facts need external tools or they will be wrong.
- **Keep the context tight.** The 256K window is a ceiling, not a working target; retrieval
  quality falls off well before it, and local KV cache costs are real.
- Google publishes optimal prompt structures for ASR and speech translation on the source
  page if you're using the audio modality.

## Audio prompt templates

Speech recognition:

```
Transcribe the following speech segment in {LANGUAGE} into {LANGUAGE} text.

Follow these specific instructions for formatting the answer:
*   Only output the transcription, with no newlines.
*   When transcribing numbers, write the digits, i.e. write 1.7 and not one point seven, and write 3 instead of three.
```

Speech translation:

```
Transcribe the following speech segment in {SOURCE_LANGUAGE}, then translate it into {TARGET_LANGUAGE}.
When formatting the answer, first output the transcription in {SOURCE_LANGUAGE}, then one newline, then output the string '{TARGET_LANGUAGE}: ', then the translation in {TARGET_LANGUAGE}.
```

A plain "transcribe the audio" also works.
