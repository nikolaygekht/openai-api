# Python — Tool Use (Responses API)

This file covers function tools (your code) and built-in OpenAI-hosted tools (`web_search`, `file_search`, `code_interpreter`, `image_generation`, `computer`, `mcp`, `apply_patch`) — all declared in the same `tools` array.

For conceptual background read `../../shared/tool-use-concepts.md` first.

---

## Function Tools — Declaration

The function tool is **flat** — there is no `function: {...}` wrapper as in Chat Completions.

```python
TOOLS = [
    {
        "type": "function",
        "name": "get_sku_inventory",
        "description": "Return inventory details for a SKU.",
        "parameters": {
            "type": "object",
            "properties": {
                "sku": {"type": "string", "description": "Stock-keeping unit identifier"},
            },
            "required": ["sku"],
            "additionalProperties": False,
        },
        "strict": True,
    },
]
```

**Define `TOOLS` at module scope.** Rebuilding the list per request invalidates the prompt cache. Same for `instructions` — see `../../shared/prompt-caching.md`.

### Auto-derive from Pydantic

The SDK helper builds a strict-mode schema from a Pydantic class:

```python
import openai
from pydantic import BaseModel, Field

class GetSkuInventory(BaseModel):
    """Return inventory details for a SKU."""
    sku: str = Field(description="Stock-keeping unit identifier")

tools = [openai.pydantic_function_tool(GetSkuInventory)]
```

The helper:
- Uses the class name (snake-cased) as the tool name.
- Uses the docstring as `description`.
- Generates a `strict: True` schema from the fields.
- Field `description=` annotations become parameter descriptions.

You can override the name explicitly: `openai.pydantic_function_tool(GetSkuInventory, name="get_inventory")`.

---

## The Agent Loop — Minimal Manual Version

```python
import json
from openai import OpenAI

client = OpenAI()

def execute_tool(name: str, arguments: dict) -> dict:
    if name == "get_sku_inventory":
        return {"sku": arguments["sku"], "in_stock": 42, "price_usd": 12.99}
    raise ValueError(f"unknown tool {name}")

resp = client.responses.create(
    model="gpt-5.2",
    instructions="You are a helpful inventory assistant.",
    input="What's the price of sku-froge-lily-pad-deluxe?",
    tools=TOOLS,
)

# Loop until the model stops calling tools
while any(item.type == "function_call" for item in resp.output):
    tool_outputs = []
    for item in resp.output:
        if item.type == "function_call":
            result = execute_tool(item.name, json.loads(item.arguments))
            tool_outputs.append({
                "type": "function_call_output",
                "call_id": item.call_id,
                "output": json.dumps(result),
            })

    resp = client.responses.create(
        model="gpt-5.2",
        previous_response_id=resp.id,
        input=tool_outputs,
    )

print(resp.output_text)
```

Notes:

- **Always parse `arguments` with `json.loads`** — it's a JSON string, not a dict.
- **Correlate by `call_id`**, not `item.id`. The model uses `call_id` to match outputs to calls.
- **`output` in the function_call_output must be a string.** Convention is JSON-encoded.
- **Use `previous_response_id`** so OpenAI reconstructs the chain and the cached prefix stays warm.
- The loop terminates naturally when the response has no `function_call` items.

---

## Parallel Tool Calls

The model emits multiple function_call items by default (`parallel_tool_calls=True`). Run them in parallel:

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def execute_tool(item):
    if item.name == "search_db":
        return await search_database(json.loads(item.arguments))
    if item.name == "fetch_url":
        return await fetch(json.loads(item.arguments))
    raise ValueError(item.name)

resp = await client.responses.create(model="gpt-5.2", input="...", tools=TOOLS)

calls = [item for item in resp.output if item.type == "function_call"]
results = await asyncio.gather(*(execute_tool(c) for c in calls))

tool_outputs = [
    {"type": "function_call_output", "call_id": c.call_id, "output": json.dumps(r)}
    for c, r in zip(calls, results)
]

resp2 = await client.responses.create(
    model="gpt-5.2",
    previous_response_id=resp.id,
    input=tool_outputs,
)
```

To disable parallel calls (force one at a time):

```python
client.responses.create(..., parallel_tool_calls=False)
```

---

## Controlling Tool Choice

```python
client.responses.create(..., tool_choice="auto")        # default
client.responses.create(..., tool_choice="none")        # text-only response
client.responses.create(..., tool_choice="required")    # must call at least one tool
client.responses.create(..., tool_choice={"type": "function", "name": "get_sku_inventory"})  # force this fn
```

Pattern: after all your function results come back, **set `tool_choice="none"` on the final follow-up** to force the model to write a synthesis paragraph rather than calling more tools.

---

## Streaming Tool Calls

```python
with client.responses.stream(
    model="gpt-5.2",
    input="What's the price of sku-foo?",
    tools=TOOLS,
) as stream:
    for event in stream:
        if event.type == "response.output_item.added" and event.item.type == "function_call":
            print(f"\n[calling {event.item.name}...]")
        elif event.type == "response.function_call_arguments.delta":
            # Partial JSON string — don't try to parse yet
            print(event.delta, end="", flush=True)
        elif event.type == "response.function_call_arguments.done":
            # Now you have the full args string
            print(f"\n[args complete: {event.arguments}]")

    final = stream.get_final_response()
    calls = [it for it in final.output if it.type == "function_call"]
    # Execute and chain as in the manual loop above
```

The argument-delta events stream the JSON as it's generated. For most UI purposes, just wait for `.done` and `json.loads` the full string.

---

## Returning Rich Outputs (images, files) from a Tool

The `output` of `function_call_output` doesn't have to be a string. It can be a content-parts list:

```python
tool_outputs = [{
    "type": "function_call_output",
    "call_id": call.call_id,
    "output": [
        {"type": "output_text", "text": "Here is the chart you asked for."},
        {"type": "input_image", "image_url": "data:image/png;base64,..."},
    ],
}]
```

The model will see the image (when the model is vision-capable, like gpt-5.2). Useful for chart-generation tools, screenshot-returning tools, etc.

---

## Built-In Tools

These are one-line opt-ins. The model decides when to call them; OpenAI runs them on its infra.

### Web search

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="What was the closing price of NVDA last Friday?",
    tools=[{
        "type": "web_search",
        "search_context_size": "medium",            # "low" | "medium" | "high"
    }],
    include=["web_search_call.results"],            # to see raw results
)

# Citations appear as annotations on assistant text:
for item in resp.output:
    if item.type == "message":
        for content in item.content:
            if content.type == "output_text":
                for ann in content.annotations:
                    if ann.type == "url_citation":
                        print(ann.url, ann.title)
```

Options: `user_location` (approximate), `filters.allowed_domains`.

### File search (vector store)

```python
# First create a vector store and upload files (see Files & Vector Stores docs)
vs = client.vector_stores.create(name="company-docs")
client.vector_stores.files.upload_and_poll(vector_store_id=vs.id, file=open("manual.pdf", "rb"))

resp = client.responses.create(
    model="gpt-5.2",
    input="How do I return a product?",
    tools=[{
        "type": "file_search",
        "vector_store_ids": [vs.id],
        "max_num_results": 5,
    }],
    include=["file_search_call.results"],
)

# file_citation annotations on output_text point to the chunks used
```

### Code interpreter

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="Plot the function y = sin(x) * exp(-x/5) for x in [0, 20].",
    tools=[{"type": "code_interpreter", "container": {"type": "auto"}}],
    include=["code_interpreter_call.outputs"],
)

# Generated files (e.g., chart PNG) appear in code_interpreter_call items
for item in resp.output:
    if item.type == "code_interpreter_call":
        for out in item.outputs or []:
            if out.type == "image":
                # out.file_id — download via client.files.content(out.file_id)
                print(f"image file: {out.file_id}")
```

Pre-installed in the sandbox: `pandas`, `numpy`, `matplotlib`, `pillow`, `python-docx`, `python-pptx`, `pypdf`, `requests` (with some restrictions).

### Image generation (inline)

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="Generate a watercolor painting of a cat.",
    tools=[{
        "type": "image_generation",
        "model": "gpt-image-1.5",
        "size": "1024x1024",
        "quality": "high",
        "partial_images": 2,                        # enables streaming preview
    }],
)
```

During streaming, `response.image_generation_call.partial_image` events carry `partial_image_b64` for progressive UI.

### MCP (remote tool server)

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="...",
    tools=[{
        "type": "mcp",
        "server_label": "stripe",
        "server_url": "https://my-mcp-server.example.com/sse",
        "require_approval": "never",                 # or "always" — default
        "allowed_tools": ["search_customers", "create_invoice"],
        "headers": {"Authorization": "Bearer ..."},
    }],
)
```

Default `require_approval="always"` — each call requires a `McpApprovalResponse` input item from your code before the call proceeds. Use `"never"` for trusted servers to auto-approve.

### Apply patch (diff-based code editing)

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="Refactor the loop in main.py to use itertools.",
    tools=[{"type": "apply_patch"}],
    # ... attach files via input items or file_search
)

# resp.output contains apply_patch_tool_call items with unified diffs
```

### Computer use

```python
resp = client.responses.create(
    model="gpt-5.2",
    input="Open Excel and create a new spreadsheet.",
    tools=[{"type": "computer"}],
)
# The model emits actions (click, type, screenshot) — your code executes them in a VM
# and feeds back ComputerCallOutput items with screenshots.
```

Higher-touch than other built-ins — you must run a sandboxed VM, execute actions, and feed back screenshots.

---

## Combining User Functions and Built-ins

Mix freely:

```python
tools = [
    {"type": "web_search"},
    {"type": "code_interpreter", "container": {"type": "auto"}},
    {
        "type": "function",
        "name": "lookup_customer",
        "parameters": {...},
        "strict": True,
    },
]
```

The model picks. Your loop only needs to execute your own function calls — built-ins run themselves.

---

## Common Pitfalls

- **`arguments` is a JSON string.** Always `json.loads(item.arguments)`. Never string-match.
- **Match outputs by `call_id`, never `id`.**
- **`output` in function_call_output is a string** (typically JSON). Or a content-parts list — but not a raw dict.
- **Define `tools` once at module scope.** Rebuilding kills prompt caching.
- **`strict: True` requires the JSON-schema subset** — see `../../shared/structured-outputs.md`. Most failures come from non-required optional fields or missing `additionalProperties: false`.
- **`tool_choice="required"` doesn't force a specific tool.** Use `tool_choice={"type": "function", "name": "..."}` for that.
- **Don't try to parse partial argument deltas mid-stream.** Wait for `response.function_call_arguments.done`.
- **MCP defaults to `require_approval="always"`.** If you're not handling approval items, set `"never"` or your model hangs at the first call.
