# Structured Outputs (Responses API)

Structured outputs constrain the model to emit JSON that exactly matches a schema you provide. The Responses API uses **`text.format`** (not the Chat Completions `response_format`). With `strict: true`, OpenAI runs constrained decoding so the output is guaranteed to validate.

---

## Request Shape

The `text.format` slot is a discriminated union on `type`:

```python
text = {"format": {"type": "text"}}                  # plain text (default)
text = {"format": {"type": "json_object"}}            # old JSON mode; avoid for new code
text = {"format": {                                   # structured outputs — preferred
    "type": "json_schema",
    "name": "weather_report",                         # required, [a-zA-Z0-9_-]{,64}
    "schema": {...},                                  # JSON schema
    "description": "...",                             # optional, model-visible
    "strict": True,                                   # required for guaranteed validation
}}
```

Full example:

```python
schema = {
    "type": "object",
    "properties": {
        "location": {"type": "string"},
        "temperature_c": {"type": "number"},
        "conditions": {"type": "string"},
    },
    "required": ["location", "temperature_c", "conditions"],
    "additionalProperties": False,
}

resp = client.responses.create(
    model="gpt-5.2",
    input="What's the weather in Tokyo?",
    text={"format": {"type": "json_schema", "name": "weather_report",
                     "schema": schema, "strict": True}},
)
import json
data = json.loads(resp.output_text)
```

---

## The `strict: true` Schema Subset

Constrained decoding only works on a JSON-schema subset. Violating any rule gets you a 400 at request time, not silent failure:

1. **Root must be an object** (`type: "object"`). Arrays and primitives at the root are not allowed — wrap them.
2. **Every property in `properties` must appear in `required`.** Optional fields are simulated by adding `"null"` to the property's `type`. There is no "optional in the JSON-schema sense."
3. **`additionalProperties: false`** must be set on every object (root and nested).
4. **No unsupported keywords**: `format`, `pattern`, `minimum`, `maximum`, `minLength`, `maxLength`, `minItems`, `maxItems`, `default`, `oneOf` are unsupported. Use `anyOf` instead of `oneOf`. Use enums instead of patterns.
5. **`enum` is fine**, including string enums.
6. **`anyOf` is allowed** but every branch must individually satisfy the subset rules.
7. **Recursive schemas via `$ref`** are allowed; OpenAI's parser unrolls them.
8. **Max nesting depth ~5**, max ~100 properties total per schema. Outsize schemas get rejected.

If the user's schema doesn't validate as `strict: true`, two options:

- Set `strict: false` — model still tries to match but no guarantee, and refusals or syntactically-broken output are possible.
- Refactor the schema to fit the subset.

---

## SDK Helpers

Python and TypeScript both provide helpers that auto-derive the schema from a typed class and parse the result back. **Always prefer these over hand-rolling the schema** — they get the strict-mode rules right.

### Python — `client.responses.parse(...)` with Pydantic

```python
from pydantic import BaseModel
from typing import List
from openai import OpenAI

class Step(BaseModel):
    explanation: str
    output: str

class MathResponse(BaseModel):
    steps: List[Step]
    final_answer: str

client = OpenAI()
rsp = client.responses.parse(
    model="gpt-5.2",
    input="solve 8x + 31 = 2",
    text_format=MathResponse,   # Pydantic class
)

# Iterate output items and use .parsed (typed Pydantic instance)
for output in rsp.output:
    if output.type == "message":
        for item in output.content:
            if item.type == "output_text" and item.parsed:
                print(item.parsed.final_answer)
```

The helper does three things: converts the Pydantic class to a strict JSON schema, sets `text.format` for you, and populates `.parsed` on each `output_text` content part with the validated Pydantic instance.

### TypeScript — `zodTextFormat` with Zod

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
const rsp = await client.responses.parse({
  model: 'gpt-5.2',
  input: 'solve 8x + 31 = 2',
  text: { format: zodTextFormat(MathResponse, 'math_response') },
});

// rsp.output_parsed is a typed MathResponse instance.
console.log(rsp.output_parsed.final_answer);
```

### C# (.NET)

The .NET SDK does not yet have a typed-class helper. Build the JSON schema by hand (or use `System.Text.Json.JsonSchema` to derive one), then pass it via `BinaryData`:

```csharp
string schema = """
{
  "type": "object",
  "properties": {
    "final_answer": { "type": "string" }
  },
  "required": ["final_answer"],
  "additionalProperties": false
}
""";

CreateResponseOptions options = new() {
    Model = "gpt-5.2",
    TextOptions = new ResponseTextOptions {
        Format = ResponseTextFormat.CreateJsonSchema(
            name: "math_response",
            jsonSchema: BinaryData.FromString(schema),
            strictModeEnabled: true),
    },
};
options.InputItems.Add(ResponseItem.CreateUserMessageItem("solve 8x + 31 = 2"));
ResponseResult response = client.CreateResponse(options);
```

### Java

```java
// Build the JSON schema as a Map or use Jackson on a typed class
ResponseFormatJsonSchema schema = ResponseFormatJsonSchema.builder()
    .name("math_response")
    .schema(JsonSchemaMap)   // your schema as a Map / String
    .strict(true)
    .build();

ResponseCreateParams params = ResponseCreateParams.builder()
    .model(ChatModel.GPT_5_2)
    .input("solve 8x + 31 = 2")
    .text(ResponseTextOptions.builder().format(schema).build())
    .build();
```

---

## Refusals

`strict: true` does NOT bypass safety. If the model refuses, the assistant message's `content` array will include a `{"type": "refusal", "refusal": "I can't help with that."}` part instead of (or alongside) `output_text`.

```python
for output in rsp.output:
    if output.type == "message":
        for item in output.content:
            if item.type == "refusal":
                print("Model refused:", item.refusal)
                return
            elif item.type == "output_text":
                # normal handling
                pass
```

Refusals are terminal. Retrying with the same prompt won't help. If the refusal is wrong, surface it to the user — don't loop.

---

## Function-Tool Schemas

The same strict-mode rules apply to function-tool `parameters` schemas (when `strict: true` is set on the tool). The SDK helpers `openai.pydantic_function_tool(MyModel)` (Python) and `zodResponsesFunction({name, parameters: MyZodSchema})` (TS) handle this for you. See `shared/tool-use-concepts.md` for the tool side.

---

## vs. Chat Completions

| Chat Completions                                  | Responses                                       |
| ------------------------------------------------- | ----------------------------------------------- |
| `response_format` (top-level)                     | `text.format` (nested under `text`)             |
| `{"type": "json_schema", "json_schema": {...}}`   | `{"type": "json_schema", ...flat...}`           |
| `json_schema.name` / `json_schema.schema`         | `text.format.name` / `text.format.schema`       |
| Helpers: `client.beta.chat.completions.parse(...)` | `client.responses.parse(...)`                  |

The single biggest mistake migrating from Chat Completions is leaving the `json_schema:` wrapper in place. On Responses it's flat.

---

## Output JSON Mode (`json_object`)

`text.format = {"type": "json_object"}` is the older mode where the model is told "return valid JSON" but no schema is enforced. It exists for backward compatibility. **Avoid it for new code** — use `json_schema` with `strict: true` instead. Even when you don't care about exact shape, a permissive schema + strict mode gives you parse-safety guarantees that `json_object` cannot.
