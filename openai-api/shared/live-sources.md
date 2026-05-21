# Live Documentation Sources

Use these when the cached information in this skill looks stale, when the user asks about features not covered here, or when verifying current model availability and pricing.

## Important: `platform.openai.com` blocks bots

As of May 2026, direct `WebFetch` against `platform.openai.com` returns **HTTP 403 Forbidden** to the WebFetch tool. Same for `openai.com`. To work around this:

1. **Use WebSearch instead** with `site:platform.openai.com` qualifiers — search snippets contain the key facts you need.
2. **Fetch the SDK source on GitHub** — the authoritative source for field names, request shapes, exception classes, and example code. GitHub URLs work fine with WebFetch.

---

## SDK Source on GitHub (preferred for verification)

The fastest way to verify "does this field exist?" / "what does this method return?" is to read the SDK source. WebFetch works against these.

### Python (`openai-python`)
- README: https://github.com/openai/openai-python/blob/main/README.md
- Type catalog: https://github.com/openai/openai-python/blob/main/api.md
- Responses resource: https://github.com/openai/openai-python/blob/main/src/openai/resources/responses/api.md
- Type definitions: https://github.com/openai/openai-python/tree/main/src/openai/types/responses
- Exception classes: https://github.com/openai/openai-python/blob/main/src/openai/_exceptions.py
- Examples (Responses): https://github.com/openai/openai-python/tree/main/examples/responses
- Examples (audio): https://github.com/openai/openai-python/blob/main/examples/audio.py

### Node / TypeScript (`openai-node`)
- README: https://github.com/openai/openai-node/blob/master/README.md
- Type catalog: https://github.com/openai/openai-node/blob/master/api.md
- Responses resource: https://github.com/openai/openai-node/blob/master/src/resources/responses/api.md
- Error classes: https://github.com/openai/openai-node/blob/master/src/core/error.ts
- Examples (Responses): https://github.com/openai/openai-node/tree/master/examples/responses

### C# / .NET (`openai-dotnet`)
- README: https://github.com/openai/openai-dotnet/blob/main/README.md
- Responses examples: https://github.com/openai/openai-dotnet/tree/main/examples/Responses
- NuGet: https://www.nuget.org/packages/OpenAI

### Java (`openai-java`)
- README: https://github.com/openai/openai-java/blob/main/README.md
- Error classes: https://github.com/openai/openai-java/tree/main/openai-java-core/src/main/kotlin/com/openai/errors
- Examples: https://github.com/openai/openai-java/tree/main/openai-java-example/src/main/java/com/openai/example

### Cookbook (cross-language patterns)
- https://github.com/openai/openai-cookbook
- Responses API examples: https://github.com/openai/openai-cookbook/tree/main/examples/responses_api

---

## Platform docs (use WebSearch, not WebFetch)

These pages are the canonical reference but cannot be directly fetched. Use `WebSearch` with a `site:platform.openai.com` qualifier and read the result snippets.

| Topic                          | Page URL                                                                  |
| ------------------------------ | ------------------------------------------------------------------------- |
| Responses API reference        | https://platform.openai.com/docs/api-reference/responses                  |
| Streaming events reference     | https://platform.openai.com/docs/api-reference/responses-streaming        |
| Migrate to Responses API       | https://platform.openai.com/docs/guides/migrate-to-responses              |
| Tools (built-in + functions)   | https://platform.openai.com/docs/guides/tools                             |
| Web search tool                | https://platform.openai.com/docs/guides/tools-web-search                  |
| File search tool               | https://platform.openai.com/docs/guides/tools-file-search                 |
| Code interpreter               | https://platform.openai.com/docs/guides/tools-code-interpreter            |
| Image generation               | https://platform.openai.com/docs/guides/image-generation                  |
| Computer use                   | https://platform.openai.com/docs/guides/tools-computer-use                |
| Remote MCP                     | https://platform.openai.com/docs/guides/tools-remote-mcp                  |
| Apply patch                    | https://platform.openai.com/docs/guides/tools-apply-patch                 |
| Function calling guide         | https://platform.openai.com/docs/guides/function-calling                  |
| Structured outputs             | https://platform.openai.com/docs/guides/structured-outputs                |
| Reasoning models               | https://platform.openai.com/docs/guides/reasoning                         |
| Conversation state             | https://platform.openai.com/docs/guides/conversation-state                |
| Prompt caching                 | https://platform.openai.com/docs/guides/prompt-caching                    |
| Rate limits                    | https://platform.openai.com/docs/guides/rate-limits                       |
| Models index                   | https://platform.openai.com/docs/models                                   |
| Pricing                        | https://platform.openai.com/docs/pricing                                  |
| Speech-to-text                 | https://platform.openai.com/docs/guides/speech-to-text                    |
| Text-to-speech                 | https://platform.openai.com/docs/guides/text-to-speech                    |
| Audio API reference            | https://platform.openai.com/docs/api-reference/audio                      |
| Realtime API (out of scope)    | https://platform.openai.com/docs/guides/realtime                          |
| Conversations API              | https://platform.openai.com/docs/api-reference/conversations              |

---

## How to use WebSearch for OpenAI docs

When the user asks something not covered here:

```
WebSearch("OpenAI Responses API <feature> site:platform.openai.com")
WebSearch("openai-python <method or class>")
WebSearch("OpenAI <model id> pricing 2026")
```

Read the snippets — they often contain the key parameter names and shapes. If a snippet shows an interesting URL on `developers.openai.com`, `cookbook.openai.com`, or a community blog, try WebFetch on it; those sometimes work.

For the *current* model list and pricing in a fresh way, the most reliable strategy is:

1. WebFetch `https://github.com/openai/openai-python/blob/main/README.md` — note which model is used in examples.
2. WebSearch for that model id plus "pricing" to find the current rate.

---

## Status & incident pages

When debugging mysterious 500s / 503s, check:

- https://status.openai.com — official status page. Often the first to confirm an incident.
- https://community.openai.com — developer community; recent threads often surface known issues.
