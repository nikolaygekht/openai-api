# TypeScript / Node — Tool Use (Responses API)

For conceptual background read `../../shared/tool-use-concepts.md` first.

---

## Function Tools — Declaration

Flat shape — no `function: {...}` wrapper:

```ts
const TOOLS: OpenAI.Responses.FunctionTool[] = [
  {
    type: 'function',
    name: 'get_sku_inventory',
    description: 'Return inventory details for a SKU.',
    parameters: {
      type: 'object',
      properties: {
        sku: { type: 'string', description: 'Stock-keeping unit identifier' },
      },
      required: ['sku'],
      additionalProperties: false,
    },
    strict: true,
  },
];
```

**Define `TOOLS` at module scope.** Rebuilding the list per request invalidates the prompt cache.

### Auto-derive from Zod

```ts
import OpenAI from 'openai';
import { zodResponsesFunction } from 'openai/helpers/zod';
import { z } from 'zod/v4';

const GetSkuInventory = z.object({
  sku: z.string().describe('Stock-keeping unit identifier'),
});

const tools = [
  zodResponsesFunction({
    name: 'get_sku_inventory',
    description: 'Return inventory details for a SKU.',
    parameters: GetSkuInventory,
  }),
];
```

The helper generates a strict-mode schema from your Zod object. Field `.describe(...)` calls become parameter descriptions.

---

## The Agent Loop — Manual Version

```ts
import OpenAI from 'openai';

const client = new OpenAI();

async function executeTool(name: string, args: Record<string, unknown>): Promise<unknown> {
  if (name === 'get_sku_inventory') {
    return { sku: args.sku, in_stock: 42, price_usd: 12.99 };
  }
  throw new Error(`unknown tool ${name}`);
}

let resp = await client.responses.create({
  model: 'gpt-5.2',
  instructions: 'You are a helpful inventory assistant.',
  input: "What's the price of sku-froge-lily-pad-deluxe?",
  tools: TOOLS,
});

while (resp.output.some((it) => it.type === 'function_call')) {
  const toolOutputs: OpenAI.Responses.FunctionCallOutputParam[] = [];

  for (const item of resp.output) {
    if (item.type === 'function_call') {
      const args = JSON.parse(item.arguments);
      const result = await executeTool(item.name, args);
      toolOutputs.push({
        type: 'function_call_output',
        call_id: item.call_id,
        output: JSON.stringify(result),
      });
    }
  }

  resp = await client.responses.create({
    model: 'gpt-5.2',
    previous_response_id: resp.id,
    input: toolOutputs,
  });
}

console.log(resp.output_text);
```

- **Always `JSON.parse(item.arguments)`** — it's a string.
- **Correlate by `call_id`**, not `id`.
- **`output` must be a string** (typically JSON-encoded) — or a content-parts array for rich results (images, files).
- **Use `previous_response_id`** to chain so the cached prefix stays warm.

---

## Parallel Tool Calls

The model may emit multiple `function_call` items in a single response (default `parallel_tool_calls: true`). Run them in parallel:

```ts
const calls = resp.output.filter((it) => it.type === 'function_call');

const results = await Promise.all(
  calls.map(async (call) => ({
    call,
    result: await executeTool(call.name, JSON.parse(call.arguments)),
  })),
);

const toolOutputs = results.map(({ call, result }) => ({
  type: 'function_call_output' as const,
  call_id: call.call_id,
  output: JSON.stringify(result),
}));

const resp2 = await client.responses.create({
  model: 'gpt-5.2',
  previous_response_id: resp.id,
  input: toolOutputs,
});
```

Disable parallel calls (one at a time):

```ts
client.responses.create({ ..., parallel_tool_calls: false });
```

---

## Controlling Tool Choice

```ts
client.responses.create({ ..., tool_choice: 'auto' });         // default
client.responses.create({ ..., tool_choice: 'none' });         // text-only
client.responses.create({ ..., tool_choice: 'required' });     // must call ≥ 1 tool
client.responses.create({ ..., tool_choice: { type: 'function', name: 'get_sku_inventory' } });  // force this fn
```

Pattern: after tool results come back, **set `tool_choice: 'none'` on the final follow-up** to force a synthesis paragraph.

---

## Streaming Tool Calls

```ts
const runner = client.responses.stream({
  model: 'gpt-5.2',
  input: "What's the price of sku-foo?",
  tools: TOOLS,
});

for await (const event of runner) {
  if (event.type === 'response.output_item.added' && event.item.type === 'function_call') {
    console.log(`\n[calling ${event.item.name}...]`);
  } else if (event.type === 'response.function_call_arguments.delta') {
    // partial JSON — don't parse yet
  } else if (event.type === 'response.function_call_arguments.done') {
    console.log(`[args: ${event.arguments}]`);
  }
}

const final = await runner.finalResponse();
const calls = final.output.filter((it) => it.type === 'function_call');
// execute and chain
```

The argument-delta events stream the JSON string. For UI, just wait for `.done` and parse the full string.

---

## Rich Outputs from a Tool

`output` doesn't have to be a string. It can be a content-parts array:

```ts
const toolOutputs = [{
  type: 'function_call_output' as const,
  call_id: call.call_id,
  output: [
    { type: 'output_text', text: 'Here is the chart you asked for.' },
    { type: 'input_image', image_url: `data:image/png;base64,${chartBase64}` },
  ],
}];
```

Vision-capable models (gpt-5.2 etc.) will see the image.

---

## Built-In Tools

### Web search

```ts
const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: 'What was the closing price of NVDA last Friday?',
  tools: [{
    type: 'web_search',
    search_context_size: 'medium',
  }],
  include: ['web_search_call.results'],
});

// citations as annotations on output_text
for (const item of resp.output) {
  if (item.type === 'message') {
    for (const part of item.content) {
      if (part.type === 'output_text') {
        for (const ann of part.annotations ?? []) {
          if (ann.type === 'url_citation') console.log(ann.url, ann.title);
        }
      }
    }
  }
}
```

### File search

```ts
const vs = await client.vectorStores.create({ name: 'company-docs' });
await client.vectorStores.files.uploadAndPoll(vs.id, fs.createReadStream('manual.pdf'));

const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: 'How do I return a product?',
  tools: [{
    type: 'file_search',
    vector_store_ids: [vs.id],
    max_num_results: 5,
  }],
  include: ['file_search_call.results'],
});
```

### Code interpreter

```ts
const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: 'Plot y = sin(x) * exp(-x/5) for x in [0, 20].',
  tools: [{ type: 'code_interpreter', container: { type: 'auto' } }],
  include: ['code_interpreter_call.outputs'],
});

for (const item of resp.output) {
  if (item.type === 'code_interpreter_call') {
    for (const out of item.outputs ?? []) {
      if (out.type === 'image') {
        const file = await client.files.content(out.file_id);
        // file is a Response — file.body / file.arrayBuffer()
      }
    }
  }
}
```

### Image generation

```ts
const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: 'Generate a watercolor painting of a cat.',
  tools: [{
    type: 'image_generation',
    model: 'gpt-image-1.5',
    size: '1024x1024',
    quality: 'high',
    partial_images: 2,
  }],
});
```

During streaming, `response.image_generation_call.partial_image` events carry `partial_image_b64`.

### MCP (remote)

```ts
const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: '...',
  tools: [{
    type: 'mcp',
    server_label: 'stripe',
    server_url: 'https://my-mcp-server.example.com/sse',
    require_approval: 'never',          // or 'always' — default
    allowed_tools: ['search_customers', 'create_invoice'],
    headers: { Authorization: 'Bearer ...' },
  }],
});
```

Default `require_approval: 'always'` — you must reply with an `McpApprovalResponse` input item to proceed. Use `'never'` for trusted servers.

---

## Common Pitfalls

- **`arguments` is a JSON string.** Always `JSON.parse`.
- **Match outputs by `call_id`, never `id`.**
- **`output` is a string** (typically JSON-encoded) — or a content-parts array.
- **Define `tools` at module scope.** Don't rebuild per request.
- **`strict: true` requires the JSON-schema subset** — see `../../shared/structured-outputs.md`.
- **`tool_choice: 'required'` doesn't force a specific tool.** Use `{ type: 'function', name: '...' }` for that.
- **Don't parse partial argument deltas mid-stream.** Wait for `.done`.
- **MCP defaults to `require_approval: 'always'`.** Set `'never'` if you're not handling approvals.
- **Use SDK types** (`OpenAI.Responses.FunctionTool`, `OpenAI.Responses.FunctionCallOutputParam`) — don't redefine your own interfaces.
