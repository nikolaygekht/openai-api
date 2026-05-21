# OpenAI Model Catalog & Pricing

Cached: 2026-05-21. The OpenAI `/v1/models` endpoint does not expose capability metadata (no `context_window`, `supports_vision`, etc.) — only `id`, `object`, `created`, `owned_by` — so this static catalog is authoritative. To refresh, run a WebSearch for "OpenAI model pricing 2026" or hit the SDK source on GitHub.

---

## Choosing a Default

For new code, default to **`gpt-5.2`**. It is the example model in OpenAI's official Python and Node SDK READMEs, it gives near-frontier quality at moderate cost, and it supports the full Responses-API feature set (tools, structured outputs, vision, reasoning, prompt caching).

Switch away only when the user has a specific need:

| Need                                          | Switch to        | Why                                                       |
| --------------------------------------------- | ---------------- | --------------------------------------------------------- |
| Hardest reasoning / research-grade            | `gpt-5.5`        | OpenAI's current flagship (Apr 2026), 6× the cost          |
| High-volume cheap chat                        | `gpt-5.4-mini`   | $0.75/$4.50 per M tokens; 1.05M context                    |
| Classification, extraction, sub-agents        | `gpt-5.4-nano`   | $0.20/$1.25; API-only; cheapest GPT-5                      |
| Inputs >400K tokens (long docs, long history) | `gpt-5.4`        | 1.05M context (surcharge above 272K input)                 |
| Coding agents specifically                    | `gpt-5.2-codex`  | Trained for terminal / code tasks                          |
| User explicitly wants o-series reasoning      | `o3` or `o4-mini` | Older reasoning trace style; o4-mini still in API         |

**Never silently downgrade** — `gpt-5.2` → `gpt-5-mini` to save money is the user's call, not yours.

---

## GPT-5 Family — Full Table

| Model ID         | Context | Max Output | Input $/1M | Cached $/1M | Output $/1M | Notes                                                |
| ---------------- | ------- | ---------- | ---------- | ----------- | ----------- | ---------------------------------------------------- |
| `gpt-5.5`        | 400K    | 128K       | $5.00      | $0.50       | $30.00      | Flagship (Apr 23 2026); deepest reasoning            |
| `gpt-5.5-pro`    | 400K    | 128K       | ~$30       | —           | ~$180       | Premium reasoning tier; highest quality, slowest     |
| `gpt-5.2`        | 400K    | 128K       | $1.75      | $0.175      | $14.00      | **Default for new code**                             |
| `gpt-5.2-pro`    | 400K    | 128K       | premium    | —           | premium     | Premium tier; supports `reasoning.effort: "xhigh"`   |
| `gpt-5.2-codex`  | 400K    | 128K       | $1.75      | $0.175      | $14.00      | Coding variant (Jan 14 2026)                         |
| `gpt-5.1`        | 400K    | 128K       | ~$1.25     | —           | ~$10.00     | "Steering" update (Nov 12 2025)                      |
| `gpt-5`          | 400K    | 128K       | $1.25      | $0.125      | $10.00      | Base GPT-5 (Aug 2025)                                |
| `gpt-5.4`        | 1.05M   | 128K       | $2.50      | $0.25       | $15.00      | Long-context; 2× input + 1.5× output above 272K input |
| `gpt-5.4-mini`   | 1.05M   | 128K       | $0.75      | —           | $4.50       | Recommended cheap default                            |
| `gpt-5.4-nano`   | 1.05M   | 128K       | $0.20      | —           | $1.25       | API-only; ultra-cheap                                |
| `gpt-5-mini`     | 400K    | 128K       | $0.25      | —           | $2.00       | Older mini tier                                      |
| `gpt-5-nano`     | 400K    | 128K       | $0.05      | —           | $0.40       | Older nano tier                                      |

---

## o-Series Reasoning Models

| Model ID  | Context | Max Output | Input $/1M | Output $/1M | Notes                                              |
| --------- | ------- | ---------- | ---------- | ----------- | -------------------------------------------------- |
| `o3`      | 200K    | 100K       | $2.00      | $8.00       | Replaced o1 at 87% price cut                       |
| `o4-mini` | 200K    | 100K       | $1.10      | $4.40       | Retired from ChatGPT Feb 13 2026; still API-callable |

**Reasoning-token gotcha:** o3 and o4-mini bill internal reasoning tokens at the output rate. A typical o3 call emits 3–10× the visible output as hidden reasoning — effective cost can be 3–10× the headline output price. For new code, prefer GPT-5.x with `reasoning.effort` settings unless the user specifically wants o-series.

---

## Audio Models

### Transcription (`/v1/audio/transcriptions`)

| Model ID                    | $/min   | Notes                                                |
| --------------------------- | ------- | ---------------------------------------------------- |
| `gpt-4o-transcribe`         | $0.006  | **Recommended default**. Best WER, multilingual      |
| `gpt-4o-mini-transcribe`    | $0.003  | Cheaper variant                                      |
| `gpt-4o-transcribe-diarize` | $0.006  | Speaker diarization; requires `chunking_strategy` for >30s |
| `whisper-1`                 | $0.006  | Legacy V2 Whisper; still supported (NOT deprecated). No `stream=True`. |

### Text-to-Speech (`/v1/audio/speech`)

| Model ID           | Pricing                                                | Notes                                            |
| ------------------ | ------------------------------------------------------ | ------------------------------------------------ |
| `gpt-4o-mini-tts`  | $0.60/M text-in + $12/M audio-out (~$0.015/min)        | **Recommended default**. Steerable via `instructions` |
| `tts-1`            | $15 per 1M characters                                  | Legacy low-latency. Ignores `instructions`.      |
| `tts-1-hd`         | ~$30 per 1M characters                                 | Legacy HD. Ignores `instructions`.               |

Voices (typed literal in SDK): `alloy`, `ash`, `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`. `marin` and `cedar` are newest (Aug 2025) and recommended for quality. `fable`, `nova`, `onyx` no longer in the typed literal but still accepted as raw strings by `tts-1` / `tts-1-hd`.

---

## Embedding Models (out of scope but commonly asked about)

| Model ID                  | Dimensions | $/1M tokens | Notes                              |
| ------------------------- | ---------- | ----------- | ---------------------------------- |
| `text-embedding-3-small`  | 1536       | $0.02       | Best value default                 |
| `text-embedding-3-large`  | 3072       | $0.13       | Marginal quality lift              |

Both support Matryoshka — request a lower-dim output to save space at minimal quality loss.

---

## Image Models (available as Responses-API built-in tool)

| Model ID            | Pricing            | Notes                                                  |
| ------------------- | ------------------ | ------------------------------------------------------ |
| `gpt-image-1.5`     | $0.009–$0.20/image | **Current flagship** (since Mar 22 2026). Use via Responses `image_generation` tool. |
| `gpt-image-1`       | $0.011–$0.25/image | Predecessor, still available                            |
| `gpt-image-1-mini`  | $0.005–$0.052/image | Budget option                                          |
| `dall-e-3`          | $0.04 / $0.08      | Marked deprecated, still callable                       |

---

## Deprecation Timeline (as of May 2026)

- **gpt-4o, gpt-4.1, gpt-4.1-mini, o4-mini**: Retired from ChatGPT Feb 13 2026. API access winding down — recommend migration.
- **chatgpt-4o-latest** (API): Retired Feb 16 2026; 3-month transition window.
- **gpt-4o in Custom GPTs** (Business/Enterprise/Edu): Available until Apr 3 2026.
- **dall-e-3, dall-e-2**: Marked deprecated; new code should target `gpt-image-1.5`.
- **whisper-1**: NOT deprecated. Still supported, feature-frozen. New code should prefer `gpt-4o-transcribe`.

---

## Live Model Discovery

The `/v1/models` endpoint returns only basic metadata:

```python
m = client.models.retrieve("gpt-5.2")
# m.id = "gpt-5.2"
# m.object = "model"
# m.created = 1733923200
# m.owned_by = "openai"
```

There are no capability flags, no context_window, no modalities. **Implication:** maintain a static catalog (this file). For "is this model available to my org?" use:

```python
for m in client.models.list():
    print(m.id)
```

---

## When the User Asks About Pricing

- Always cite per-million-token rates explicitly, with both input and output.
- For cached prefixes, mention the 90% discount on gpt-5.x (e.g., "gpt-5.2 input drops from $1.75/M to $0.175/M on cache hits").
- For reasoning-heavy workloads (o-series or `reasoning.effort: "high"`), warn that the *effective* output cost is multiples of the headline number because reasoning tokens bill at output rate.
- For very long inputs (>272K tokens on `gpt-5.4`), warn about the 2× input / 1.5× output surcharge.
- For batch / async workloads, the Batch API gives a 50% discount (but is out of scope for this skill).
