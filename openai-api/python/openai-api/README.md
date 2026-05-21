# Python — OpenAI Responses API

Official SDK: `openai` (≥ 2.37 as of May 2026). Requires Python ≥ 3.9.

This file covers installation, authentication, the minimal Responses call, async, errors, retries. For streaming see `streaming.md`. For tools see `tool-use.md`. For audio see `audio.md`. For secret handling see `../../shared/secrets.md` — read that BEFORE you write key-loading code.

---

## Install

```bash
pip install openai
# optional:
pip install python-dotenv         # for loading .env in local dev
pip install "openai[aiohttp]"     # AsyncOpenAI with aiohttp transport (faster than httpx for some workloads)
```

---

## Authentication

The SDK reads `OPENAI_API_KEY` from the environment automatically. **That is the right pattern.** Hardcoding the key, printing it, or committing it is never acceptable — see `../../shared/secrets.md`.

```python
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()                  # local dev only — loads .env into os.environ

client = OpenAI()              # picks up OPENAI_API_KEY, OPENAI_ORG_ID, OPENAI_PROJECT_ID, OPENAI_BASE_URL
```

Optional env vars the SDK respects:

- `OPENAI_API_KEY` — required.
- `OPENAI_ORG_ID` / `OPENAI_PROJECT_ID` — optional org / project routing.
- `OPENAI_BASE_URL` — override the API host (for Azure, on-prem proxies, etc.).
- `OPENAI_LOG` — `info` or `debug` for SDK logging.

For Azure OpenAI, instead use `AzureOpenAI`:

```python
from openai import AzureOpenAI
client = AzureOpenAI(
    api_version="2026-01-01",
    azure_endpoint="https://your-resource.openai.azure.com/",
    # AZURE_OPENAI_API_KEY also picked up from env
)
```

---

## Minimal Responses Call

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5.2",
    instructions="You are a coding assistant that talks like a pirate.",
    input="How do I check if a Python object is an instance of a class?",
)
print(response.output_text)
```

Key things:

- **`instructions`** is the top-level system message. Don't fake it via a `{"role":"system", "content":...}` item — use this field. It also doesn't carry across `previous_response_id` chains, so set it on every call.
- **`input`** can be a plain string (single user message) or an array of items (see below).
- **`response.output_text`** is a convenience accessor that concatenates all `output_text` content parts across `response.output[]`. For most chat-style use cases this is all you need.

### Multi-modal input

```python
response = client.responses.create(
    model="gpt-5.2",
    input=[{
        "role": "user",
        "content": [
            {"type": "input_text", "text": "What's in this image?"},
            {"type": "input_image", "image_url": "https://example.com/cat.jpg"},
        ],
    }],
)
```

Image content options:
- `image_url` — public URL or `data:` base64 URL.
- `file_id` — id of a previously uploaded file.
- `detail` — `"low" | "high" | "auto" | "original"` (controls how the image is downsampled).

### Multi-turn conversation

Three ways, in increasing order of "OpenAI manages state":

**a) `previous_response_id` (recommended for chat UIs)**

```python
resp1 = client.responses.create(model="gpt-5.2", input="Hi, my name is Sam.")
resp2 = client.responses.create(
    model="gpt-5.2",
    previous_response_id=resp1.id,
    input="What did I just tell you?",
)
```

Server reconstructs the chain. Cheap (cached prefix), simple, default mode. Remember to re-supply `instructions` if you set them — they don't inherit.

**b) Conversations API (for durable multi-session state)**

```python
conv = client.conversations.create()
client.responses.create(model="gpt-5.2", conversation=conv.id, input="...")
# Later:
client.responses.create(model="gpt-5.2", conversation=conv.id, input="...")
```

You can list and delete past items via `client.conversations.items.*`.

**c) Stateless (`store=False`) for ZDR / regulated environments**

```python
client.responses.create(
    model="gpt-5.2",
    input=full_history_items,
    store=False,
    include=["reasoning.encrypted_content"],   # carry reasoning forward
)
```

You manage the full input array each turn and echo any `ResponseReasoningItem` (with `encrypted_content`) back to preserve chain-of-thought across turns.

---

## Async

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def main():
    resp = await client.responses.create(model="gpt-5.2", input="Hello")
    print(resp.output_text)

asyncio.run(main())
```

For high-throughput workloads with aiohttp backend:

```python
from openai import AsyncOpenAI, DefaultAioHttpClient

async with AsyncOpenAI(http_client=DefaultAioHttpClient()) as client:
    resp = await client.responses.create(model="gpt-5.2", input="...")
```

---

## Resource Methods

```python
client.responses.create(**params)                  # → Response
client.responses.retrieve(response_id)             # → Response
client.responses.delete(response_id)               # → None
client.responses.cancel(response_id)               # → Response
client.responses.compact(**params)                 # → CompactedResponse (summarize a long chain)
client.responses.stream(**params)                  # context manager for streaming (see streaming.md)
client.responses.parse(**params)                   # structured-output helper (see structured outputs)
client.responses.input_items.list(response_id)     # paginate items in a response
client.responses.input_tokens.count(**params)      # pre-flight token count
```

---

## Counting Tokens Before Sending

```python
count = client.responses.input_tokens.count(
    model="gpt-5.2",
    input=long_input,
)
print(count.input_tokens)
```

Useful when you need to fit into a budget or check cache feasibility (cached prefixes ≥ 1024 tokens).

---

## Errors

The SDK raises typed exceptions. Catch by class:

```python
from openai import (
    OpenAIError,         # base
    APIError,            # any API-side error
    APIStatusError,      # 4xx / 5xx with response
    BadRequestError,     # 400
    AuthenticationError, # 401
    PermissionDeniedError, # 403
    NotFoundError,       # 404
    ConflictError,       # 409
    UnprocessableEntityError, # 422
    RateLimitError,      # 429
    InternalServerError, # 5xx
    APIConnectionError,  # network errors
    APITimeoutError,     # timeout
    LengthFinishReasonError,
    ContentFilterFinishReasonError,
)

try:
    client.responses.create(model="gpt-5.2", input="...")
except RateLimitError as e:
    print("rate limited:", e.request_id, e.response.headers.get("retry-after"))
except APIStatusError as e:
    print(f"{e.status_code} request_id={e.request_id}: {e.body}")
except APIConnectionError as e:
    print("network problem:", e)
```

`APIStatusError` exposes `status_code`, `response`, `request_id`, `body`. Every retryable failure includes `request_id` — log it for OpenAI support tickets.

For HTTP-level details (status codes, headers, retry strategy), see `../../shared/error-codes.md`.

---

## Retries & Timeouts

Defaults: **2 retries**, **600 s timeout**. Retries fire on connection errors plus 408 / 409 / 429 / ≥500.

Override per client:

```python
client = OpenAI(max_retries=5, timeout=30.0)
```

Override per request:

```python
client.with_options(max_retries=0, timeout=5.0).responses.create(...)
```

To disable built-in retries and roll your own (e.g., for cleaner observability):

```python
client = OpenAI(max_retries=0)
# then handle RateLimitError / APIConnectionError / InternalServerError manually
```

See `../../shared/error-codes.md` for a canonical backoff implementation.

---

## Reading Headers

For success — use the `with_raw_response` accessor:

```python
raw = client.responses.with_raw_response.create(model="gpt-5.2", input="hi")
print(raw.headers["x-request-id"])
resp = raw.parse()                       # the typed Response object
```

For failures — the exception's `.response` attribute exposes headers:

```python
try:
    client.responses.create(model="gpt-5.2", input="hi")
except APIStatusError as e:
    print(e.response.headers.get("x-ratelimit-remaining-requests"))
```

---

## Usage & Caching

Every response carries usage in a consistent shape:

```python
resp = client.responses.create(model="gpt-5.2", input="...")

resp.usage.input_tokens                              # total input
resp.usage.input_tokens_details.cached_tokens        # cached portion (NOT prompt_tokens_details — that's Chat Completions)
resp.usage.output_tokens                             # visible output
resp.usage.output_tokens_details.reasoning_tokens    # hidden reasoning (billed at output rate)
resp.usage.total_tokens
```

For cache strategy and the silent-invalidator checklist see `../../shared/prompt-caching.md`.

---

## Reasoning Models (gpt-5.x, o-series)

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="Prove that there are infinitely many primes.",
    reasoning={"effort": "high", "summary": "auto"},
)
print(resp.output_text)
# Reasoning items with summaries are also in resp.output:
for item in resp.output:
    if item.type == "reasoning":
        for s in item.summary:
            print(s.text)
```

`reasoning.effort`: `"none" | "minimal" | "low" | "medium" | "high" | "xhigh"`.
- `gpt-5.1` / `gpt-5.2` default to `"none"`; bump up explicitly for hard tasks.
- Pre-5.1 reasoning models default to `"medium"` and don't support `"none"`.
- `gpt-5-pro` is locked to `"high"`.
- `"xhigh"` available only on models after `gpt-5.1-codex-max`.

For stateless mode (`store=False`), see "Multi-turn" section above — encrypted reasoning is required to chain across turns.

---

## Common Pitfalls (Python-specific)

- **Don't `print()` the API key.** See `../../shared/secrets.md`. The default `OpenAI()` constructor reads from env; don't override unless you have a reason.
- **`output_text` vs. `output`.** `output_text` is the convenience accessor — concatenated assistant text. For tool calls, reasoning items, refusals, citations, iterate `response.output[]` and dispatch on `item.type`.
- **Parse `arguments` with `json.loads`.** Tool-call `arguments` is a JSON string, not a dict. Always parse — never string-match.
- **Don't reorder `tools` between requests.** Cache invalidation. Define the tools list once at module scope.
- **`previous_response_id` requires `store=True`** (the default). If you set `store=False` you must pass the full input each turn.
- **The `temperature` parameter is ignored by gpt-5.x and o-series.** Reasoning models calibrate their own sampling.
- **Long inputs:** if you hit `context_length_exceeded` (400), switch to `gpt-5.4` for its 1.05M window, or summarize. Don't truncate silently.
- **Don't reach for `chat.completions`** unless the user has existing code that uses it. The Responses API is the right surface for new code.
