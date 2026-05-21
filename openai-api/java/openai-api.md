# Java — OpenAI Responses API

Official SDK: **`com.openai:openai-java`** (v4.36+ as of May 2026). Works on Java 8 and up; Kotlin and Scala consume the same artifact.

For conceptual background read SKILL.md "Architecture" and the relevant `shared/*.md` files. For secret handling see `../shared/secrets.md` — read BEFORE writing key-loading code.

---

## Install

### Maven

```xml
<dependency>
  <groupId>com.openai</groupId>
  <artifactId>openai-java</artifactId>
  <version>4.36.0</version>
</dependency>
```

### Gradle

```kotlin
implementation("com.openai:openai-java:4.36.0")
```

Optional: `io.github.cdimascio:dotenv-java` for `.env` file loading in local dev.

---

## Imports

```java
import com.openai.client.OpenAIClient;
import com.openai.client.OpenAIClientAsync;
import com.openai.client.okhttp.OpenAIOkHttpClient;
import com.openai.client.okhttp.OpenAIOkHttpClientAsync;
import com.openai.core.RequestOptions;
import com.openai.core.http.StreamResponse;
import com.openai.models.ChatModel;
import com.openai.models.responses.ResponseCreateParams;
import com.openai.models.responses.ResponseInputItem;
import com.openai.models.responses.ResponseStreamEvent;
import com.openai.models.responses.ResponseFunctionToolCall;
import com.openai.models.responses.Response;
```

---

## Authentication

The OkHttp clients read `OPENAI_API_KEY`, `OPENAI_ORG_ID`, `OPENAI_PROJECT_ID`, `OPENAI_BASE_URL` from the environment when you use `fromEnv()`. Never hardcode — see `../shared/secrets.md`.

```java
import io.github.cdimascio.dotenv.Dotenv;

Dotenv dotenv = Dotenv.configure().ignoreIfMissing().load();   // local dev only

OpenAIClient client = OpenAIOkHttpClient.fromEnv();             // reads OPENAI_API_KEY
OpenAIClientAsync clientAsync = OpenAIOkHttpClientAsync.fromEnv();
```

Or explicit builder (still reading env, NOT hardcoding):

```java
OpenAIClient client = OpenAIOkHttpClient.builder()
    .apiKey(System.getenv("OPENAI_API_KEY"))
    .organization(System.getenv("OPENAI_ORG_ID"))
    .project(System.getenv("OPENAI_PROJECT_ID"))
    .baseUrl("https://api.openai.com/v1")
    .maxRetries(4)
    .build();
```

For Spring Boot, prefer `@Value("${OPENAI_API_KEY}")` injected from `application-local.yml` (gitignored) or Spring Cloud Config / Vault in production. Don't put the key in committed `application.properties`.

---

## Minimal Responses Call

```java
OpenAIClient client = OpenAIOkHttpClient.fromEnv();

ResponseCreateParams params = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .instructions("You are a coding assistant that talks like a pirate.")
    .input("How do I check for null in Java?")
    .build();

Response response = client.responses().create(params);

response.output().stream()
    .flatMap(item -> item.message().stream())
    .flatMap(message -> message.content().stream())
    .flatMap(content -> content.outputText().stream())
    .forEach(o -> System.out.println(o.text()));
```

The fluent `flatMap` chain navigates the typed item hierarchy. For a quicker accessor, the response also exposes `outputText()` directly (see SDK version).

### Multi-turn

```java
Response r1 = client.responses().create(params);

ResponseCreateParams params2 = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .previousResponseId(r1.id())
    .input("What did I just tell you?")
    .build();

Response r2 = client.responses().create(params2);
```

### Multi-modal input

```java
ResponseCreateParams params = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input(ResponseCreateParams.Input.ofResponse(List.of(
        ResponseInputItem.ofMessage(ResponseInputItem.Message.builder()
            .role(ResponseInputItem.Message.Role.USER)
            .addInputTextContent("What's in this image?")
            .addInputImageContent("https://example.com/cat.jpg")
            .build())
    )))
    .build();
```

(Builder method names may vary; check `ResponseInputItem.Message.builder()` in your SDK version.)

---

## Async

`OpenAIClientAsync` returns `CompletableFuture`:

```java
OpenAIClientAsync client = OpenAIOkHttpClientAsync.fromEnv();

client.responses().create(params)
    .thenAccept(response -> response.output().stream()
        .flatMap(item -> item.message().stream())
        .flatMap(message -> message.content().stream())
        .flatMap(content -> content.outputText().stream())
        .forEach(o -> System.out.println(o.text())))
    .join();
```

---

## Streaming

`StreamResponse<ResponseStreamEvent>` is `AutoCloseable` — use try-with-resources.

```java
try (StreamResponse<ResponseStreamEvent> stream =
        client.responses().createStreaming(params)) {
    stream.stream()
        .flatMap(event -> event.outputTextDelta().stream())
        .forEach(t -> System.out.print(t.delta()));
}
```

Pattern-match on the event subtypes via `event.<eventType>()` Optionals. Some common ones:

- `event.outputTextDelta()` — text fragment with `.delta()`.
- `event.outputItemAdded()` — new item appended; `.item()` carries the partial.
- `event.functionCallArgumentsDelta()` — `.delta()` is a JSON string fragment.
- `event.functionCallArgumentsDone()` — `.arguments()` is the full JSON string.
- `event.completed()` — `.response()` is the final Response object with `usage`.
- `event.responseReasoningTextDelta()` / `.responseReasoningSummaryTextDelta()` — for reasoning models.

For async streams, use `OpenAIClientAsync.responses().createStreaming(params)` which returns `AsyncStreamResponse<ResponseStreamEvent>`:

```java
client.responses().createStreaming(params).subscribe(event -> {
    event.outputTextDelta().ifPresent(t -> System.out.print(t.delta()));
}).onCompleteFuture().join();
```

---

## Tool Use

The Java SDK supports two styles: **raw JSON schema** or **typed Java classes with Jackson annotations** (the SDK auto-derives the schema and parses arguments back into the class).

### Typed-class style (recommended)

```java
import com.fasterxml.jackson.annotation.JsonClassDescription;
import com.fasterxml.jackson.annotation.JsonPropertyDescription;

@JsonClassDescription("Return inventory details for a SKU.")
static class GetSkuInventory {
    @JsonPropertyDescription("Stock-keeping unit identifier")
    public String sku;

    public Map<String, Object> execute() {
        // your business logic
        return Map.of("sku", sku, "in_stock", 42, "price_usd", 12.99);
    }
}

ResponseCreateParams.Builder b = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input("What's the price of sku-froge-lily-pad-deluxe?")
    .addTool(GetSkuInventory.class);

Response response = client.responses().create(b.build());

// Agent loop
List<ResponseInputItem> inputs = new ArrayList<>();
inputs.add(ResponseInputItem.ofMessage(
    ResponseInputItem.Message.builder()
        .role(ResponseInputItem.Message.Role.USER)
        .addInputTextContent("What's the price of sku-froge-lily-pad-deluxe?")
        .build()));

while (response.output().stream().anyMatch(it -> it.isFunctionCall())) {
    for (var item : response.output()) {
        if (item.isFunctionCall()) {
            ResponseFunctionToolCall call = item.asFunctionCall();
            inputs.add(ResponseInputItem.ofFunctionCall(call));

            // Parse arguments into the typed class:
            GetSkuInventory args = call.arguments(GetSkuInventory.class);
            Object result = args.execute();

            inputs.add(ResponseInputItem.ofFunctionCallOutput(
                ResponseInputItem.FunctionCallOutput.builder()
                    .callId(call.callId())
                    .outputAsJson(result)             // SDK serializes via Jackson
                    .build()));
        }
    }

    response = client.responses().create(ResponseCreateParams.builder()
        .model("gpt-5.2")
        .previousResponseId(response.id())
        .input(ResponseCreateParams.Input.ofResponse(inputs))
        .build());
}

System.out.println(response.outputText());
```

Notes:
- **`call.arguments(MyClass.class)`** deserializes the JSON args into your typed class. Use this — it's safer than manual parsing.
- **Match by `callId()`**, not the item id.
- **`outputAsJson(obj)`** serializes via Jackson — return a `Map`, a record, a POJO with Jackson annotations.

### Raw JSON schema style

```java
ResponseCreateParams.Builder b = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input("...")
    .addTool(FunctionTool.builder()
        .name("get_sku_inventory")
        .description("Return inventory details for a SKU.")
        .parameters(JsonValue.from(Map.of(
            "type", "object",
            "properties", Map.of(
                "sku", Map.of("type", "string", "description", "SKU identifier")),
            "required", List.of("sku"),
            "additionalProperties", false)))
        .strict(true)
        .build());
```

The typed-class style is almost always cleaner; reach for raw JSON only when you need dynamic schemas (e.g., user-defined tools at runtime).

---

## Structured Outputs

Use either raw JSON schema:

```java
ResponseCreateParams params = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input("solve 8x + 31 = 2")
    .text(ResponseTextOptions.builder()
        .format(ResponseFormatJsonSchema.builder()
            .name("math_response")
            .schema(JsonValue.from(Map.of(
                "type", "object",
                "properties", Map.of(
                    "steps", Map.of("type", "array",
                        "items", Map.of("type", "object",
                            "properties", Map.of(
                                "explanation", Map.of("type", "string"),
                                "output", Map.of("type", "string")),
                            "required", List.of("explanation", "output"),
                            "additionalProperties", false)),
                    "final_answer", Map.of("type", "string")),
                "required", List.of("steps", "final_answer"),
                "additionalProperties", false)))
            .strict(true)
            .build())
        .build())
    .build();
```

…or a typed class with Jackson (the SDK derives the schema):

```java
static class MathResponse {
    public List<Step> steps;
    public String finalAnswer;
}
static class Step {
    public String explanation;
    public String output;
}

ResponseCreateParams params = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input("solve 8x + 31 = 2")
    .textFormat(MathResponse.class)
    .build();

Response response = client.responses().create(params);
MathResponse parsed = response.outputAs(MathResponse.class);
System.out.println(parsed.finalAnswer);
```

(Exact builder methods may differ across SDK versions — check the `examples/` folder in the openai-java repo for current syntax.)

The `strict: true` schema rules apply — see `../shared/structured-outputs.md`.

---

## Reasoning

```java
ResponseCreateParams params = ResponseCreateParams.builder()
    .model("gpt-5.2")
    .input("Prove that there are infinitely many primes.")
    .reasoning(ResponseReasoning.builder()
        .effort(ResponseReasoning.Effort.HIGH)
        .summary(ResponseReasoning.Summary.AUTO)
        .build())
    .build();

Response response = client.responses().create(params);
System.out.println("reasoning tokens: " +
    response.usage().outputTokensDetails().reasoningTokens());
```

Effort levels: `NONE`, `MINIMAL`, `LOW`, `MEDIUM`, `HIGH`, `XHIGH`. Defaults per-model — see SKILL.md.

---

## Errors

The Java SDK has a proper per-status-code exception hierarchy (unlike .NET):

```
OpenAIException                          (extends RuntimeException)
├── OpenAIServiceException               (base for HTTP failures)
│   ├── BadRequestException              400
│   ├── UnauthorizedException            401   (note: NOT AuthenticationException)
│   ├── PermissionDeniedException        403
│   ├── NotFoundException                404
│   ├── UnprocessableEntityException     422
│   ├── RateLimitException               429
│   ├── InternalServerException          5xx
│   └── UnexpectedStatusCodeException    other status codes
├── OpenAIIoException                    network I/O failures
├── OpenAIRetryableException             generic retryable failure
├── OpenAIInvalidDataException           parse / schema failures
├── SseException                         SSE streaming errors after a 2xx
├── SubjectTokenProviderException
└── InvalidWebhookSignatureException
```

The 401 class is called `UnauthorizedException`, NOT `AuthenticationException` — minor naming divergence from Python / Node.

```java
try {
    client.responses().create(params);
} catch (RateLimitException e) {
    // 429 — respect retry-after, backoff
    System.err.println("rate limited: status=" + e.statusCode());
} catch (UnauthorizedException e) {
    // 401 — fix key
    throw e;
} catch (OpenAIServiceException e) {
    System.err.println("API error: " + e.statusCode() + " " + e.getMessage());
    throw e;
} catch (OpenAIIoException e) {
    // network problem
    throw e;
}
```

Each `OpenAIServiceException` exposes `statusCode()`, `body()`, `headers()`, `requestId()`.

For HTTP-level details see `../shared/error-codes.md`.

---

## Retries & Timeouts

Defaults: **2 retries**, **10-minute timeout**.

Override on the builder:

```java
OpenAIClient client = OpenAIOkHttpClient.builder()
    .fromEnv()
    .maxRetries(4)
    .build();
```

Per-request timeout:

```java
client.responses().create(
    params,
    RequestOptions.builder().timeout(Duration.ofSeconds(30)).build());
```

Additional headers / query params per request:

```java
ResponseCreateParams params = ResponseCreateParams.builder()
    .putAdditionalHeader("X-Custom-Header", "value")
    .putAdditionalQueryParam("custom_param", "value")
    .build();
```

---

## Audio

```java
// Transcription
TranscriptionCreateParams params = TranscriptionCreateParams.builder()
    .file(Path.of("meeting.mp3"))
    .model("gpt-4o-transcribe")
    .language("en")
    .responseFormat(TranscriptionCreateParams.ResponseFormat.VERBOSE_JSON)
    .timestampGranularities(List.of(
        TranscriptionCreateParams.TimestampGranularity.SEGMENT,
        TranscriptionCreateParams.TimestampGranularity.WORD))
    .temperature(0f)
    .build();

Transcription tx = client.audio().transcriptions().create(params);
System.out.println(tx.text());

// TTS
SpeechCreateParams ttsParams = SpeechCreateParams.builder()
    .model("gpt-4o-mini-tts")
    .voice(SpeechCreateParams.Voice.MARIN)
    .input("The quick brown fox jumps over the lazy dog.")
    .instructions("Cheerful, warm, and a bit playful.")
    .responseFormat(SpeechCreateParams.ResponseFormat.MP3)
    .speed(1.0)
    .build();

byte[] audio = client.audio().speech().create(ttsParams).readAllBytes();
Files.write(Path.of("speech.mp3"), audio);
```

(Builder names may differ slightly across SDK versions — consult the openai-java README.)

Voices: `ALLOY`, `ASH`, `BALLAD`, `CORAL`, `ECHO`, `SAGE`, `SHIMMER`, `VERSE`, `MARIN`, `CEDAR`. `MARIN` and `CEDAR` are newest and recommended.

For the wire-level reference (constraints, gotchas), see the Python `audio.md` — fields and limits are identical.

---

## Common Pitfalls

- **`UnauthorizedException`, not `AuthenticationException`** for 401. Catching `AuthenticationException` won't compile.
- **`StreamResponse` is `AutoCloseable`.** Always wrap in try-with-resources or you leak the HTTP connection.
- **`call.arguments(MyClass.class)`** is safer than manually parsing — use it.
- **Match by `callId()`, not `id()`** on function call items.
- **Spring Boot:** don't put `OPENAI_API_KEY` in committed `application.properties`. Use `application-local.yml` (gitignored), user-supplied env, or Spring Cloud Config / Vault.
- **OkHttp clients are thread-safe** — instantiate once at startup, reuse across requests.
- **Default timeout is 10 minutes** — fine for streaming; non-streaming `gpt-5.5` deep reasoning runs can occasionally exceed it.
