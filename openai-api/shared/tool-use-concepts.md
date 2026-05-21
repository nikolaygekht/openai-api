# Tool Use (Responses API) — Concepts

The Responses API mixes **user-defined function tools** (your code) and **OpenAI-hosted built-in tools** (`web_search`, `file_search`, `code_interpreter`, `image_generation`, `computer`, `mcp`, `apply_patch`, `local_shell`) in the same `tools` array. The model decides which to call.

This file covers the conceptual model. For language-specific code see `{lang}/openai-api/tool-use.md` (Python / TS) or the tool section of `{lang}/openai-api.md` (C# / Java).

---

## The Agent Loop

The pattern is the same across all four SDKs:

1. **You call `responses.create(input=..., tools=[...])`.**
2. **The model may emit `function_call` items** in the response's `output[]` array, alongside (or instead of) assistant text.
3. **You execute each function call** locally (your code, your data) and produce a result.
4. **You call `responses.create` again**, passing `previous_response_id=<prior>` and `input=[function_call_output items]`. The model resumes.
5. **Repeat until** the model stops emitting tool calls and just returns text.

```
┌──────────────────┐
│ responses.create │◄────────────────────┐
│ (initial input)  │                     │
└────────┬─────────┘                     │
         │                               │
         ▼                               │
┌──────────────────┐                     │
│ Response with    │ has function_call?  │
│  output[]        ├──── yes ──┐         │
└────────┬─────────┘           │         │
         │ no                  ▼         │
         │           ┌─────────────────┐ │
         │           │ Execute fn      │ │
         │           │ locally         │ │
         ▼           └────────┬────────┘ │
   ┌──────────┐               │          │
   │  Done.   │               ▼          │
   │ output_  │     ┌──────────────────┐ │
   │ text=... │     │ Build function_  │ │
   └──────────┘     │ call_output item │ │
                    └────────┬─────────┘ │
                             │           │
                             └───────────┘
```

For built-in tools, OpenAI executes them on its own infrastructure — you don't see step 3 for those, just the result items in `output[]`.

---

## Function Tool Shape

The function tool is **flat** — no `function: {...}` envelope as in Chat Completions:

```python
{
    "type": "function",                    # literal
    "name": "get_sku_inventory",           # required, [a-zA-Z0-9_-]{,64}
    "description": "Return inventory details for a SKU.",
    "parameters": {                        # JSON Schema
        "type": "object",
        "properties": {
            "sku": {"type": "string", "description": "..."},
        },
        "required": ["sku"],
        "additionalProperties": False,
    },
    "strict": True,                        # default; enables constrained decoding
}
```

When `strict: True`, the `parameters` schema must follow the same subset rules as structured outputs (see `shared/structured-outputs.md`). The model's `arguments` are then guaranteed to validate.

---

## How Tool Calls Appear in the Response

When the model decides to call a function, the response's `output[]` array contains a `function_call` item:

```json
{
    "type": "function_call",
    "id": "fc_abc",
    "call_id": "call_xyz",
    "name": "get_sku_inventory",
    "arguments": "{\"sku\":\"sku-froge-lily-pad-deluxe\"}",
    "status": "completed"
}
```

- **`call_id`** — opaque correlation id. Use this — NOT `id` — to match your function_call_output later.
- **`arguments`** — a JSON **string**, not an object. Parse with `json.loads()` / `JSON.parse()`. Status can be `in_progress`, `completed`, or `incomplete`.

The same response may contain multiple `function_call` items (parallel tool calls — enabled by default via `parallel_tool_calls=True`), an assistant `message`, reasoning items, or built-in tool call items, all interleaved.

---

## Sending Results Back

Reply with a `function_call_output` input item, keyed by `call_id`:

```python
function_call_output = {
    "type": "function_call_output",
    "call_id": call.call_id,
    "output": json.dumps({"in_stock": 7, "price_usd": 12.99}),  # STRING
}

resp2 = client.responses.create(
    model="gpt-5.2",
    previous_response_id=resp1.id,
    input=[function_call_output],
)
```

- **`output`** is a string. Convention is JSON-encoded, but it can also be a content-parts list with text / images / files — useful for returning a screenshot to a vision-capable model, or a PDF to file-search.
- **`previous_response_id`** chains to the original request so the server has the full conversation. If you're running stateless (`store=false`), instead append the function_call item AND your function_call_output item to the next request's `input` array.

---

## Forcing or Suppressing Tool Calls

`tool_choice` controls when the model calls tools:

```python
tool_choice="none"      # never call a tool; respond with text
tool_choice="auto"      # default — model decides
tool_choice="required"  # must call at least one tool

tool_choice={"type": "function", "name": "get_sku_inventory"}  # call this specific function
```

A common pattern: on the **final follow-up** after your function executes, set `tool_choice="none"` to force the model to produce a synthesis text rather than calling more tools.

`parallel_tool_calls=False` disables parallel calls — the model emits one tool call at a time. Useful when calls have side effects you need to serialize, or when calls depend on each other's results.

---

## Built-In Tools — Quick Catalog

Each is one line of opt-in in the `tools` array. The model decides when to call them; OpenAI runs them on its infra and returns the result.

### `web_search`

```python
{
    "type": "web_search",
    "search_context_size": "medium",      # "low" | "medium" | "high"
    "user_location": {                    # optional
        "type": "approximate",
        "city": "San Francisco",
        "region": "California",
        "country": "US",                  # ISO-2
        "timezone": "America/Los_Angeles",
    },
    "filters": {"allowed_domains": ["pubmed.ncbi.nlm.nih.gov"]},
}
```

Produces `web_search_call` items in `output[]` plus `url_citation` annotations on assistant text. Add `include=["web_search_call.results", "web_search_call.action.sources"]` on the request to receive the raw results.

### `file_search`

```python
{
    "type": "file_search",
    "vector_store_ids": ["vs_abc"],       # required
    "max_num_results": 5,                 # 1..50
    "filters": {...},                     # metadata filter
    "ranking_options": {
        "ranker": "auto",
        "score_threshold": 0.5,
    },
}
```

Produces `file_search_call` items; assistant text gets `file_citation` annotations. Add `include=["file_search_call.results"]` to see chunks.

### `code_interpreter`

```python
{"type": "code_interpreter", "container": {"type": "auto"}}
```

Runs Python in a sandbox; preinstalled libs include `pandas`, `numpy`, `matplotlib`, `pillow`, `python-docx`, `python-pptx`, `pypdf`. Streams Python source via `response.code_interpreter_call_code.delta` events. Returns generated files / images. Add `include=["code_interpreter_call.outputs"]` to receive outputs in the response body.

### `image_generation`

```python
{
    "type": "image_generation",
    "model": "gpt-image-1.5",
    "size": "1024x1024",
    "quality": "high",
    "partial_images": 2,                  # 0..3, enables streaming preview
}
```

Produces `image_generation_call` items. During streaming, `response.image_generation_call.partial_image` events carry `partial_image_b64`.

### `computer` / `computer_use_preview`

```python
{"type": "computer"}
```

Controls a virtual computer (clicks, types, scrolls, screenshots). The model emits actions; your code must execute them (via a sandboxed VM you provide) and feed back screenshots as `ComputerCallOutput` items. Higher-touch than other built-ins.

### `mcp` (remote Model Context Protocol)

```python
{
    "type": "mcp",
    "server_label": "myserver",
    "server_url": "https://my-mcp.example.com/sse",
    "require_approval": "never",          # "always" | "never" | {tool filters}
    "allowed_tools": ["search", "create"],
    "headers": {"Authorization": "Bearer ..."},
}
```

Connects the model to an external MCP server. Default `require_approval="always"` — each call requires your code to reply with an `McpApprovalResponse` input item. Set `"never"` to auto-approve.

### `apply_patch`

```python
{"type": "apply_patch"}
```

The model emits unified-diff patches and OpenAI applies them. Useful for code-edit workflows. Outputs `apply_patch_tool_call` items.

### `local_shell` / `function_shell`

Shell commands in the model's local environment (`local_shell`) or your declared shell function (`function_shell`). Higher-risk; only enable when needed.

---

## Mixing User Functions and Built-In Tools

```python
tools = [
    # OpenAI-hosted
    {"type": "web_search"},
    {"type": "code_interpreter", "container": {"type": "auto"}},
    # Your function
    {"type": "function", "name": "search_inventory", "parameters": {...}, "strict": True},
]
```

The model will pick whichever is most appropriate. Your code only needs to execute your own function calls — the built-ins run themselves.

---

## Streaming Tool Calls

During `stream=True`:

1. `response.output_item.added` — a new `function_call` item appears in `output[]` (arguments are empty so far).
2. A series of `response.function_call_arguments.delta` events stream the JSON arguments as a string, fragment by fragment.
3. `response.function_call_arguments.done` — full arguments string delivered.
4. `response.output_item.done` — the function_call item is complete.

You can start parsing partial JSON if you want progress UI, but most code waits for `.done` and parses the final string with `json.loads()`.

For built-in tools, you also see per-tool lifecycle events (e.g., `response.web_search_call.in_progress`, `.searching`, `.completed`).

---

## Common Patterns

### Parallel tool calls

The model may emit multiple `function_call` items in a single response. Execute them in parallel locally, then send all `function_call_output` items in the next request's `input`.

```python
calls = [item for item in resp.output if item.type == "function_call"]
outputs = await asyncio.gather(*[execute(c) for c in calls])

next_inputs = [
    {"type": "function_call_output", "call_id": c.call_id, "output": json.dumps(o)}
    for c, o in zip(calls, outputs)
]
resp2 = client.responses.create(
    model="gpt-5.2",
    previous_response_id=resp.id,
    input=next_inputs,
)
```

### Tool-result chaining without `previous_response_id`

For stateless (`store=false`) usage, build the full input array each turn:

```python
input_items = [
    {"role": "user", "content": "..."},
    # ... prior assistant response items, including reasoning items with encrypted_content
    {"type": "function_call", "call_id": "call_xyz", ...},   # echo back
    {"type": "function_call_output", "call_id": "call_xyz", "output": "..."},
]
client.responses.create(model="gpt-5.2", input=input_items, store=False)
```

### Final synthesis turn

After tool results have come back, force a text-only synthesis:

```python
final = client.responses.create(
    model="gpt-5.2",
    previous_response_id=resp_with_tool_results.id,
    tool_choice="none",   # no more tool calls
    input=[],             # nothing new to add
)
print(final.output_text)
```
