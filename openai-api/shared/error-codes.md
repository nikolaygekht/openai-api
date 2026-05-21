# OpenAI HTTP Errors & Retries

Every OpenAI endpoint returns the same set of HTTP status codes and JSON error body shape. Each SDK wraps them into language-idiomatic exception types — but the *status code* is the source of truth. When in doubt, switch on the integer.

---

## Status Code Reference

| Code | Meaning                                                                | Retry?                     | Action                                                                  |
| ---- | ---------------------------------------------------------------------- | -------------------------- | ----------------------------------------------------------------------- |
| 400  | `BadRequestError` — malformed request, invalid params, schema mismatch | **No**                     | Fix the request payload.                                                |
| 401  | `AuthenticationError` — missing / invalid `Authorization` bearer       | **No**                     | Fix the API key, org, or project.                                       |
| 403  | `PermissionDeniedError` — country/region blocked or no access to model | **No**                     | Switch model or region.                                                 |
| 404  | `NotFoundError` — wrong endpoint, deleted response/file, unknown model | **No**                     | Verify URL and resource id.                                             |
| 408  | Request Timeout (server-side)                                           | **Yes** (auto-retried)     | Retry with backoff.                                                     |
| 409  | `ConflictError` — concurrent modification on same resource             | **Yes** (auto-retried by Python/Node) | Retry; re-read state if needed.                                |
| 422  | `UnprocessableEntityError` — semantically invalid request              | **No**                     | Fix the request.                                                        |
| 429  | `RateLimitError` — RPM / TPM / quota / overloaded                      | **Yes**                    | Respect `Retry-After`; exponential backoff with jitter.                 |
| 500  | `InternalServerError`                                                  | **Yes**                    | Retry with backoff.                                                     |
| 502  | Bad Gateway                                                            | **Yes**                    | Retry with backoff.                                                     |
| 503  | Service Unavailable                                                    | **Yes**                    | Retry with backoff; honor `Retry-After` if present.                     |
| 504  | Gateway Timeout                                                        | **Yes**                    | Retry with longer timeout; consider streaming for very long generations. |

---

## SDK Defaults

| SDK        | Default `max_retries` | Default timeout | Retries on                                |
| ---------- | --------------------- | --------------- | ----------------------------------------- |
| Python     | 2                     | 600 s (10 min)  | conn err, 408, 409, 429, ≥500             |
| Node       | 2                     | 10 min          | conn err, 408, 409, 429, ≥500             |
| .NET       | 3                     | 100 s (NetworkTimeout) | 408, 429, 500, 502, 503, 504        |
| Java       | 2                     | 10 min          | retryable subset (`OpenAIRetryableException`) |

Override per-client or per-request — see the language file for syntax.

---

## Standard Error Body

All non-2xx responses (when JSON is returned) follow this shape:

```json
{
  "error": {
    "message": "The model `gpt-9999` does not exist.",
    "type": "invalid_request_error",
    "code": "model_not_found",
    "param": "model"
  }
}
```

- `message` — always present, human-readable.
- `type` — one of `invalid_request_error`, `authentication_error`, `permission_error`, `not_found_error`, `rate_limit_error`, `server_error`, `api_error`, `tokens_exceeded_error`.
- `code` — machine identifier. Common values:
  - `model_not_found`
  - `insufficient_quota` — billing problem, not a rate limit
  - `context_length_exceeded`
  - `invalid_api_key`
  - `invalid_organization`
  - `content_filter` — your input was blocked
- `param` — name of the offending request parameter, when applicable.

For 5xx errors the server may not return JSON (e.g., HTML from a proxy). SDKs surface that as `InternalServerError` (or `ClientResultException` with `Status >= 500` in .NET) with an empty body.

---

## Useful Response Headers

| Header                             | Purpose                                                                                  |
| ---------------------------------- | ---------------------------------------------------------------------------------------- |
| `x-request-id`                     | Server-side correlation id. **Always log this on failures.**                              |
| `x-ratelimit-limit-requests`       | Account RPM cap                                                                          |
| `x-ratelimit-remaining-requests`   | RPM remaining                                                                            |
| `x-ratelimit-reset-requests`       | Time until RPM resets (e.g., `7m12s`)                                                    |
| `x-ratelimit-limit-tokens`         | Account TPM cap                                                                          |
| `x-ratelimit-remaining-tokens`     | TPM remaining                                                                            |
| `x-ratelimit-reset-tokens`         | Time until TPM resets                                                                    |
| `retry-after`                      | Sent on 429 / 503 — seconds (or HTTP-date) to wait before retrying. **Always respect.**   |
| `openai-organization`              | The org that processed the request                                                       |
| `openai-processing-ms`             | Server processing time in milliseconds                                                   |
| `openai-version`                   | API spec version that served the request                                                 |

**Known issue**: `x-ratelimit-reset-*` may transiently return `0` or `-1`. Code defensively — fall back to exponential backoff if you can't parse the value.

---

## Reading Headers in Each SDK

### Python

On success — use `with_raw_response`:

```python
raw = client.responses.with_raw_response.create(model="gpt-5.2", input="hi")
print(raw.headers["x-request-id"])
resp = raw.parse()
```

On failure — the exception carries the response:

```python
try:
    client.responses.create(model="gpt-5.2", input="hi")
except APIStatusError as e:
    print(e.request_id)
    print(e.response.headers.get("retry-after"))
```

### Node

```ts
const { data, response } = await client.responses
    .create({ model: 'gpt-5.2', input: 'hi' })
    .withResponse();
console.log(response.headers.get('x-request-id'));
```

On failure:

```ts
try {
    await client.responses.create({ model: 'gpt-5.2', input: 'hi' });
} catch (err) {
    if (err instanceof OpenAI.APIError) {
        console.log(err.status, err.code, err.request_id);
    }
}
```

### .NET

```csharp
try {
    var r = client.CreateResponse("gpt-5.2", "hi");
} catch (ClientResultException ex) {
    var resp = ex.GetRawResponse();
    Console.WriteLine($"status={ex.Status} request_id={resp.Headers.TryGetValue("x-request-id", out var v) ? v : null}");
}
```

### Java

```java
try {
    client.responses().create(params);
} catch (OpenAIServiceException e) {
    System.err.println("status=" + e.statusCode() + " body=" + e.body());
}
```

---

## Retry Strategy

The SDKs all implement essentially the same algorithm. When the built-in retries aren't enough (or you need to wrap a streaming call), follow this contract:

1. **Cap retries at ~5 attempts.** Failures past that almost always indicate a real problem.
2. **Exponential delay with full jitter**: `delay = random(0, min(cap, base * 2^attempt))`. Typical values: `base=0.5s`, `cap=8s` interactive, `cap=60s` batch.
3. **Honor `Retry-After`** if present. Parse as seconds first, fall back to HTTP-date.
4. **On 429**, prefer `x-ratelimit-reset-{requests,tokens}` over naive backoff — wait until reset plus a small jitter.
5. **Never retry 400 / 401 / 403 / 404 / 422.** They're deterministic.
6. **Idempotency**: the SDKs generate an `Idempotency-Key` header automatically for retried POSTs, so the server deduplicates. If you build raw HTTP, do the same.
7. **Log `x-request-id`** on every failure — OpenAI support tickets require it.

### Python canonical implementation

```python
import random, time
from openai import OpenAI, RateLimitError, APIConnectionError, InternalServerError

client = OpenAI(max_retries=0)   # disable built-in, manage ourselves

def with_backoff(fn, *, attempts=5, base=0.5, cap=30.0):
    for i in range(attempts):
        try:
            return fn()
        except (RateLimitError, APIConnectionError, InternalServerError) as e:
            if i == attempts - 1:
                raise
            retry_after = None
            if isinstance(e, RateLimitError) and e.response is not None:
                retry_after = float(e.response.headers.get("retry-after") or 0)
            delay = retry_after or random.uniform(0, min(cap, base * 2 ** i))
            time.sleep(delay)
```

---

## Common Issues & Fixes

- **`401 invalid_api_key`** — check `OPENAI_API_KEY` env var, the key prefix (`sk-...`), and that the key isn't disabled in the dashboard.
- **`403` with no clear reason** — likely a regional block (some countries are not supported) or your org doesn't have access to the model (some new models are tier-gated).
- **`404 model_not_found`** — check spelling (`gpt-5.2` not `gpt5.2`), the model may have been retired, or your tier doesn't have access.
- **`429 insufficient_quota`** — billing problem (no credit card on file, hit hard spend cap), NOT a rate limit. Won't recover with backoff. Tell the user to check billing.
- **`429 rate_limit_exceeded`** — true rate limit. Backoff or raise tier.
- **`400 context_length_exceeded`** — input is too long. Switch to `gpt-5.4` (1.05M context), summarize, or chunk.
- **`400 content_filter`** — input was blocked by policy. Surface to the user; do not retry.
- **`500` / `502` / `503` storm** — usually an OpenAI incident. Check status.openai.com via the user; backoff aggressively.
- **Stream cut off mid-response** — SDKs don't auto-resume streams. For long generations, use `background=True` + `response.retrieve(stream=True, starting_after=N)` to resume.
