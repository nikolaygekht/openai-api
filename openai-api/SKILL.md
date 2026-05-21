---
name: openai-api
description: "Build apps with the OpenAI API using the Responses API. TRIGGER when: code imports `openai`/`@openai/api`/`OpenAI` namespace (C#)/`com.openai.client` (Java), the user mentions OpenAI, GPT-5, gpt-5.x, o-series, Responses API, function calling on OpenAI, Whisper transcription, gpt-4o-mini-tts, OpenAI prompt caching, or asks how to call any OpenAI endpoint in any language. ALSO TRIGGER when the user has an OpenAI key and wants to do text generation, structured outputs, tool use, vision, audio transcription, or text-to-speech — even if they don't name the API explicitly. DO NOT TRIGGER when: code imports `anthropic`/`@anthropic-ai/sdk`/`claude_agent_sdk` or the user explicitly wants Claude/Anthropic, or general non-LLM programming."
---

# Building Apps with the OpenAI Responses API

This skill helps you build LLM-powered applications with OpenAI's **Responses API** (`POST /v1/responses`) — OpenAI's current primary endpoint. Choose the right model, detect the project language, then read the language-specific documentation.

## Defaults

Unless the user specifies otherwise:

- **Model**: `gpt-5.2` — the canonical example model in OpenAI's SDK READMEs (May 2026). Cheap, fast, top-tier quality. Use this instead of `gpt-5`, `gpt-4o`, or older defaults.
- **Reasoning**: omit the `reasoning` parameter — `gpt-5.2` decides on its own (default effort is `medium` for non-trivial tasks). Only set `reasoning={"effort": "..."}` when the user asks for more reasoning depth or wants to cut cost.
- **Endpoint**: **Responses API** (`client.responses.create(...)` / `/v1/responses`) — NOT Chat Completions (`/v1/chat/completions`). The Responses API is OpenAI's strategic direction; Chat Completions is legacy. Only use Chat Completions if the user explicitly asks for it or is maintaining existing code that uses it.
- **Streaming**: use streaming (`stream=True` / the `responses.stream(...)` helper) for any request that may produce long output, runs tools, or feeds a UI. It avoids HTTP timeouts and lets users see progress.
- **Output limit**: don't lowball `max_output_tokens`. For non-streaming default to ~16000, for streaming up to ~64000. The Responses API counts reasoning tokens against this limit too.

---

## Secrets & API Keys (read before writing any code)

The OpenAI API key (`sk-...`) is a production secret. Treat it like a password — even when "just getting things working." This is non-negotiable and applies in every language.

**Always:**

- **Read the key from an environment variable** (`OPENAI_API_KEY`) — never hardcode it in source files. All four official SDKs read this env var automatically when you instantiate the client with no arguments. That's the default; use it.
- **For local development**, store the key in a **`.env` file that is excluded from version control**. Before creating `.env`, make sure `.gitignore` contains `.env` (and `.env.*` for variants like `.env.local`). If `.gitignore` doesn't exist, create it. Add a `.env.example` file with the key *names* but no values, so teammates know what to set.
- **For deployment**, use the platform's secret store: Azure Key Vault, AWS Secrets Manager, GCP Secret Manager, Kubernetes Secrets, GitHub Actions repository secrets, etc. Inject the value into the runtime environment — never bake it into a container image, config file, or CI log.

**Never:**

- **Never print, log, or echo the key value** — not in console output, not in error messages, not in committed example code, not when "showing the user what was loaded." If you need to confirm the key is set, print *that* it is set (e.g., "OPENAI_API_KEY is configured") without the value, or print only `key[:7] + "..."` redacted form if you absolutely must.
- **Never hardcode** a key in code you write, even as a "placeholder you'll replace later" — that placeholder gets committed and forgotten. Use the env var read pattern from the start.
- **Never write a real-looking key value into example or test code** that gets committed. Use `os.environ["OPENAI_API_KEY"]` or the equivalent reader.
- **Never include the key in shell commands you suggest to the user** (`curl -H "Authorization: Bearer sk-..."`) unless reading it from env via `$OPENAI_API_KEY` — the value would end up in shell history.

Before adding any OpenAI-using code to a repo, verify the secret handling: check that `.gitignore` excludes `.env`, that no key value sits in a committed file, and that `client = OpenAI()` (or equivalent) is the call pattern in use.

See `shared/secrets.md` for the full pattern in each language plus `.env` / `.gitignore` templates.

---

## Language Detection

Before reading code examples, determine which language the user is working in:

1. **Look at project files** to infer the language:
   - `*.py`, `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile` → **Python** — read from `python/openai-api/`
   - `*.ts`, `*.tsx`, `package.json`, `tsconfig.json` → **TypeScript** — read from `typescript/openai-api/`
   - `*.js`, `*.jsx` (no `.ts` files present) → **TypeScript** — JS uses the same SDK, read from `typescript/openai-api/`
   - `*.cs`, `*.csproj` → **C#** — read from `csharp/openai-api.md`
   - `*.java`, `pom.xml`, `build.gradle`, `*.kt`, `*.kts` → **Java** — Kotlin uses the Java SDK, read from `java/openai-api.md`

2. **If multiple languages detected** (e.g., both Python and TypeScript files):
   - Check which language the user's current file or question relates to.
   - If still ambiguous, ask: "I see both Python and TypeScript files. Which one is using the OpenAI API?"

3. **If language can't be inferred** (empty project, no source files):
   - Use AskUserQuestion with options: Python, TypeScript, C#, Java, cURL/raw HTTP.
   - If AskUserQuestion is unavailable, default to Python and note: "Showing Python examples — say so if you need a different language."

4. **If unsupported language detected** (Go, Ruby, PHP, Rust, Swift, etc.):
   - Suggest cURL/raw HTTP examples from `curl/examples.md`. OpenAI does not publish official SDKs for these languages as of May 2026, but the REST API is the same.
   - Offer to show Python or TypeScript examples as reference implementations.

### Language-Specific Feature Support

| Language    | Native `responses` resource | Stream helper                        | Per-status exception classes |
| ----------- | --------------------------- | ------------------------------------ | ---------------------------- |
| Python      | Yes (`client.responses`)    | `responses.stream()` context manager | Yes (10 classes)             |
| TypeScript  | Yes (`client.responses`)    | `responses.stream()` emitter         | Yes (10 classes)             |
| C# (.NET)   | Yes (`ResponsesClient`) — **experimental flag `OPENAI001`** | `CreateResponseStreaming` | No — single `ClientResultException` with `.Status` |
| Java        | Yes (`client.responses()`)  | `StreamResponse<ResponseStreamEvent>` | Yes (8 service classes)      |
| cURL        | N/A — raw HTTP              | SSE stream                            | N/A                          |

---

## Which Surface Should I Use?

> **The Responses API is the default.** It is OpenAI's unified, agent-loop endpoint with built-in tools (web search, file search, code interpreter, image generation, computer use, MCP), native multimodal input, structured outputs, reasoning, and durable conversation state. Use Chat Completions only if the user is maintaining legacy code.

| Use Case                                     | Surface                          | Why                                                         |
| -------------------------------------------- | -------------------------------- | ----------------------------------------------------------- |
| Text gen, classification, extraction, Q&A    | **Responses API**                | One call, simplest                                          |
| Multi-turn conversation                      | **Responses API** + `previous_response_id` or Conversations API | Server-side state, no resending history |
| Function calling / agent loop                | **Responses API** + `tools`      | Flat tool shape; SDK runs the loop                          |
| Structured outputs (JSON schema)             | **Responses API** + `text.format` | Constrained decoding via `strict: true`                    |
| Web search, file search, code execution      | **Responses API** + built-in tools | Hosted on OpenAI's infra; no plumbing                     |
| Speech-to-text (Whisper / gpt-4o-transcribe) | `/v1/audio/transcriptions`       | Separate endpoint — see `*/audio.md`                        |
| Text-to-speech (gpt-4o-mini-tts)             | `/v1/audio/speech`               | Separate endpoint — see `*/audio.md`                        |
| Voice agent / WebRTC                         | Realtime API (out of scope)      | Different connection model — defer to OpenAI's Realtime docs |
| Embedding text for retrieval                 | `/v1/embeddings` (out of scope)  | Defer to OpenAI's embeddings docs                            |

---

## Architecture

Everything in scope goes through **`POST /v1/responses`** plus the two audio endpoints. The Responses API is itself a meta-endpoint: tools (your functions plus OpenAI-hosted ones), structured outputs, reasoning, and multimodal I/O are all features of this single endpoint.

- **Items, not messages.** Input and output are heterogeneous arrays of *items* (text messages, tool calls, tool outputs, reasoning items, MCP approvals, computer-use actions, …). Each item has a `type` discriminator. This is the biggest structural difference from Chat Completions.

- **User-defined tools.** You declare tools in a flat shape (`{type: "function", name, description, parameters, strict}`) — there is no `function: {...}` envelope as in Chat Completions. The model emits `function_call` items; you reply with `function_call_output` items keyed by `call_id`.

- **Built-in tools.** Each is one line of opt-in: `web_search`, `file_search`, `code_interpreter`, `image_generation`, `computer` / `computer_use_preview`, `mcp`, `apply_patch`, `local_shell`. Results come back as items in `output[]` plus annotations on assistant text.

- **Structured outputs.** Live under `text.format = {type: "json_schema", name, schema, strict: true}`. Old Chat Completions used `response_format` — that key does not exist on Responses.

- **Conversation state.** Three modes: pass `previous_response_id` (server reconstructs the chain), use the Conversations API (`/v1/conversations` for durable, multi-session state), or run stateless with `store=false` and `include: ["reasoning.encrypted_content"]` to manually carry reasoning forward.

---

## Current Models (cached: 2026-05-21)

| Model                | Model ID            | Context | Input $/1M | Cached $/1M | Output $/1M | Best for                          |
| -------------------- | ------------------- | ------- | ---------- | ----------- | ----------- | --------------------------------- |
| GPT-5.5 (flagship)   | `gpt-5.5`           | 400K    | $5.00      | $0.50       | $30.00      | Hardest reasoning / research      |
| GPT-5.2 (**default**) | `gpt-5.2`          | 400K    | $1.75      | $0.175      | $14.00      | General-purpose chat / agents     |
| GPT-5.2-Codex        | `gpt-5.2-codex`     | 400K    | $1.75      | $0.175      | $14.00      | Coding agents                     |
| GPT-5.4 (long ctx)   | `gpt-5.4`           | 1.05M   | $2.50      | $0.25       | $15.00      | Very long inputs (>400K tokens)*   |
| GPT-5.4-mini         | `gpt-5.4-mini`      | 1.05M   | $0.75      | —           | $4.50       | High-volume cheap                 |
| GPT-5.4-nano         | `gpt-5.4-nano`      | 1.05M   | $0.20      | —           | $1.25       | Classification, extraction, sub-agents |
| o3 (reasoning)       | `o3`                | 200K    | $2.00      | —           | $8.00       | When user explicitly wants o-series |

\* `gpt-5.4` above 272K input tokens applies a 2× input / 1.5× output surcharge.

**ALWAYS use `gpt-5.2` unless the user explicitly names a different model.** Do not downgrade to gpt-4o, gpt-4.1, or other older models for cost — that is the user's decision. If the user requests `gpt-5.5` for hard reasoning or `gpt-5.4-nano` for high volume, honor it.

**Older models** (`gpt-4o`, `gpt-4.1`, `gpt-4.1-mini`, `o4-mini`, `chatgpt-4o-latest`): retired from ChatGPT Feb 13 2026; API access has been winding down. New code should target gpt-5.x. If the user is on an older model, recommend a migration unless they have a reason to stay.

**Audio models**:
- Transcription: `gpt-4o-transcribe` (~$0.006/min). Cheaper: `gpt-4o-mini-transcribe` (~$0.003/min). Legacy: `whisper-1` (still supported, NOT deprecated). Diarization: `gpt-4o-transcribe-diarize`.
- TTS: `gpt-4o-mini-tts` (~$0.015/min, steerable via `instructions`). Legacy: `tts-1`, `tts-1-hd`.

**Live capability lookup:** The `/v1/models` endpoint **does NOT expose capabilities** (no `context_window`, `supports_vision`, etc.) — only `id`, `object`, `created`, `owned_by`. Use the table above for capability info; use `client.models.list()` only to enumerate which models the org can access. See `shared/models.md`.

---

## Reasoning & Effort (Quick Reference)

GPT-5.x and o-series models reason internally before producing output. Configure via the `reasoning` parameter on `responses.create()`:

```python
reasoning={"effort": "medium", "summary": "auto"}
```

- **`effort`**: `"none" | "minimal" | "low" | "medium" | "high" | "xhigh"`.
  - `gpt-5.1` and `gpt-5.2` default to `"none"` (no reasoning) — set explicitly for hard tasks.
  - Pre-5.1 reasoning models default to `"medium"` and do NOT support `"none"`.
  - `gpt-5-pro` is locked to `"high"`.
  - `"xhigh"` is available only on models after `gpt-5.1-codex-max`.
- **`summary`**: `"auto" | "concise" | "detailed"` — returns a human-readable summary of the reasoning chain as a `ResponseReasoningItem.summary[*].text`.
- **Reasoning tokens are billed at the output rate** but don't appear in `output_text`. Track via `response.usage.output_tokens_details.reasoning_tokens`.
- **Encrypted reasoning (stateless mode):** When `store=false`, the server cannot remember reasoning between turns. Pass `include: ["reasoning.encrypted_content"]` to get the encrypted blob back, then echo it in the next request's `input` array. Without this, multi-turn reasoning quality drops sharply.

---

## Prompt Caching (Quick Reference)

**Fully automatic** — no code changes required. Any prefix of ≥1024 tokens that you reuse within ~5–10 minutes is cached. Cached input is billed at **10% of the base input price** on gpt-5.x (a 90% discount).

**Prefix-stability rules** — any byte change anywhere in the prefix invalidates everything after it. Render order: `instructions` → `tools` → earliest `input` items. Keep stable content first (system instructions, tool definitions, retrieved docs); put volatile content (timestamps, per-request IDs, current question) LAST.

**Verify cache hits** with `response.usage.input_tokens_details.cached_tokens`. If it's zero across repeated requests, a silent invalidator is at work: timestamps in `instructions`, tool list reordering, unsorted dict keys, varying model snapshot IDs.

**Multi-tenant routing**: pass `prompt_cache_key="tenant-42-v3"` to route same-prefix requests to the same backend machine. Keep each unique `(prefix, prompt_cache_key)` pair below ~15 RPM.

**Extended retention**: set `prompt_cache_retention="24h"` to keep prefixes up to 24 hours (default is 5–10 min, up to 1h off-peak).

Full architecture and the silent-invalidator checklist: read `shared/prompt-caching.md`.

---

## Reading Guide

After detecting the language, read the relevant files based on what the user needs:

### Quick Task Reference

**Single text generation / classification / extraction / Q&A:**
→ Read only `{lang}/openai-api/README.md` (or `{lang}/openai-api.md` for C# / Java)

**Chat UI or real-time response display:**
→ Read `{lang}/openai-api/README.md` + `{lang}/openai-api/streaming.md` (Python / TS), or the streaming section of `{lang}/openai-api.md` (C# / Java)

**Function calling / tool use / agent loop:**
→ Read `{lang}/openai-api/README.md` + `shared/tool-use-concepts.md` + `{lang}/openai-api/tool-use.md` (Python / TS), or the tool section of `{lang}/openai-api.md`

**Structured outputs (JSON schema):**
→ Read `shared/structured-outputs.md` + the relevant language file

**Speech-to-text (transcription) / text-to-speech:**
→ Read `{lang}/openai-api/audio.md` (Python / TS) or the audio section of `{lang}/openai-api.md` (C# / Java)

**Prompt caching / optimize cache hit rate:**
→ Read `shared/prompt-caching.md`

**Debugging an HTTP error:**
→ Read `shared/error-codes.md`

**Latest docs / something not covered here:**
→ Read `shared/live-sources.md`

### Full File Index

**`shared/`** — language-agnostic concepts:
- `models.md` — full model catalog, pricing details, deprecation timeline.
- `prompt-caching.md` — caching architecture, invalidators, multi-tenant routing.
- `structured-outputs.md` — JSON schema mode, `strict: true` requirements, refusal handling.
- `tool-use-concepts.md` — function tools, hosted tools (web/file search, code interpreter, MCP, image generation), the agent loop.
- `error-codes.md` — HTTP status codes, retry guidance, headers, error body shape.
- `secrets.md` — API-key handling per language, `.env` / `.gitignore` setup, CI/CD pattern. **Always consult before writing code that touches `OPENAI_API_KEY`.**
- `live-sources.md` — WebFetch URLs for latest official docs (note: `platform.openai.com` returns 403 to bots — use WebSearch instead, or fetch the SDK source on GitHub).

**`python/openai-api/`** — Python (`openai` SDK ≥ 2.37):
- `README.md` — install, auth, minimal Responses call, exception classes, retries.
- `streaming.md` — `stream=True` vs. `responses.stream(...)` helper, all event types.
- `tool-use.md` — function tools (Pydantic helper), tool result handling, hosted tools.
- `audio.md` — Whisper / gpt-4o-transcribe / gpt-4o-mini-tts.

**`typescript/openai-api/`** — Node / TypeScript (`openai` SDK ≥ 6.38):
- Same four files as Python.

**`csharp/openai-api.md`** — single file for the official `OpenAI` NuGet package (v2.10+).

**`java/openai-api.md`** — single file for `com.openai:openai-java` (v4.36+).

**`curl/examples.md`** — raw HTTP for unsupported languages.

---

## When to Use WebFetch / WebSearch

Use WebSearch (NOT WebFetch — `platform.openai.com` returns 403 to bots) to get the latest documentation when:

- The user asks for "latest" or "current" information about model pricing, model availability, or new features.
- The cached model table in `shared/models.md` looks stale (more than a few months old).
- The user asks about features not covered here (e.g., Realtime API, Assistants, Batch, Embeddings).

For SDK details (method signatures, exception classes), prefer `WebFetch` against `github.com/openai/openai-{python,node,dotnet,java}` — the SDK source files are authoritative and the site does not 403.

See `shared/live-sources.md` for URL patterns.

---

## Common Pitfalls

- **Don't truncate inputs silently.** If content is too long for the context window, tell the user and discuss options (chunking, summarization, switching to `gpt-5.4` for its 1.05M window) — don't just lose tokens.
- **Don't use `response_format`.** That's Chat Completions. On Responses use `text.format = {type: "json_schema", ...}` — flat shape, no nested `json_schema` envelope.
- **Don't define `tools` with a `function: {...}` envelope.** On Responses the function fields are flat: `{type: "function", name, description, parameters, strict}` — not `{type: "function", function: {name, ...}}`.
- **Don't use `messages`.** The Responses input parameter is `input`, accepting either a plain string or an array of items. The system-role equivalent is the top-level `instructions` field.
- **Don't read `response.choices[0].message.content`.** The Responses convenience accessor is `response.output_text`. For finer-grained access iterate `response.output[]` and filter by `type`.
- **`previous_response_id` does NOT inherit `instructions`.** Restate them each turn or the persona resets.
- **Reasoning tokens count toward `max_output_tokens`.** A `gpt-5.2` request with `reasoning.effort: "high"` may emit 3–10× the visible tokens as hidden reasoning. Budget accordingly.
- **`store=false` breaks reasoning continuity.** Pair it with `include: ["reasoning.encrypted_content"]` and echo the reasoning items in the next turn's input.
- **Don't reorder or rebuild `tools` between requests in a cached prefix.** Any change invalidates the cache. Define the tool list once and reuse the same list object.
- **`prompt_cache_retention` is not a TTL extender for arbitrary use cases.** `"24h"` opts into a separate storage backend with the same 1024-token min prefix; it still invalidates on prefix change.
- **`.NET SDK is experimental for Responses.** Suppress `OPENAI001` warning. Single `ClientResultException` — no per-status subclasses; switch on `.Status`.
- **Java SDK uses `UnauthorizedException`, not `AuthenticationException`** for 401 — minor naming divergence from Python / Node.
- **TTS `input` is capped at 4096 characters.** For longer-form, split on sentence boundaries and concatenate audio.
- **Audio file uploads are capped at 25 MB.** Pre-compress to low-bitrate mono MP3 for long recordings, or split on silence.
