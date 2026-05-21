# OpenAI Prompt Caching

OpenAI caches prompt prefixes automatically on all gpt-5.x, o-series, and gpt-4o-class models. There is no `cache_control` parameter to manage — caching is purely prefix-based and best-effort. Done right, you pay **10% of the base input rate** on cached tokens (a 90% discount on gpt-5.x). Done wrong, you pay full price every call and don't realize it.

This file explains how to design your prompts so caching actually works, how to verify it, and the silent invalidators that quietly destroy cache hit rates.

---

## How OpenAI's Caching Works

1. **Prefix routing.** The first ~256 tokens of your request are hashed and used to route the request to a backend machine that has seen that prefix before. If no machine has it, you get a cold start; the prefix is then stored on the machine that handled the request.

2. **Minimum prefix length.** Prefixes below **1024 tokens** are not cached at all — there is no caching for short prompts. Anything under that threshold pays full price.

3. **Granularity.** Once cached, the longest contiguous prefix match (in increments of 128 tokens roughly) is reused; everything after the first divergent byte must be recomputed.

4. **TTL.** Default cache lifetime is **5–10 minutes of inactivity**, extending to **up to 1 hour off-peak**. Set `prompt_cache_retention="24h"` on `responses.create()` to opt into a separate 24-hour storage backend (uses GPU-local offload — more expensive infrastructure, hence not the default).

5. **Discount.** On gpt-5.x, the cached input rate is exactly **10% of base input** (90% off). Examples:
   - `gpt-5.2`: $1.75 → $0.175 per million cached input tokens.
   - `gpt-5.5`: $5.00 → $0.50.
   - `gpt-realtime-2` audio: $32 → $0.40 (~98.75% off — the biggest lever in production Realtime apps).

---

## Render Order in Responses API

Inputs are concatenated in this order for caching purposes:

1. `instructions` (top-level)
2. `tools` array (serialized)
3. `input` items (in order)

**Put stable content first.** Bad ordering kills caching:

```python
# BAD — timestamp in instructions breaks every call
client.responses.create(
    model="gpt-5.2",
    instructions=f"You are a helpful assistant. Current time: {datetime.now()}",
    input=user_question,
)
```

```python
# GOOD — stable instructions; dynamic content goes in input
client.responses.create(
    model="gpt-5.2",
    instructions="You are a helpful assistant.",  # stable across calls
    input=[
        {"role": "user", "content": f"[time: {datetime.now()}] {user_question}"},
    ],
)
```

For tools, define the list once and reuse the same Python object / TypeScript const. Don't rebuild it inside the call, don't sort it differently per request, don't insert "current" timestamps into a tool description.

---

## Verifying Cache Hits

Every Response has the cache count in `usage`:

```python
resp = client.responses.create(model="gpt-5.2", input=...)
print(resp.usage.input_tokens)
print(resp.usage.input_tokens_details.cached_tokens)
```

- `usage.input_tokens` — total input tokens (cached + uncached).
- `usage.input_tokens_details.cached_tokens` — the portion that hit the cache.

If `cached_tokens` is 0 across repeated requests with what *should* be the same prefix, something is silently invalidating the cache. Run through the checklist below.

**Path note:** This is the Responses API field. On Chat Completions it's `usage.prompt_tokens_details.cached_tokens`.

---

## Silent Invalidator Checklist

If repeated requests aren't cache-hitting, check each of these in order:

1. **Timestamps in `instructions`.** `f"Current time is {datetime.now()}"` invalidates every call. Move dynamic data to the trailing user message.

2. **Tool list reordering.** Python `set` → `list` produces different orders each run. Always use a stable, sorted `list` of tool dicts defined at module scope.

3. **Tool definition mutation.** A `description` that interpolates the current user's name. A `parameters` schema that includes the current week's enum values. These look stable but aren't.

4. **Unsorted JSON keys.** If you build the request body manually (raw HTTP) and dump a dict to JSON, `json.dumps({"a":1,"b":2})` and `json.dumps({"b":2,"a":1})` produce different bytes — and Python `dict` insertion order changes between code paths. Use `sort_keys=True` if hand-building.

5. **Model snapshot drift.** `model="gpt-5.2"` and `model="gpt-5.2-2025-12-11"` are different cache namespaces. Pick one and stick with it across requests in a session.

6. **`previous_response_id` chaining without resending `instructions`.** When chaining via `previous_response_id`, the prior `instructions` are NOT carried over — if you re-supply different instructions on the follow-up, the prefix mismatches the chain. (Set `instructions` consistently across the whole chain, or use the Conversations API.)

7. **Prefix shorter than 1024 tokens.** Sub-1024 prefixes are never cached, full stop. Pad with system instructions or move long context (retrieved docs) early in the input array.

8. **First 256-token region varies.** Even if most of your prefix is identical, the first ~256 tokens determine routing. If they differ, the request lands on a cold machine and pays full price.

9. **Mixing `store=true` and `store=false` for the same logical conversation.** Stateless mode (`store=false`) has different cache semantics — don't interleave them.

10. **Concurrent fan-out exceeding ~15 RPM per cache key.** Past ~15 RPM, the same logical prefix spreads across multiple backend machines and some hit cold paths. Use `prompt_cache_key` to keep traffic pinned (see below).

---

## Multi-Tenant Routing: `prompt_cache_key`

By default, all requests in your account compete for cache space. For multi-tenant apps where many tenants share a system prompt but you want each to have its own cache slot — or for high-volume single-tenant apps where one prefix is hitting cache fragmentation — pass `prompt_cache_key`:

```python
client.responses.create(
    model="gpt-5.2",
    instructions=SYSTEM_PROMPT,
    input=user_message,
    prompt_cache_key=f"tenant-{tenant_id}-v3",
)
```

The key is hashed together with the prefix to route requests. Same `(prefix, prompt_cache_key)` → same backend machine → better hit rate.

Rules of thumb:
- Use a stable, low-cardinality key per tenant. Don't burn cardinality on request IDs.
- Keep each unique `(prefix, prompt_cache_key)` pair below ~15 RPM. Beyond that the cache fans out and you lose the benefit.
- Bump the version suffix (`-v3` → `-v4`) when you intentionally invalidate (e.g., system prompt change).

---

## Extended Retention: `prompt_cache_retention`

For prefixes you'll reuse across hours but not minutes, opt into 24-hour retention:

```python
client.responses.create(
    model="gpt-5.2",
    instructions=BIG_SYSTEM_PROMPT,
    input=user_message,
    prompt_cache_retention="24h",  # or "in_memory" (default)
)
```

This uses a separate storage backend (GPU-local offload). Same 1024-token minimum still applies. Same invalidators still kill the cache.

---

## Architecture Pattern: Stable Prefix, Volatile Suffix

The most effective cache-friendly prompt structure:

```
┌────────────────────────────────────┐
│ instructions (frozen system prompt) │  ← cache prefix starts here
│                                     │
│ tools (stable list)                 │
│                                     │
│ input[0]: retrieved docs (stable    │
│           per session)              │
│                                     │
│ input[1..n-1]: conversation history │  ← cache extends as conversation grows
│                                     │
│ input[n]: current user message      │  ← only this differs per turn
└────────────────────────────────────┘
```

With this layout, every new user turn caches the entire prior chain. On gpt-5.2 with a 50K-token context, you pay ~$0.0175 for 10K cached input + ~$0.07 for 4K new input = roughly $0.09 per turn, versus $0.10+ uncached. The savings compound as the conversation grows.

If you're building a chat UI: use `previous_response_id` or the Conversations API — the server reconstructs the cached prefix for you, and you only send the new turn.

---

## Worked Example: Tool-Heavy Agent

```python
TOOLS = [
    {"type": "function", "name": "search_db", "description": "...", "parameters": {...}, "strict": True},
    {"type": "function", "name": "send_email", "description": "...", "parameters": {...}, "strict": True},
    # ... 20 more tools
]
SYSTEM = "You are a customer-support agent. " * 50  # ~500 tokens — pad to ≥1024 with real content

def handle_turn(conversation, user_msg):
    return client.responses.create(
        model="gpt-5.2",
        instructions=SYSTEM,             # never changes
        tools=TOOLS,                     # never changes
        input=conversation + [{"role": "user", "content": user_msg}],
    )
```

After the first turn, every subsequent call should show non-zero `cached_tokens`. If it doesn't, check whether `TOOLS` or `SYSTEM` is being rebuilt per call (e.g., inside a `def` instead of at module scope).
