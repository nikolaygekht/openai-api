# cURL / Raw HTTP — OpenAI Responses API

Use this when no official SDK is available for your language (Go, Ruby, PHP, Rust, Swift, etc.) or when scripting tests against the API. The endpoints, request shapes, and response shapes are identical to what the SDKs send under the hood.

For secret handling see `../shared/secrets.md` — the value of `$OPENAI_API_KEY` should come from an env var, never paste a literal key on a shell command line (it ends up in shell history).

---

## Setup

```bash
# Set the env var once — in ~/.zshrc / ~/.bashrc for personal dev,
# or via `set -a; source .env; set +a` for project-local .env
export OPENAI_API_KEY=$(grep '^OPENAI_API_KEY=' .env | cut -d= -f2-)

# Test:
echo "key set: ${OPENAI_API_KEY:0:7}…"   # redacted display only — never echo full value
```

---

## Endpoints

| Operation                            | Method | URL                                     |
| ------------------------------------ | ------ | --------------------------------------- |
| Create response                      | POST   | `https://api.openai.com/v1/responses`   |
| Retrieve response                    | GET    | `https://api.openai.com/v1/responses/{id}` |
| Delete response                      | DELETE | `https://api.openai.com/v1/responses/{id}` |
| Cancel response                      | POST   | `https://api.openai.com/v1/responses/{id}/cancel` |
| List response input items            | GET    | `https://api.openai.com/v1/responses/{id}/input_items` |
| Conversations                        | POST   | `https://api.openai.com/v1/conversations` |
| Audio: transcription                 | POST   | `https://api.openai.com/v1/audio/transcriptions` |
| Audio: translation (→ English)        | POST   | `https://api.openai.com/v1/audio/translations` |
| Audio: TTS                           | POST   | `https://api.openai.com/v1/audio/speech` |
| Models: list                         | GET    | `https://api.openai.com/v1/models`      |

---

## Minimal Responses Call

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "instructions": "You are a coding assistant that talks like a pirate.",
    "input": "How do I check if a Python object is an instance of a class?"
  }'
```

Response shape (abbreviated):

```json
{
  "id": "resp_abc123",
  "object": "response",
  "created_at": 1716239021.5,
  "status": "completed",
  "model": "gpt-5.2",
  "output": [
    {
      "type": "message",
      "role": "assistant",
      "content": [
        { "type": "output_text", "text": "Arr matey, ye use `isinstance(obj, Cls)`…" }
      ]
    }
  ],
  "usage": {
    "input_tokens": 42,
    "input_tokens_details": { "cached_tokens": 0 },
    "output_tokens": 87,
    "output_tokens_details": { "reasoning_tokens": 0 },
    "total_tokens": 129
  }
}
```

For just the assistant text, pipe through `jq`:

```bash
curl ... | jq -r '.output[] | select(.type=="message") | .content[] | select(.type=="output_text") | .text'
```

---

## Multi-Modal Input (vision)

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "input": [{
      "role": "user",
      "content": [
        {"type": "input_text", "text": "What is in this image?"},
        {"type": "input_image", "image_url": "https://example.com/cat.jpg"}
      ]
    }]
  }'
```

For a base64 data URL:

```bash
B64=$(base64 -w0 < cat.jpg)
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gpt-5.2\",
    \"input\": [{
      \"role\": \"user\",
      \"content\": [
        {\"type\": \"input_text\", \"text\": \"What is in this image?\"},
        {\"type\": \"input_image\", \"image_url\": \"data:image/jpeg;base64,$B64\"}
      ]
    }]
  }"
```

---

## Multi-Turn Conversation (`previous_response_id`)

```bash
# Turn 1:
RESP1=$(curl -s https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model": "gpt-5.2", "input": "Hi, my name is Sam."}')

R1_ID=$(echo "$RESP1" | jq -r .id)

# Turn 2:
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{
    \"model\": \"gpt-5.2\",
    \"previous_response_id\": \"$R1_ID\",
    \"input\": \"What did I just tell you?\"
  }"
```

`previous_response_id` requires the prior response was stored (`store: true`, the default).

---

## Streaming (SSE)

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "input": "Write a haiku about caching.",
    "stream": true
  }' \
  --no-buffer
```

You get a Server-Sent Events stream:

```
event: response.created
data: {"type":"response.created","response":{...}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","delta":"Mem"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","delta":"ory"}

...

event: response.completed
data: {"type":"response.completed","response":{"usage":{...}}}
```

For programmatic parsing in shell, use a tool like `openai-stream` parser or read line-by-line and accumulate `data:` lines.

---

## Function Tool Use

Initial request — declare the tool, the model emits a `function_call`:

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "input": "What is the weather in Tokyo?",
    "tools": [{
      "type": "function",
      "name": "get_current_weather",
      "description": "Get the current weather in a given location",
      "parameters": {
        "type": "object",
        "properties": {
          "location": {"type": "string"},
          "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
        },
        "required": ["location", "unit"],
        "additionalProperties": false
      },
      "strict": true
    }]
  }'
```

Response contains a function_call item:

```json
{
  "id": "resp_xyz",
  "output": [{
    "type": "function_call",
    "call_id": "call_abc",
    "name": "get_current_weather",
    "arguments": "{\"location\":\"Tokyo\",\"unit\":\"celsius\"}",
    "status": "completed"
  }]
}
```

Follow-up — execute the function locally, then send the result back keyed by `call_id`:

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "previous_response_id": "resp_xyz",
    "input": [{
      "type": "function_call_output",
      "call_id": "call_abc",
      "output": "{\"temperature_c\": 22, \"conditions\": \"clear\"}"
    }]
  }'
```

Loop until the response has no `function_call` items.

---

## Structured Outputs (JSON Schema)

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "input": "solve 8x + 31 = 2",
    "text": {
      "format": {
        "type": "json_schema",
        "name": "math_response",
        "strict": true,
        "schema": {
          "type": "object",
          "properties": {
            "steps": {
              "type": "array",
              "items": {
                "type": "object",
                "properties": {
                  "explanation": {"type": "string"},
                  "output": {"type": "string"}
                },
                "required": ["explanation", "output"],
                "additionalProperties": false
              }
            },
            "final_answer": {"type": "string"}
          },
          "required": ["steps", "final_answer"],
          "additionalProperties": false
        }
      }
    }
  }'
```

The `output_text` content part contains the JSON string. Parse with `jq`:

```bash
curl ... | jq -r '.output[].content[]? | select(.type=="output_text") | .text' | jq .
```

Notes:
- `text.format` is **flat** — there's no `json_schema:` wrapper as in Chat Completions.
- `strict: true` requires the JSON-schema subset (every property in `required`, `additionalProperties: false` everywhere). See `../shared/structured-outputs.md`.

---

## Reasoning

```bash
curl https://api.openai.com/v1/responses \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.2",
    "input": "Prove that there are infinitely many primes.",
    "reasoning": {"effort": "high", "summary": "auto"}
  }'
```

The response includes a `reasoning` item in `output` with `summary` text, and `usage.output_tokens_details.reasoning_tokens` records the hidden token count.

---

## Built-in Tools

```bash
# Web search:
-d '{"model":"gpt-5.2", "input":"...", "tools":[{"type":"web_search","search_context_size":"medium"}]}'

# File search:
-d '{"model":"gpt-5.2", "input":"...", "tools":[{"type":"file_search","vector_store_ids":["vs_abc"]}]}'

# Code interpreter:
-d '{"model":"gpt-5.2", "input":"...", "tools":[{"type":"code_interpreter","container":{"type":"auto"}}]}'

# Image generation:
-d '{"model":"gpt-5.2", "input":"...", "tools":[{"type":"image_generation","model":"gpt-image-1.5","size":"1024x1024"}]}'

# MCP:
-d '{"model":"gpt-5.2", "input":"...", "tools":[{"type":"mcp","server_label":"my","server_url":"https://...","require_approval":"never"}]}'
```

---

## Audio: Transcription

```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: multipart/form-data" \
  -F file="@meeting.mp3" \
  -F model="gpt-4o-transcribe" \
  -F language="en" \
  -F response_format="verbose_json" \
  -F "timestamp_granularities[]=word" \
  -F "timestamp_granularities[]=segment"
```

Diarization:

```bash
curl https://api.openai.com/v1/audio/transcriptions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F file="@interview.wav" \
  -F model="gpt-4o-transcribe-diarize" \
  -F response_format="diarized_json" \
  -F chunking_strategy="auto"
```

File size: 25 MB max. Formats: `flac mp3 mp4 mpeg mpga m4a ogg wav webm`.

---

## Audio: Translation (any → English)

```bash
curl https://api.openai.com/v1/audio/translations \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -F file="@spanish.m4a" \
  -F model="whisper-1" \
  -F response_format="text"
```

Only `whisper-1`. Output always English. No `language` param.

---

## Audio: TTS

```bash
curl https://api.openai.com/v1/audio/speech \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o-mini-tts",
    "voice": "marin",
    "input": "The quick brown fox jumps over the lazy dog.",
    "instructions": "Cheerful, warm, and a bit playful.",
    "response_format": "mp3",
    "speed": 1.0
  }' \
  --output speech.mp3
```

`input` max 4096 characters. Voices: `alloy ash ballad coral echo sage shimmer verse marin cedar`. `instructions` is ignored on `tts-1` / `tts-1-hd`.

---

## Models: List

```bash
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $OPENAI_API_KEY"
```

Returns minimal metadata only — no `context_window`, no capability flags. See `../shared/models.md` for the full capability table.

---

## Reading Response Headers

```bash
curl -i https://api.openai.com/v1/responses ...
# or:
curl -D /tmp/headers.txt https://api.openai.com/v1/responses ...
cat /tmp/headers.txt
```

Useful headers:
- `x-request-id` — log on every failure.
- `x-ratelimit-remaining-{requests,tokens}` and `x-ratelimit-reset-{requests,tokens}`.
- `retry-after` (on 429 / 503) — wait this many seconds before retrying.
- `openai-processing-ms` — server-side processing time.

---

## Common Pitfalls

- **Don't paste the key on the command line.** Use `$OPENAI_API_KEY` from env — otherwise it ends up in `~/.bash_history` forever.
- **Don't echo `$OPENAI_API_KEY`** — never print, never log.
- **`text.format` is flat** — no `json_schema: {}` envelope inside it.
- **Function tools are flat** — `{type:"function", name, description, parameters, strict}`, no `function: {...}` wrapper.
- **`arguments` is a JSON-encoded string**, not an object.
- **Function output is a string** in `function_call_output`. JSON-encode complex results.
- **Match by `call_id`, not `id`.**
- **`previous_response_id` requires the prior call had `store: true`** (the default).
- **For streaming, use `--no-buffer`** so chunks display as they arrive.
- **For multipart audio uploads, use `-F`** not `-d`.
- **25 MB audio limit is bytes, not minutes.**
- **TTS `input` is capped at 4096 characters.**
- **The Responses API does NOT accept `messages`** — use `input` (string or array of items).
- **The Responses API does NOT accept `response_format`** — use `text.format` instead.
