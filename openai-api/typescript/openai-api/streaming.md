# TypeScript / Node — Streaming (Responses API)

Two patterns: raw async iteration via `stream: true`, or the helper `client.responses.stream(...)` which adds named events and a `finalResponse()` promise.

For the conceptual event taxonomy see `../python/openai-api/streaming.md` "Event Catalog" — event types are identical across SDKs.

---

## Helper: `client.responses.stream(...)` (recommended)

```ts
import OpenAI from 'openai';

const client = new OpenAI();

const runner = client.responses.stream({
  model: 'gpt-5.2',
  input: 'Write a one-sentence bedtime story about a unicorn.',
});

runner.on('response.output_text.delta', (e) => process.stdout.write(e.delta));
runner.on('response.completed', (e) => {
  console.log(`\n[${e.response.usage.output_tokens} output tokens]`);
});

for await (const event of runner) {
  // also iterable
}

const final = await runner.finalResponse();
```

The helper is an event emitter AND an async iterable. Use whichever style fits.

### With structured outputs (Zod)

```ts
import OpenAI from 'openai';
import { zodTextFormat } from 'openai/helpers/zod';
import { z } from 'zod/v4';

const MathResponse = z.object({
  steps: z.array(z.object({
    explanation: z.string(),
    output: z.string(),
  })),
  final_answer: z.string(),
});

const client = new OpenAI();

const runner = client.responses.stream({
  model: 'gpt-5.2',
  input: 'solve 8x + 31 = 2',
  text: { format: zodTextFormat(MathResponse, 'math_response') },
});

for await (const event of runner) {
  if (event.type === 'response.output_text.delta') {
    process.stdout.write(event.delta);
  }
}

const final = await runner.finalResponse();
console.log('\nAnswer:', final.output_parsed.final_answer);   // typed Z.infer<typeof MathResponse>
```

### With tools (Zod)

```ts
import { zodResponsesFunction } from 'openai/helpers/zod';

const Query = z.object({
  table_name: z.string(),
  columns: z.array(z.string()),
});

const tool = zodResponsesFunction({ name: 'query', parameters: Query });

const runner = client.responses.stream({
  model: 'gpt-5.2',
  input: 'Find all orders from last week.',
  tools: [tool],
});

for await (const event of runner) {
  if (event.type === 'response.function_call_arguments.done') {
    console.log('args:', event.arguments);
  }
}
```

See `tool-use.md` for the full tool result handling pattern.

---

## Raw Iteration: `stream: true`

```ts
const stream = await client.responses.create({
  model: 'gpt-5.2',
  input: 'Write a haiku about caching.',
  stream: true,
});

for await (const event of stream) {
  if (event.type === 'response.output_text.delta') {
    process.stdout.write(event.delta);
  }
}
```

The async iterator handles the SSE connection. **Always iterate to completion** (or call `stream.controller.abort()` and clean up) — leaking the connection starves your worker.

---

## Pattern: Stream Text + Detect Tool Calls

```ts
const runner = client.responses.stream({
  model: 'gpt-5.2',
  input: 'What\'s the price of sku-foo?',
  tools: TOOLS,
});

const calls: any[] = [];

for await (const event of runner) {
  if (event.type === 'response.output_text.delta') {
    process.stdout.write(event.delta);
  } else if (event.type === 'response.output_item.added' && event.item.type === 'function_call') {
    console.log(`\n[calling ${event.item.name}...]`);
  } else if (event.type === 'response.function_call_arguments.done') {
    console.log(`[args: ${event.arguments}]`);
  }
}

const final = await runner.finalResponse();
for (const item of final.output) {
  if (item.type === 'function_call') calls.push(item);
}
```

---

## Pattern: Background + Resume

For long-running jobs that must survive a process restart:

```ts
const stream = await client.responses.create({
  model: 'gpt-5.2',
  input: veryLongTask,
  background: true,
  stream: true,
});

let responseId: string | null = null;
for await (const event of stream) {
  if (event.type === 'response.created') responseId = event.response.id;
  if (event.sequence_number >= 10) break;
}

// Persist responseId to a database, then in a separate process:
const resumed = await client.responses.retrieve(responseId!, {
  stream: true,
  starting_after: 10,
});
for await (const event of resumed) {
  console.log(event);
}
```

This is the only pattern that survives a process crash mid-stream.

---

## Server-Sent Events to a Browser

Common pattern: serve OpenAI streaming through your own HTTP endpoint to a browser EventSource or fetch reader.

```ts
// Next.js Route Handler (app/api/chat/route.ts)
import OpenAI from 'openai';

export async function POST(req: Request) {
  const { messages } = await req.json();
  const client = new OpenAI();

  const stream = await client.responses.create({
    model: 'gpt-5.2',
    input: messages,
    stream: true,
  });

  const encoder = new TextEncoder();
  const readable = new ReadableStream({
    async start(controller) {
      try {
        for await (const event of stream) {
          if (event.type === 'response.output_text.delta') {
            controller.enqueue(encoder.encode(`data: ${JSON.stringify({ delta: event.delta })}\n\n`));
          }
        }
        controller.enqueue(encoder.encode('data: [DONE]\n\n'));
      } catch (err) {
        controller.error(err);
      } finally {
        controller.close();
      }
    },
  });

  return new Response(readable, {
    headers: {
      'content-type': 'text/event-stream',
      'cache-control': 'no-cache',
      'connection': 'keep-alive',
    },
  });
}
```

On the browser, use `EventSource` or `fetch` with a streamed body reader. Never call OpenAI directly from the browser — the proxy pattern keeps your API key on the server.

---

## Common Pitfalls

- **`response.output_text` is empty during streaming.** Accumulate from `response.output_text.delta` events. Final value populated on `response.completed`.
- **Always iterate to completion or abort.** Hanging iterators leak HTTP connections.
- **Don't parse partial argument deltas.** Wait for `response.function_call_arguments.done`, then `JSON.parse`.
- **In edge runtimes** (Workers, Vercel Edge) ensure your fetch polyfill supports streamed bodies — Node 18+ and modern edges do; older Node/polyfills don't.
- **Audio events** only appear in Realtime / WebSocket mode, not in standard `responses.create`.
- **Stream events are interleaved.** Use `item_id` to disambiguate when content from multiple items arrives between events.
