# C# / .NET — OpenAI Responses API

Official SDK: **`OpenAI`** NuGet package (v2.10+ as of May 2026). The Responses API surface is **experimental** — suppress the `OPENAI001` diagnostic to use it.

For conceptual background read the SKILL.md "Architecture" section and the relevant `shared/*.md` files. For secret handling see `../shared/secrets.md` — read BEFORE writing key-loading code.

---

## Install

```bash
dotnet add package OpenAI
# Optional for local .env loading:
dotnet add package DotNetEnv
```

Current NuGet version (April 2026): **2.10.0**. Target framework: `.NET 8` or `.NET Standard 2.1` and up.

```xml
<ItemGroup>
  <PackageReference Include="OpenAI" Version="2.10.0" />
</ItemGroup>
```

---

## Usings

```csharp
using OpenAI;                 // OpenAIClient (parent), OpenAIClientOptions
using OpenAI.Responses;       // ResponsesClient, ResponseResult, ResponseItem,
                              // CreateResponseOptions, StreamingResponseUpdate,
                              // ResponseTool, FunctionTool, FunctionCallResponseItem,
                              // ResponseReasoningOptions, ResponseReasoningEffortLevel
using OpenAI.Audio;           // AudioClient (transcription / translation / TTS)
using System.ClientModel;     // ApiKeyCredential, ClientResult, ClientResultException
```

Other available subclients (out of scope for this skill but available): `OpenAI.Chat` (legacy), `OpenAI.Embeddings`, `OpenAI.Images`, `OpenAI.Batch`, `OpenAI.Assistants`, `OpenAI.Models`.

---

## Suppress the Experimental Warning

The Responses surface is marked experimental:

```csharp
#pragma warning disable OPENAI001
using OpenAI.Responses;

ResponsesClient client = new(apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));
```

Or globally in your `.csproj`:

```xml
<NoWarn>OPENAI001</NoWarn>
```

This is what every official example does. The API is stable enough to ship; OpenAI just hasn't removed the `[Experimental]` attribute yet.

---

## Authentication

Read `OPENAI_API_KEY` from the environment. Never hardcode, never log — see `../shared/secrets.md`.

```csharp
using DotNetEnv;
Env.Load();                                                  // local dev only

ResponsesClient client = new(
    apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));
```

Or with explicit credential + custom endpoint:

```csharp
ResponsesClient client = new(
    credential: new ApiKeyCredential(Environment.GetEnvironmentVariable("OPENAI_API_KEY")!),
    options: new OpenAIClientOptions { Endpoint = new Uri("https://api.openai.com/v1") });
```

For ASP.NET production use, prefer the user-secrets / Key Vault providers:

```csharp
builder.Services.AddSingleton(_ =>
    new ResponsesClient(apiKey: builder.Configuration["OpenAI:ApiKey"]));
```

…with the value supplied via `dotnet user-secrets` in development and Azure Key Vault (or equivalent) in production. Don't put `OpenAI:ApiKey` in `appsettings.json` — that file is committed.

You can also instantiate the top-level `OpenAIClient` once and grab subclients:

```csharp
OpenAIClient root = new(apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));
ResponsesClient responses = root.GetResponsesClient();
AudioClient audio = root.GetAudioClient("gpt-4o-transcribe");   // audio is per-model
```

---

## Minimal Responses Call

```csharp
ResponsesClient client = new(apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));
ResponseResult response = client.CreateResponse(
    model: "gpt-5.2",
    userInputText: "Say 'this is a test.'");
Console.WriteLine($"[ASSISTANT]: {response.GetOutputText()}");
```

Or with full options for anything beyond a single user message:

```csharp
CreateResponseOptions options = new()
{
    Model = "gpt-5.2",
    Instructions = "You are a coding assistant that talks like a pirate.",
};
options.InputItems.Add(ResponseItem.CreateUserMessageItem(
    "How do I check if a value is null in C#?"));

ResponseResult response = await client.CreateResponseAsync(options);
Console.WriteLine(response.GetOutputText());
```

### Multi-modal input

```csharp
CreateResponseOptions options = new() { Model = "gpt-5.2" };
options.InputItems.Add(
    ResponseItem.CreateUserMessageItem(new List<ResponseContentPart>
    {
        ResponseContentPart.CreateInputTextPart("What's in this image?"),
        ResponseContentPart.CreateInputImagePart(new Uri("https://example.com/cat.jpg")),
    }));
```

### Multi-turn

```csharp
ResponseResult r1 = await client.CreateResponseAsync(options1);

CreateResponseOptions options2 = new()
{
    Model = "gpt-5.2",
    PreviousResponseId = r1.Id,
};
options2.InputItems.Add(ResponseItem.CreateUserMessageItem("What did I just tell you?"));
ResponseResult r2 = await client.CreateResponseAsync(options2);
```

---

## Async

Every sync method has an `Async` twin (e.g. `CreateResponse` → `CreateResponseAsync`, `CreateResponseStreaming` → `CreateResponseStreamingAsync`). There is no separate async client type.

---

## Streaming

```csharp
CollectionResult<StreamingResponseUpdate> updates =
    client.CreateResponseStreaming("gpt-5.2", "Write a haiku about caching.");

foreach (StreamingResponseUpdate update in updates)
{
    if (update is StreamingResponseOutputTextDeltaUpdate delta)
        Console.Write(delta.Delta);
}
```

Async variant returns `IAsyncEnumerable`:

```csharp
await foreach (StreamingResponseUpdate update in
    client.CreateResponseStreamingAsync(options))
{
    switch (update)
    {
        case StreamingResponseOutputTextDeltaUpdate delta:
            Console.Write(delta.Delta);
            break;
        case StreamingResponseCompletedUpdate done:
            Console.WriteLine($"\n[tokens: {done.Response.Usage.TotalTokens}]");
            break;
    }
}
```

Pattern-match on the `StreamingResponseUpdate` subtypes:
- `StreamingResponseOutputTextDeltaUpdate` → `.Delta` (text fragment)
- `StreamingResponseFunctionCallArgumentsDeltaUpdate` → `.Delta` (tool arg JSON fragment)
- `StreamingResponseOutputItemAddedUpdate` → new item appeared
- `StreamingResponseCompletedUpdate` → final response with full usage
- (and parallels for reasoning, file_search, web_search, code_interpreter, image_generation, MCP)

Event taxonomy is the same as Python / Node — see `../python/openai-api/streaming.md` "Event Catalog" for the full list; the C# class names mirror the event types.

---

## Tool Use

```csharp
FunctionTool getWeather = ResponseTool.CreateFunctionTool(
    functionName: "GetCurrentWeather",
    functionDescription: "Get the current weather in a given location",
    functionParameters: BinaryData.FromBytes("""
    {
      "type": "object",
      "properties": {
        "location": { "type": "string" },
        "unit":     { "type": "string", "enum": ["celsius", "fahrenheit"] }
      },
      "required": ["location"],
      "additionalProperties": false
    }
    """u8.ToArray()),
    strictModeEnabled: true);

List<ResponseItem> inputItems = [
    ResponseItem.CreateUserMessageItem("What's the weather in Tokyo?"),
];

CreateResponseOptions options = new()
{
    Model = "gpt-5.2",
    Tools = { getWeather },
};
foreach (var item in inputItems) options.InputItems.Add(item);

ResponseResult response = await client.CreateResponseAsync(options);

// Agent loop
while (response.OutputItems.Any(i => i is FunctionCallResponseItem))
{
    foreach (ResponseItem outputItem in response.OutputItems)
    {
        if (outputItem is FunctionCallResponseItem call)
        {
            using JsonDocument argsDoc = JsonDocument.Parse(call.FunctionArguments);
            string location = argsDoc.RootElement.GetProperty("location").GetString()!;
            string result = $"{{\"temperature_c\":24,\"location\":\"{location}\"}}";

            inputItems.Add(call);                                                            // echo the call back
            inputItems.Add(new FunctionCallOutputResponseItem(call.CallId, result));         // append output
        }
    }

    CreateResponseOptions next = new()
    {
        Model = "gpt-5.2",
        PreviousResponseId = response.Id,
    };
    foreach (var item in inputItems) next.InputItems.Add(item);

    response = await client.CreateResponseAsync(next);
}

Console.WriteLine(response.GetOutputText());
```

Notes:
- **Function call arguments** come as `BinaryData` / string — parse with `JsonDocument` or `JsonSerializer`.
- **Match outputs by `CallId`** (not `Id`).
- **Output value of `FunctionCallOutputResponseItem`** is a string (typically JSON).
- **`strictModeEnabled: true`** turns on constrained decoding — schema must follow the strict-mode subset (`additionalProperties: false`, every property in `required`, no unsupported keywords; see `../shared/structured-outputs.md`).

### Built-in tools

```csharp
options.Tools.Add(ResponseTool.CreateWebSearchTool(
    searchContextSize: ResponseWebSearchContextSize.Medium));

options.Tools.Add(ResponseTool.CreateFileSearchTool(
    vectorStoreIds: new[] { "vs_abc" },
    maxNumberOfResults: 5));

options.Tools.Add(ResponseTool.CreateCodeInterpreterTool());

options.Tools.Add(ResponseTool.CreateImageGenerationTool(
    model: "gpt-image-1.5",
    size: "1024x1024",
    quality: "high"));
```

Exact class names may vary slightly by SDK minor version — check `OpenAI.Responses.ResponseTool` for the available factory methods.

---

## Structured Outputs

The .NET SDK does not yet have a typed-class auto-derivation helper. Pass a JSON schema string:

```csharp
string schema = """
{
  "type": "object",
  "properties": {
    "steps": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "explanation": { "type": "string" },
          "output": { "type": "string" }
        },
        "required": ["explanation", "output"],
        "additionalProperties": false
      }
    },
    "final_answer": { "type": "string" }
  },
  "required": ["steps", "final_answer"],
  "additionalProperties": false
}
""";

CreateResponseOptions options = new()
{
    Model = "gpt-5.2",
    TextOptions = new ResponseTextOptions
    {
        Format = ResponseTextFormat.CreateJsonSchema(
            name: "math_response",
            jsonSchema: BinaryData.FromString(schema),
            strictModeEnabled: true),
    },
};
options.InputItems.Add(ResponseItem.CreateUserMessageItem("solve 8x + 31 = 2"));

ResponseResult response = await client.CreateResponseAsync(options);
using JsonDocument doc = JsonDocument.Parse(response.GetOutputText());
string answer = doc.RootElement.GetProperty("final_answer").GetString()!;
```

`strict: true` requires the JSON-schema subset — see `../shared/structured-outputs.md`.

---

## Reasoning

```csharp
CreateResponseOptions options = new()
{
    Model = "gpt-5.2",
    ReasoningOptions = new ResponseReasoningOptions
    {
        ReasoningEffortLevel = ResponseReasoningEffortLevel.High,
        SummaryMode = ResponseReasoningSummaryMode.Auto,
    },
};
options.InputItems.Add(ResponseItem.CreateUserMessageItem(
    "Prove that there are infinitely many primes."));

ResponseResult response = await client.CreateResponseAsync(options);
Console.WriteLine($"reasoning tokens: {response.Usage.OutputTokenDetails.ReasoningTokens}");
```

Effort levels: `None`, `Minimal`, `Low`, `Medium`, `High`, `XHigh`. `gpt-5.1` / `gpt-5.2` default to `None` — set explicitly for hard tasks.

---

## Errors

There is **one** exception type: `System.ClientModel.ClientResultException`. No per-status-code subclasses (this is the biggest divergence from Python / Node / Java).

```csharp
try
{
    var r = await client.CreateResponseAsync(options);
}
catch (ClientResultException ex)
{
    var raw = ex.GetRawResponse();
    string? requestId = raw.Headers.TryGetValue("x-request-id", out var v) ? v : null;
    Console.Error.WriteLine($"status={ex.Status} request_id={requestId} message={ex.Message}");

    switch (ex.Status)
    {
        case 400: /* BadRequest — fix payload */ throw;
        case 401: /* Authentication — check key */ throw;
        case 403: /* PermissionDenied */ throw;
        case 404: /* NotFound */ throw;
        case 429: /* RateLimit — backoff */ break;
        case >= 500: /* server — backoff */ break;
    }
}
```

For HTTP-level reference see `../shared/error-codes.md`.

---

## Retries & Timeouts

Defaults: **3 retries** (more aggressive than Python / Node's 2), **100s NetworkTimeout**. Retries fire on 408, 429, 500, 502, 503, 504.

Override via `OpenAIClientOptions`:

```csharp
OpenAIClientOptions opts = new()
{
    NetworkTimeout = TimeSpan.FromMinutes(10),
    // RetryPolicy = ... (custom ClientRetryPolicy if you really need to)
};
ResponsesClient client = new(
    credential: new ApiKeyCredential(Environment.GetEnvironmentVariable("OPENAI_API_KEY")!),
    options: opts);
```

The SDK exposes the underlying `ClientRetryPolicy` from `System.ClientModel.Primitives`. For most cases the defaults are fine.

---

## Reading Headers

```csharp
ClientResult<ResponseResult> result = client.CreateResponse(model: "gpt-5.2", userInputText: "hi");
var raw = result.GetRawResponse();
string? requestId = raw.Headers.TryGetValue("x-request-id", out var v) ? v : null;
string? cached = raw.Headers.TryGetValue("openai-processing-ms", out var p) ? p : null;
```

---

## Usage & Caching

```csharp
ResponseResult resp = await client.CreateResponseAsync(options);
resp.Usage.InputTokens;
resp.Usage.InputTokenDetails.CachedTokens;
resp.Usage.OutputTokens;
resp.Usage.OutputTokenDetails.ReasoningTokens;
resp.Usage.TotalTokens;
```

For caching strategy see `../shared/prompt-caching.md`.

---

## Audio

The audio surface lives under `OpenAI.Audio.AudioClient`. Each instance is bound to a model.

### Transcription

```csharp
AudioClient audio = new(
    model: "gpt-4o-transcribe",
    apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));

using FileStream audioStream = File.OpenRead("meeting.mp3");
AudioTranscription tx = await audio.TranscribeAudioAsync(
    audioStream,
    "meeting.mp3",
    new AudioTranscriptionOptions
    {
        Language = "en",
        ResponseFormat = AudioTranscriptionFormat.Verbose,
        TimestampGranularities = AudioTimestampGranularities.Word | AudioTimestampGranularities.Segment,
        Temperature = 0f,
    });

Console.WriteLine(tx.Text);
foreach (var seg in tx.Segments)
    Console.WriteLine($"[{seg.StartTime:s\\.ff}-{seg.EndTime:s\\.ff}] {seg.Text}");
```

### TTS

```csharp
AudioClient audio = new(model: "gpt-4o-mini-tts",
    apiKey: Environment.GetEnvironmentVariable("OPENAI_API_KEY"));

BinaryData speech = await audio.GenerateSpeechAsync(
    "The quick brown fox jumps over the lazy dog.",
    GeneratedSpeechVoice.Marin,
    new SpeechGenerationOptions
    {
        Instructions = "Cheerful, warm, and a bit playful.",
        ResponseFormat = GeneratedSpeechFormat.Mp3,
        Speed = 1.0f,
    });

await File.WriteAllBytesAsync("speech.mp3", speech.ToArray());
```

Voices: `Alloy`, `Ash`, `Ballad`, `Coral`, `Echo`, `Sage`, `Shimmer`, `Verse`, `Marin`, `Cedar`. `marin` and `cedar` are newest and recommended.

For streaming TTS use the streaming overloads on `AudioClient` (consult the API docs for exact method names in your SDK version).

For the full audio reference (parameters, gotchas), see the Python or TypeScript `audio.md` — the wire-level fields are identical, only the C# property names differ.

---

## Common Pitfalls

- **Don't forget `#pragma warning disable OPENAI001`** for Responses code.
- **Single exception class.** Catch `ClientResultException` and switch on `.Status` — there's no `RateLimitException` subclass to catch separately.
- **`FunctionArguments` is a string** (often `BinaryData`). Parse with `JsonDocument` / `JsonSerializer`.
- **Match outputs by `CallId`, not `Id`.**
- **Don't hardcode the API key** in `appsettings.json` or source. Use env / user-secrets / Key Vault.
- **Watch the experimental flag.** New Responses features ship under `OPENAI001` first; check the latest example folder when something seems missing.
- **Default `NetworkTimeout` is 100s** — much shorter than Python/Node's 10 min. For long generations bump it or stream.
- **No typed-class structured-output helper yet.** Hand-roll the JSON schema (or use a code generator).
- **Audio clients are per-model.** A `new AudioClient(model: "gpt-4o-transcribe", ...)` is for transcription; switching to TTS needs a new `AudioClient(model: "gpt-4o-mini-tts", ...)`.
