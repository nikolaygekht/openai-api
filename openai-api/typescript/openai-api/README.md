# TypeScript / Node — OpenAI Responses API

Official SDK: `openai` (≥ 6.38 as of May 2026). Works in Node, Deno, Bun, Cloudflare Workers, and edge runtimes. **Do not call OpenAI directly from a browser** — see Secrets.

For secret handling see `../../shared/secrets.md` — read that BEFORE writing key-loading code. For streaming see `streaming.md`. For tools see `tool-use.md`. For audio see `audio.md`.

---

## Install

```bash
npm install openai
# or
pnpm add openai
yarn add openai
```

For local dev:

```bash
npm install dotenv
```

(Framework already-bundled in Next.js / Vite / Astro — don't add `dotenv` there.)

---

## Authentication

The SDK reads `OPENAI_API_KEY` from `process.env` automatically. Never hardcode, never log — see `../../shared/secrets.md`.

```ts
import 'dotenv/config';                    // local dev only — loads .env into process.env
import OpenAI from 'openai';

const client = new OpenAI();                // picks up OPENAI_API_KEY, OPENAI_ORG_ID, OPENAI_PROJECT_ID, OPENAI_BASE_URL
```

Optional env vars: `OPENAI_API_KEY`, `OPENAI_ORG_ID`, `OPENAI_PROJECT_ID`, `OPENAI_BASE_URL`.

For Azure OpenAI use `AzureOpenAI`:

```ts
import { AzureOpenAI } from 'openai';
const client = new AzureOpenAI({
  apiVersion: '2026-01-01',
  endpoint: 'https://your-resource.openai.azure.com/',
});
```

**Browser warning:** Never instantiate `OpenAI` in client-side code. The key would be inlined into your JS bundle and visible to every visitor. Proxy via a server endpoint (Next.js Route Handler, Express, Cloudflare Worker, etc.). Don't put `OPENAI_API_KEY` behind a `NEXT_PUBLIC_` / `VITE_` / `REACT_APP_` prefix — those are inlined into client JS.

---

## Minimal Responses Call

```ts
import OpenAI from 'openai';

const client = new OpenAI();

const response = await client.responses.create({
  model: 'gpt-5.2',
  instructions: 'You are a coding assistant that talks like a pirate.',
  input: 'How do I check if a JavaScript value is an array?',
});

console.log(response.output_text);
```

Key things:

- **`instructions`** is the top-level system message. Don't fake it with a `{role:'system', content:...}` item.
- **`input`** is a string OR an array of items.
- **`response.output_text`** is the convenience accessor — concatenated assistant text. For tool calls or reasoning items, iterate `response.output`.

### Multi-modal input

```ts
const response = await client.responses.create({
  model: 'gpt-5.2',
  input: [{
    role: 'user',
    content: [
      { type: 'input_text', text: "What's in this image?" },
      { type: 'input_image', image_url: 'https://example.com/cat.jpg' },
    ],
  }],
});
```

### Multi-turn conversation

```ts
const r1 = await client.responses.create({ model: 'gpt-5.2', input: 'Hi, I am Sam.' });
const r2 = await client.responses.create({
  model: 'gpt-5.2',
  previous_response_id: r1.id,
  input: 'What did I just tell you?',
});
```

For durable / multi-session state use the Conversations API:

```ts
const conv = await client.conversations.create();
await client.responses.create({ model: 'gpt-5.2', conversation: conv.id, input: '...' });
// Later
await client.responses.create({ model: 'gpt-5.2', conversation: conv.id, input: '...' });
```

For stateless (ZDR) mode: `store: false` plus `include: ['reasoning.encrypted_content']` and echo reasoning items back. See `../../shared/tool-use-concepts.md`.

---

## Async

Everything in the Node SDK is already async — methods return `Promise<T>`. There is no separate async client class.

---

## Resource Methods

```ts
client.responses.create(params)                 // → Promise<Response>
client.responses.retrieve(responseId)           // → Promise<Response>
client.responses.delete(responseId)             // → Promise<void>
client.responses.cancel(responseId)             // → Promise<Response>
client.responses.compact(params)                // → Promise<CompactedResponse>
client.responses.stream(params)                 // → Stream<Event>  (helper, see streaming.md)
client.responses.parse(params)                  // → ParsedResponse (zod-typed)
client.responses.inputItems.list(responseId)    // → Page<Item>
client.responses.inputTokens.count(params)      // → InputTokenCount
```

---

## Counting Tokens

```ts
const count = await client.responses.inputTokens.count({
  model: 'gpt-5.2',
  input: longInput,
});
console.log(count.input_tokens);
```

---

## Errors

The SDK exposes a typed exception hierarchy on `OpenAI`:

```ts
import OpenAI from 'openai';
const {
  OpenAIError,                  // base
  APIError,                     // any API-side error
  APIConnectionError,           // network
  APIConnectionTimeoutError,
  APIUserAbortError,            // AbortController cancellation
  BadRequestError,              // 400
  AuthenticationError,          // 401
  PermissionDeniedError,        // 403
  NotFoundError,                // 404
  ConflictError,                // 409
  UnprocessableEntityError,     // 422
  RateLimitError,               // 429
  InternalServerError,          // 5xx
  LengthFinishReasonError,
  ContentFilterFinishReasonError,
} = OpenAI;

try {
  await client.responses.create({ model: 'gpt-5.2', input: '...' });
} catch (err) {
  if (err instanceof OpenAI.RateLimitError) {
    console.log('rate limited:', err.status, err.request_id, err.headers?.['retry-after']);
  } else if (err instanceof OpenAI.APIError) {
    console.log(`${err.status} ${err.code}:`, err.message);
  } else {
    throw err;
  }
}
```

`APIError` exposes `status`, `code`, `param`, `type`, `request_id`, `headers`, `error`, `message`.

For HTTP-level details see `../../shared/error-codes.md`.

---

## Retries & Timeouts

Defaults: **2 retries**, **10-minute timeout**. Retries fire on connection errors, 408, 409, 429, ≥500.

Override per client:

```ts
const client = new OpenAI({
  maxRetries: 5,
  timeout: 30 * 1000,            // milliseconds
});
```

Override per request:

```ts
await client.responses.create(
  { model: 'gpt-5.2', input: '...' },
  { maxRetries: 0, timeout: 5000 },
);
```

Disable built-in retries and roll your own:

```ts
const client = new OpenAI({ maxRetries: 0 });
// then catch and retry RateLimitError / APIConnectionError / InternalServerError
```

See `../../shared/error-codes.md` for a backoff template.

---

## Reading Headers

For success, use `.withResponse()`:

```ts
const { data, response } = await client.responses
  .create({ model: 'gpt-5.2', input: 'hi' })
  .withResponse();
console.log(response.headers.get('x-request-id'));
```

For raw HTTP access (no parsing) use `.asResponse()`:

```ts
const response = await client.responses
  .create({ model: 'gpt-5.2', input: 'hi' })
  .asResponse();
// response is a fetch Response — call .json(), .text(), .body, etc.
```

For failures the exception carries headers:

```ts
catch (err) {
  if (err instanceof OpenAI.APIError) {
    console.log(err.headers?.['x-ratelimit-remaining-requests']);
  }
}
```

---

## Usage & Caching

```ts
const resp = await client.responses.create({ model: 'gpt-5.2', input: '...' });

resp.usage.input_tokens;                                  // total input
resp.usage.input_tokens_details.cached_tokens;            // cached portion
resp.usage.output_tokens;
resp.usage.output_tokens_details.reasoning_tokens;        // hidden reasoning
resp.usage.total_tokens;
```

For caching strategy see `../../shared/prompt-caching.md`.

---

## Reasoning Models (gpt-5.x, o-series)

```ts
const resp = await client.responses.create({
  model: 'gpt-5.2',
  input: 'Prove that there are infinitely many primes.',
  reasoning: { effort: 'high', summary: 'auto' },
});

for (const item of resp.output) {
  if (item.type === 'reasoning') {
    for (const s of item.summary ?? []) console.log(s.text);
  }
}
```

`reasoning.effort`: `'none' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'`. Defaults differ per model — see SKILL.md's "Reasoning & Effort" section.

---

## Cancellation (AbortController)

```ts
const ac = new AbortController();
setTimeout(() => ac.abort(), 5000);     // cancel after 5s

try {
  await client.responses.create(
    { model: 'gpt-5.2', input: 'Write a novel.' },
    { signal: ac.signal },
  );
} catch (err) {
  if (err instanceof OpenAI.APIUserAbortError) {
    console.log('cancelled');
  } else throw err;
}
```

---

## Common Pitfalls (TS-specific)

- **Never use OpenAI from the browser.** Proxy through a server. The key cannot be safely embedded in client-side JS.
- **Don't put the key behind `NEXT_PUBLIC_` / `VITE_` / `REACT_APP_`.** Those env prefixes are *intentionally* inlined into client bundles.
- **`output_text` vs `output`.** `output_text` is the convenience accessor; `output` is the array of typed items (`type: 'message' | 'function_call' | 'reasoning' | ...`).
- **Parse `arguments` with `JSON.parse`.** Tool-call arguments come as a JSON string.
- **Use SDK types**, don't redefine your own: `OpenAI.Responses.Response`, `OpenAI.Responses.FunctionTool`, etc.
- **Don't rebuild `tools` per request.** Define at module scope to keep the cached prefix stable.
- **`previous_response_id` requires `store: true`** (the default). For `store: false` you manage the full input array.
- **TS strictness:** if you set `reasoning.effort: 'none'` on a model that doesn't support it (pre-5.1 reasoning models), you'll get a 400. Branch on model.
- **Don't use `chat.completions`** unless maintaining legacy code.
