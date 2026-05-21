# Python — Audio APIs (Transcription, Translation, TTS)

Three endpoints: `/v1/audio/transcriptions`, `/v1/audio/translations`, `/v1/audio/speech`. These are **separate from the Responses API** — you call them directly on `client.audio.*`.

For pricing and model selection see `../../shared/models.md`. For secret handling see `../../shared/secrets.md`.

---

## Transcription (speech-to-text)

Endpoint: `/v1/audio/transcriptions`. Resource: `client.audio.transcriptions`.

### Models (May 2026)

- **`gpt-4o-transcribe`** — recommended default. Best WER, multilingual, supports streaming.
- **`gpt-4o-mini-transcribe`** — cheaper variant. Half the price of `gpt-4o-transcribe`.
- **`gpt-4o-transcribe-diarize`** — adds speaker diarization. Requires `chunking_strategy` for audio >30 s.
- **`whisper-1`** — legacy. Still supported (NOT deprecated), but no streaming and no diarization. Use when you specifically need offline-compatible Whisper.

### Minimal call

```python
from openai import OpenAI

client = OpenAI()

with open("meeting.mp3", "rb") as f:
    tx = client.audio.transcriptions.create(
        model="gpt-4o-transcribe",
        file=f,
        language="en",                           # ISO-639-1 (2-letter); improves accuracy
        response_format="verbose_json",
        timestamp_granularities=["segment", "word"],
        temperature=0,
    )

print(tx.text)
for seg in tx.segments:
    print(f"[{seg.start:.2f}-{seg.end:.2f}] {seg.text}")
```

### Parameters

| Field                       | Type                                | Notes                                                   |
| --------------------------- | ----------------------------------- | ------------------------------------------------------- |
| `file`                      | binary, required                    | Open in `"rb"`. Max **25 MB**.                          |
| `model`                     | string, required                    | See list above.                                         |
| `language`                  | string                              | ISO-639-1 (`en`, `de`, `ja`). Rejected: `en-US`, 3-letter codes. |
| `prompt`                    | string                              | Style / vocabulary hint, or continuation of a prior chunk. |
| `response_format`           | enum                                | `json` (default), `text`, `srt`, `verbose_json`, `vtt`, `diarized_json`. |
| `temperature`               | float [0, 1]                        | If 0, server auto-raises until threshold.                |
| `timestamp_granularities`   | list of `"segment"` / `"word"`      | Requires `response_format="verbose_json"`.              |
| `include`                   | list of `"logprobs"`                | Only with `gpt-4o-transcribe*` + `response_format="json"`. |
| `stream`                    | bool                                | SSE streaming. NOT supported on `whisper-1`.            |
| `chunking_strategy`         | `"auto"` or dict                    | Required for `gpt-4o-transcribe-diarize` audio >30 s.   |
| `known_speaker_names`       | list[str] up to 4                   | Diarize only.                                           |
| `known_speaker_references`  | list of data URLs (2–10s clips)     | Diarize only.                                           |

### Supported audio formats

`flac`, `mp3`, `mp4`, `mpeg`, `mpga`, `m4a`, `ogg`, `wav`, `webm`.

### File size: 25 MB

This is **bytes**, not minutes. Plan accordingly:

- 25 MB uncompressed WAV ≈ 3 minutes.
- 25 MB at 64 kbps mono MP3 ≈ 50 minutes.

For longer audio, pre-compress (ffmpeg: `ffmpeg -i input.wav -b:a 64k -ac 1 output.mp3`) or split on silence boundaries with `ffmpeg -af silencedetect` or `pydub.silence.split_on_silence`, then concatenate transcripts.

### Streaming

```python
stream = client.audio.transcriptions.create(
    model="gpt-4o-transcribe",
    file=open("call.wav", "rb"),
    response_format="json",
    stream=True,
    include=["logprobs"],
)
for event in stream:
    if event.type == "transcript.text.delta":
        print(event.delta, end="", flush=True)
    elif event.type == "transcript.text.done":
        print(f"\n--- final ---\n{event.text}")
```

Event types: `transcript.text.delta`, `transcript.text.segment` (diarized only), `transcript.text.done`.

### Diarization

```python
tx = client.audio.transcriptions.create(
    model="gpt-4o-transcribe-diarize",
    file=open("interview.wav", "rb"),
    response_format="diarized_json",
    chunking_strategy="auto",
    known_speaker_names=["Alice", "Bob"],
)
for s in tx.segments:
    print(f"{s.speaker} [{s.start:.2f}-{s.end:.2f}] {s.text}")
```

Without `known_speaker_names`, speakers come back as `A`, `B`, etc.

**Diarize-specific constraints:** does NOT support `prompt`, `include=["logprobs"]`, or `timestamp_granularities`. Only `response_format` of `json`, `text`, or `diarized_json`.

---

## Translation (any → English)

Endpoint: `/v1/audio/translations`. Always outputs English regardless of source language.

### Model

Only **`whisper-1`** supports this endpoint. There is no `gpt-4o-translate`. For X → Y where Y is not English, transcribe with `gpt-4o-transcribe` and translate the text via `client.responses.create(...)`.

### Minimal call

```python
tr = client.audio.translations.create(
    model="whisper-1",
    file=open("spanish.m4a", "rb"),
    response_format="text",
    prompt="Use formal English phrasing.",     # English only
)
print(tr)
```

Parameters: subset of transcription — `file`, `model`, `prompt`, `response_format`, `temperature`. No `language`, no `stream`, no `timestamp_granularities`, no diarization fields. 25 MB limit still applies.

---

## Text-to-Speech

Endpoint: `/v1/audio/speech`. Resource: `client.audio.speech`.

### Models

- **`gpt-4o-mini-tts`** — recommended default. Steerable via `instructions`. ~$0.015/min.
- **`tts-1`** — legacy low-latency. **Ignores `instructions`.**
- **`tts-1-hd`** — legacy higher quality. **Ignores `instructions`.**

### Voices

Typed literal in SDK: `alloy`, `ash`, `ballad`, `coral`, `echo`, `sage`, `shimmer`, `verse`, `marin`, `cedar`. **`marin` and `cedar` are newest (Aug 2025) and recommended.** Legacy `fable`, `nova`, `onyx` still accepted as raw strings on `tts-1` / `tts-1-hd`.

For custom voice clones: pass `voice={"id": "voice_..."}`.

### Minimal call — save to file

```python
from pathlib import Path
from openai import OpenAI

client = OpenAI()
out = Path("speech.mp3")

with client.audio.speech.with_streaming_response.create(
    model="gpt-4o-mini-tts",
    voice="marin",
    input="The quick brown fox jumps over the lazy dog.",
    instructions="Cheerful, warm, and a bit playful.",
    response_format="mp3",
    speed=1.0,
) as response:
    response.stream_to_file(out)
```

**Always use `with_streaming_response.create(...)`** as a context manager. The bare `.create(...)` returns binary in memory — fine for short snippets, but for anything longer it bloats memory and you lose the chunked write.

### Parameters

| Field             | Type                                                        | Notes                                                |
| ----------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| `input`           | string, required                                            | Max **4096 characters**.                             |
| `model`           | string, required                                            | See list above.                                      |
| `voice`           | string or `{"id": "voice_..."}`                             | See voice list.                                      |
| `instructions`    | string                                                      | Style / tone / pacing / accent. Ignored on `tts-1` / `tts-1-hd`. |
| `response_format` | `"mp3" \| "opus" \| "aac" \| "flac" \| "wav" \| "pcm"`      | Default `"mp3"`.                                     |
| `speed`           | float [0.25, 4.0]                                           | Default `1.0`.                                       |
| `stream_format`   | `"sse" \| "audio"`                                          | NOT supported on `tts-1` / `tts-1-hd`.               |

### `instructions` — steering style (gpt-4o-mini-tts only)

> "Speak with urgency and a serious news-anchor tone. Slightly faster pace."
> "Slow, deliberate, audiobook narrator. Thoughtful pauses."
> "Very soothing, very slow, long pauses. Meditation guide."
> "Warm, inviting, like a friendly receptionist."

Applies to the whole `input` (not segmented). For multi-section narration with different styles, chunk into multiple calls and concatenate audio.

### Streaming raw PCM

```python
with client.audio.speech.with_streaming_response.create(
    model="gpt-4o-mini-tts",
    voice="cedar",
    input=long_text,
    response_format="pcm",                  # 24 kHz, 16-bit signed LE, mono, NO header
) as response:
    for chunk in response.iter_bytes(chunk_size=4096):
        audio_device.write(chunk)
```

**PCM has no header.** Out-of-band info: 24 kHz, s16le, mono. If you need a self-describing format, use `"wav"` instead.

### Long-form TTS

The 4096-character cap means you split:

```python
import re

def chunks(text: str, max_chars: int = 3800) -> list[str]:
    sentences = re.split(r"(?<=[.!?])\s+", text)
    out, cur = [], ""
    for s in sentences:
        if len(cur) + len(s) + 1 > max_chars and cur:
            out.append(cur)
            cur = s
        else:
            cur = f"{cur} {s}".strip() if cur else s
    if cur:
        out.append(cur)
    return out

for i, chunk in enumerate(chunks(article)):
    with client.audio.speech.with_streaming_response.create(
        model="gpt-4o-mini-tts",
        voice="marin",                          # same voice & instructions throughout
        input=chunk,
        instructions="Calm narrator.",
        response_format="mp3",
        speed=1.0,
    ) as response:
        response.stream_to_file(f"part_{i:03d}.mp3")

# Concatenate with ffmpeg / pydub
```

Keep `voice`, `instructions`, `speed`, `response_format` identical across chunks to avoid audible seams.

---

## Common Pitfalls

- **25 MB is bytes, not duration.** Pre-compress to 64 kbps mono MP3 for long recordings.
- **`whisper-1` cannot stream.** Pick `gpt-4o-transcribe` / `gpt-4o-mini-transcribe` if you need `stream=True`.
- **`whisper-1` is NOT deprecated.** Use it freely; it's just feature-frozen.
- **`gpt-4o-transcribe-diarize` requires `chunking_strategy`** for audio >30 s and rejects `prompt`, `logprobs`, `timestamp_granularities`.
- **`timestamp_granularities` requires `response_format="verbose_json"`.** Other formats will reject it.
- **`language` must be ISO-639-1** (2 letters). `en-US`, `eng`, `english` are rejected.
- **Translation only targets English.** For other targets, transcribe → translate via Responses.
- **TTS `input` cap is 4096 chars.** Plan to chunk for anything longer.
- **PCM output has no header.** Use WAV if you need a self-describing format.
- **`instructions` is silently ignored on `tts-1` / `tts-1-hd`** — no error, just a flat read. Use `gpt-4o-mini-tts` for steering.
- **Use the streaming response context manager** (`with_streaming_response.create(...)`) for chunked writes — the bare call buffers the whole binary.
