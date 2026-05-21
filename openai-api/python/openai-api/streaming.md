# Python — Streaming (Responses API)

Streaming sends results back as a sequence of server-sent events instead of one big response. Use it for any UI that displays output incrementally, for any request that could exceed the SDK's HTTP timeout (~10 min), or when you want to react to tool calls or reasoning summaries as they happen.

Two patterns: raw event iteration via `stream=True`, or the helper context manager `client.responses.stream(...)`. Prefer the helper for typed events and a clean `get_final_response()` — but raw iteration is sometimes simpler for tiny scripts.

---

## Helper: `client.responses.stream(...)` (recommended)

```python
from openai import OpenAI

client = OpenAI()

with client.responses.stream(
    model="gpt-5.2",
    input="Write a one-sentence bedtime story about a unicorn.",
) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)
    print()
    final = stream.get_final_response()
    print(f"\n[{final.usage.output_tokens} output tokens]")
```

The context manager:
- Reads SSE chunks and yields typed events.
- Buffers a `final_response` you can grab with `.get_final_response()` once iteration completes.
- Closes the HTTP connection when the `with` block exits (even on exceptions).

### With structured outputs

```python
from pydantic import BaseModel
from typing import List

class Step(BaseModel):
    explanation: str
    output: str

class MathResponse(BaseModel):
    steps: List[Step]
    final_answer: str

with client.responses.stream(
    model="gpt-5.2",
    input="solve 8x + 31 = 2",
    text_format=MathResponse,           # auto-derives the json_schema
) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)
    final = stream.get_final_response()
    # parsed result on the typed Pydantic instance:
    for output in final.output:
        if output.type == "message":
            for content in output.content:
                if content.type == "output_text" and content.parsed:
                    print(f"\n\nAnswer: {content.parsed.final_answer}")
```

`text_format=PydanticModel` is the streaming equivalent of `client.responses.parse(...)`.

### With tools

```python
import openai
from pydantic import BaseModel

class Query(BaseModel):
    table_name: str
    columns: list[str]

with client.responses.stream(
    model="gpt-5.2",
    input="Look up all my orders from last week.",
    tools=[openai.pydantic_function_tool(Query)],
) as stream:
    for event in stream:
        if event.type == "response.function_call_arguments.delta":
            print(event.delta, end="", flush=True)
```

See `tool-use.md` for the full tool result handling pattern.

---

## Raw Iteration: `stream=True`

```python
stream = client.responses.create(
    model="gpt-5.2",
    input="Write a haiku about caching.",
    stream=True,
)
for event in stream:
    if event.type == "response.output_text.delta":
        print(event.delta, end="", flush=True)
print()
```

This is just `stream=True` on the regular `.create()` call — no context manager. **Always iterate to completion** or call `stream.close()` — otherwise the HTTP connection leaks.

For async:

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def main():
    stream = await client.responses.create(model="gpt-5.2", input="...", stream=True)
    async for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)

asyncio.run(main())
```

---

## Event Catalog

Every streaming event has a `type` field and a `sequence_number`. Below are the events you'll commonly handle, organized by category.

### Lifecycle (one each per response)

| Type                       | Carries                                | When                                                |
| -------------------------- | -------------------------------------- | --------------------------------------------------- |
| `response.created`         | `response` (no output yet)             | First event; has the `id` so you can resume         |
| `response.in_progress`     | `response`                             | Server-side work started                            |
| `response.queued`          | `response`                             | Background mode — waiting for capacity              |
| `response.completed`       | `response` (full output)               | Last event on success — capture final usage here    |
| `response.failed`          | `response.error`                       | Last event on failure                               |
| `response.incomplete`      | `response.incomplete_details`          | Truncated (hit `max_output_tokens` etc.)            |
| `error`                    | error info                             | Out-of-band error                                   |

### Item-level

| Type                            | Carries                       | When                                  |
| ------------------------------- | ----------------------------- | ------------------------------------- |
| `response.output_item.added`    | `item`, `output_index`        | New message / tool_call / reasoning item appears |
| `response.output_item.done`     | `item`, `output_index`        | An item is complete                   |
| `response.content_part.added`   | `part`, `content_index`       | A content part inside a message appears |
| `response.content_part.done`    | `part`, `content_index`       | Content part complete                 |

### Text deltas

| Type                                       | Carries                                                            |
| ------------------------------------------ | ------------------------------------------------------------------ |
| `response.output_text.delta`               | `delta: str`, `item_id`, `output_index`, `content_index`, `logprobs` |
| `response.output_text.done`                | `text: str` (full content of the part)                             |
| `response.output_text.annotation.added`    | `annotation` (citation, file path, URL)                            |
| `response.refusal.delta`                   | `delta: str`                                                       |
| `response.refusal.done`                    | `refusal: str`                                                     |

### Reasoning deltas (gpt-5 / o-series)

| Type                                       | Carries                |
| ------------------------------------------ | ---------------------- |
| `response.reasoning_text.delta`            | `delta: str`           |
| `response.reasoning_text.done`             | `text: str`            |
| `response.reasoning_summary_part.added`    | `part`                 |
| `response.reasoning_summary_part.done`     | `part`                 |
| `response.reasoning_summary_text.delta`    | `delta: str`           |
| `response.reasoning_summary_text.done`     | `text: str`            |

### Function-tool argument deltas

| Type                                            | Carries                                |
| ----------------------------------------------- | -------------------------------------- |
| `response.function_call_arguments.delta`        | `delta: str` (JSON fragment), `item_id` |
| `response.function_call_arguments.done`         | `arguments: str` (full JSON)           |

The JSON deltas are partial strings — you can't reliably `json.loads` them mid-stream. Wait for `.done` and parse the full payload.

### Custom-tool deltas (free-form non-JSON tools)

| Type                                            | Carries        |
| ----------------------------------------------- | -------------- |
| `response.custom_tool_call_input.delta`         | `delta: str`   |
| `response.custom_tool_call_input.done`          | `input: str`   |

### Built-in tool lifecycle

For each built-in tool, you get `*.in_progress` → `*.<verb>` → `*.completed`:

| Tool             | Events                                                          |
| ---------------- | --------------------------------------------------------------- |
| `file_search`    | `.in_progress`, `.searching`, `.completed`                      |
| `web_search`     | `.in_progress`, `.searching`, `.completed`                      |
| `code_interpreter` | `.in_progress`, `.interpreting`, `.completed` + `.code.delta`, `.code.done` (Python source as written) |
| `image_generation` | `.in_progress`, `.generating`, `.completed`, `.partial_image` (carries `partial_image_b64`) |
| `mcp`            | `.in_progress`, `.completed`, `.failed`, `.arguments.delta`, `.arguments.done` |
| `mcp_list_tools` | `.in_progress`, `.completed`, `.failed`                         |

### Audio (Realtime-style)

| Type                              | Carries     |
| --------------------------------- | ----------- |
| `response.audio.delta`            | `delta` (binary chunk) |
| `response.audio.done`             |              |
| `response.audio_transcript.delta` | `delta: str` |
| `response.audio_transcript.done`  | `transcript: str` |

(These appear in Realtime / WebSocket-style usage; not in standard `responses.create`.)

---

## Pattern: Stream Text + Detect Tool Calls

```python
with client.responses.stream(model="gpt-5.2", input=..., tools=tools) as stream:
    for event in stream:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)
        elif event.type == "response.output_item.added" and event.item.type == "function_call":
            print(f"\n[calling {event.item.name}...]")
        elif event.type == "response.function_call_arguments.done":
            print(f"\n[args: {event.arguments}]")

    final = stream.get_final_response()
    calls = [it for it in final.output if it.type == "function_call"]
```

---

## Pattern: Background + Resume

For very long generations or batch UI, run in background and resume from any sequence number:

```python
# Start in background:
resp = client.responses.create(
    model="gpt-5.2",
    input=very_long_task,
    background=True,
    stream=True,
)
response_id = None
for event in resp:
    if event.type == "response.created":
        response_id = event.response.id
    # Display first 10 events, then disconnect
    if event.sequence_number >= 10:
        break

# Later (could be a different process):
resumed = client.responses.retrieve(
    response_id,
    stream=True,
    starting_after=10,
)
for event in resumed:
    print(event)
```

This is the only pattern that survives a process crash mid-stream — `responses.create(stream=True)` without `background=True` cannot be resumed.

---

## Common Pitfalls

- **`response.output_text` is empty during streaming.** Build the text from `response.output_text.delta` events. The final value is populated on `response.completed`.
- **Always close the stream.** Use the context manager (`with ... as stream:`) or call `stream.close()` in a `finally`. Leaking connections starves your worker.
- **Don't `json.loads` partial argument deltas.** Wait for `.done` events. Even valid-looking JSON fragments may be parts of larger structures.
- **Audio events only appear in Realtime / WebSocket mode** (not standard Responses). For ordinary text generation, ignore them.
- **Stream events have order, not nesting.** `response.output_item.added` for item A can be followed by item B's events, then a `response.output_text.delta` for item A again — interleaved. Use `item_id` to disambiguate when you care.
