# TypeScript / Node — Audio APIs (Transcription, Translation, TTS)

Three endpoints, all on `client.audio.*`. These are **separate from the Responses API**. For pricing and model selection see `../../shared/models.md`. For secret handling see `../../shared/secrets.md`.

---

## Transcription (speech-to-text)

Endpoint: `/v1/audio/transcriptions`. Resource: `client.audio.transcriptions`.

### Models (May 2026)

- **`gpt-4o-transcribe`** — recommended default.
- **`gpt-4o-mini-transcribe`** — cheaper.
- **`gpt-4o-transcribe-diarize`** — speaker diarization.
- **`whisper-1`** — legacy. No streaming, no diarization.

### Minimal call

```ts
import fs from 'fs';
import OpenAI, { toFile } from 'openai';

const client = new OpenAI();

const tx = await client.audio.transcriptions.create({
  model: 'gpt-4o-transcribe',
  file: fs.createReadStream('meeting.mp3'),
  language: 'en',                                    // ISO-639-1
  response_format: 'verbose_json',
  timestamp_granularities: ['segment', 'word'],
  temperature: 0,
});

console.log(tx.text);
for (const seg of tx.segments ?? []) {
  console.log(`[${seg.start.toFixed(2)}-${seg.end.toFixed(2)}] ${seg.text}`);
}
```

For a `Blob` / `File` (browser-server proxy) or in-memory buffer, use `toFile`:

```ts
import { toFile } from 'openai';
const file = await toFile(buffer, 'audio.mp3');
const tx = await client.audio.transcriptions.create({ model: 'gpt-4o-transcribe', file });
```

### Parameters

| Field                       | Type                                       | Notes                                                    |
| --------------------------- | ------------------------------------------ | -------------------------------------------------------- |
| `file`                      | binary, required                           | Max **25 MB**.                                            |
| `model`                     | string, required                           | See list above.                                           |
| `language`                  | string                                     | ISO-639-1 (`en`, `de`, `ja`). Reject: `en-US`, 3-letter.  |
| `prompt`                    | string                                     | Style hint / continuation.                               |
| `response_format`           | enum                                       | `json` (default), `text`, `srt`, `verbose_json`, `vtt`, `diarized_json`. |
| `temperature`               | float [0, 1]                               | If 0, server auto-raises until threshold.                 |
| `timestamp_granularities`   | `('segment' \| 'word')[]`                  | Requires `response_format: 'verbose_json'`.              |
| `include`                   | `('logprobs')[]`                           | Only with `gpt-4o-transcribe*` + `response_format: 'json'`. |
| `stream`                    | bool                                       | SSE streaming. NOT supported on `whisper-1`.              |
| `chunking_strategy`         | `'auto'` or object                         | Required for diarize on >30s audio.                      |
| `known_speaker_names`       | `string[]` up to 4                         | Diarize only.                                             |
| `known_speaker_references`  | data URLs (2–10s clips)                    | Diarize only.                                             |

### Audio formats

`flac`, `mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `ogg`, `wav`, `webm`. Max 25 MB (bytes, not minutes — pre-compress for long recordings).

### Streaming

```ts
const stream = await client.audio.transcriptions.create({
  model: 'gpt-4o-transcribe',
  file: fs.createReadStream('call.wav'),
  response_format: 'json',
  stream: true,
});

for await (const event of stream) {
  if (event.type === 'transcript.text.delta') {
    process.stdout.write(event.delta);
  } else if (event.type === 'transcript.text.done') {
    console.log(`\n--- final ---\n${event.text}`);
  }
}
```

### Diarization

```ts
const tx = await client.audio.transcriptions.create({
  model: 'gpt-4o-transcribe-diarize',
  file: fs.createReadStream('interview.wav'),
  response_format: 'diarized_json',
  chunking_strategy: 'auto',
  known_speaker_names: ['Alice', 'Bob'],
});

for (const s of tx.segments) {
  console.log(`${s.speaker} [${s.start.toFixed(2)}-${s.end.toFixed(2)}] ${s.text}`);
}
```

Diarize does NOT support `prompt`, `include: ['logprobs']`, or `timestamp_granularities`. Only `response_format` of `json`, `text`, or `diarized_json`.

---

## Translation (any → English)

Endpoint: `/v1/audio/translations`. Always English output.

```ts
const tr = await client.audio.translations.create({
  model: 'whisper-1',                          // only supported model
  file: fs.createReadStream('spanish.m4a'),
  response_format: 'text',
});
console.log(tr);
```

Subset of transcription params — `file`, `model`, `prompt` (English only), `response_format`, `temperature`. No `language`, no `stream`, no diarization. 25 MB limit applies.

For X → Y where Y ≠ English: transcribe with `gpt-4o-transcribe`, then translate via Responses.

---

## Text-to-Speech

Endpoint: `/v1/audio/speech`. Resource: `client.audio.speech`.

### Models

- **`gpt-4o-mini-tts`** — recommended default. Steerable via `instructions`.
- **`tts-1`** — legacy low-latency. Ignores `instructions`.
- **`tts-1-hd`** — legacy HD. Ignores `instructions`.

### Voices

`alloy`, `ash`, `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`. **`marin` and `cedar` (Aug 2025) are recommended.** Legacy `fable`, `nova`, `onyx` still work on `tts-1` / `tts-1-hd` as raw strings.

Custom clone: `voice: { id: 'voice_...' }`.

### Minimal call — save to file

```ts
import fs from 'fs';
import OpenAI from 'openai';

const client = new OpenAI();

const audio = await client.audio.speech.create({
  model: 'gpt-4o-mini-tts',
  voice: 'marin',
  input: 'The quick brown fox jumps over the lazy dog.',
  instructions: 'Cheerful, warm, and a bit playful.',
  response_format: 'mp3',
  speed: 1.0,
});

const buffer = Buffer.from(await audio.arrayBuffer());
fs.writeFileSync('speech.mp3', buffer);
```

### Parameters

| Field             | Type                                                | Notes                                       |
| ----------------- | --------------------------------------------------- | ------------------------------------------- |
| `input`           | string, required                                    | Max **4096 characters**.                    |
| `model`           | string, required                                    | See list above.                             |
| `voice`           | string \| `{ id: 'voice_...' }`                      | See voice list.                             |
| `instructions`    | string                                              | Steering. Ignored on `tts-1` / `tts-1-hd`.   |
| `response_format` | `'mp3' \| 'opus' \| 'aac' \| 'flac' \| 'wav' \| 'pcm'` | Default `'mp3'`.                            |
| `speed`           | float [0.25, 4.0]                                   | Default 1.0.                                |
| `stream_format`   | `'sse' \| 'audio'`                                  | NOT supported on `tts-1` / `tts-1-hd`.       |

### Streaming PCM (audio buffer)

```ts
const response = await client.audio.speech.create({
  model: 'gpt-4o-mini-tts',
  voice: 'cedar',
  input: longText,
  response_format: 'pcm',                      // 24 kHz, 16-bit signed LE, mono, NO header
});

const reader = response.body!.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  audioDevice.write(value);
}
```

**PCM has no header.** Out-of-band info: 24 kHz, s16le, mono. For self-describing, use `'wav'`.

### Streaming MP3 to a browser

```ts
// Server (Next.js Route Handler)
export async function POST(req: Request) {
  const { text } = await req.json();
  const client = new OpenAI();

  const audio = await client.audio.speech.create({
    model: 'gpt-4o-mini-tts',
    voice: 'marin',
    input: text,
    response_format: 'mp3',
  });

  return new Response(audio.body, {
    headers: { 'content-type': 'audio/mpeg' },
  });
}
```

Browser sets `<audio>.src` to this endpoint URL and streams.

### Long-form TTS

The 4096-character cap means chunking:

```ts
function chunkText(text: string, maxChars = 3800): string[] {
  const sentences = text.split(/(?<=[.!?])\s+/);
  const out: string[] = [];
  let cur = '';
  for (const s of sentences) {
    if (cur.length + s.length + 1 > maxChars && cur) {
      out.push(cur);
      cur = s;
    } else {
      cur = cur ? `${cur} ${s}` : s;
    }
  }
  if (cur) out.push(cur);
  return out;
}

for (const [i, chunk] of chunkText(article).entries()) {
  const audio = await client.audio.speech.create({
    model: 'gpt-4o-mini-tts',
    voice: 'marin',                            // same throughout
    input: chunk,
    instructions: 'Calm narrator.',
    response_format: 'mp3',
    speed: 1.0,
  });
  const buf = Buffer.from(await audio.arrayBuffer());
  fs.writeFileSync(`part_${String(i).padStart(3, '0')}.mp3`, buf);
}
// concatenate with ffmpeg
```

Keep `voice`, `instructions`, `speed`, `response_format` identical across chunks.

---

## Common Pitfalls

- **25 MB is bytes, not duration.** Pre-compress for long recordings.
- **`whisper-1` cannot stream.** Use `gpt-4o-transcribe` for streaming.
- **`whisper-1` is NOT deprecated** — still supported, feature-frozen.
- **`gpt-4o-transcribe-diarize` requires `chunking_strategy`** for >30s audio and rejects `prompt`, `logprobs`, `timestamp_granularities`.
- **`timestamp_granularities` requires `response_format: 'verbose_json'`.**
- **`language` must be ISO-639-1.** `en-US`, `eng`, `english` are rejected.
- **Translation only targets English.**
- **TTS `input` cap is 4096 chars.** Plan to chunk.
- **PCM output has no header.** Use WAV if self-describing matters.
- **`instructions` is silently ignored on `tts-1` / `tts-1-hd`.** Use `gpt-4o-mini-tts` for steering.
- **In a browser proxy, stream audio responses directly** rather than buffering server-side — faster TTFB for the user.
