---
name: feature-voice
description: Use when adding voice to an agent — text-to-speech (TTS), speech-to-text (STT), or live duplex WebSocket streaming. Requires tier-gated voice_generation capability.
---

# Voice

## What it is

Three surfaces: **TTS** (text → audio), **STT** (audio → text), **live duplex** (bidirectional WebSocket stream with the agent). Voice is tier-gated — `voiceGeneration` must be unlocked on the project.

## When to use

- Voice journaling (companion archetype)
- Phone IVR / voice support
- Voice-mode for companion / coach apps
- Real-time voice chat in games

## When NOT to use

- TTS for one-off announcements unrelated to an agent — use a regular TTS service
- Long async voice generation (>30s of audio per request) — TTS is best for short responses; for long content, batch and stream

## Tier check (mandatory)

```python
caps = client.agents.get_capabilities(agent_id)
if not caps.voice_generation:
    raise RuntimeError("Voice not unlocked on this project. Contact admin / upgrade tier.")
print(caps.voice_tier, caps.voice_id)
```

`voiceGeneration` and friends are NOT toggleable via `update_capabilities` — they're set by billing tier / admin action.

## SDK surface

### TTS

```python
tts_response = client.agents.voice.tts(
    agent_id,
    text="Hello, how are you?",
    voice_name="Kore",                # from voices.list()
    language="en-US",
)
audio_bytes = base64.b64decode(tts_response.audio)
# tts_response also carries sample_rate, format
```

### STT

```python
stt_response = client.agents.voice.stt(
    agent_id,
    audio=base64.b64encode(pcm_buffer).decode(),
    audio_format="pcm",
    language="en-US",
)
print(stt_response.transcript)
```

### Live duplex

```python
# Mint a token (short-lived)
token = client.agents.voice.get_token(
    agent_id,
    voice_name="Kore",
    language="en-US",
    user_id="user-123",
)

# Open the WebSocket stream
stream = client.agents.voice.stream(token)

# Send (interleave as needed)
stream.send_text("Hello!")
stream.send_audio(pcm_16khz_mono_bytes)

# Receive events
for event in stream:
    if event.type == "input_transcript":
        print("User:", event.text)
    elif event.type == "output_transcript":
        print("Agent:", event.text)
    elif event.type == "audio":
        play_pcm(event.audio)        # 24 kHz PCM out
    elif event.type == "session_ended":
        break

stream.close()
```

```typescript
const token = await client.agents.voice.getToken(agentId, {
  voiceName: "Kore", language: "en-US", userId: "user-123",
});
const stream = await client.agents.voice.stream(token);
stream.sendText("Hello!");
for await (const event of stream) {
  if (event.type === "audio") playPCM(event.audio);
  if (event.type === "session_ended") break;
}
stream.close();
```

### Voice catalog

```python
voices = client.voices.list()
for v in voices:
    print(v.name, v.gender, v.style)
```

## Audio formats

| Direction | Format |
|---|---|
| STT input | PCM 16 kHz mono (typical) — verify against `voices.list()` per-voice specs |
| Live stream input | PCM 16 kHz mono |
| TTS output | 24 kHz PCM (default) |
| Live stream output | 24 kHz PCM |

## Decisions linked

- `decisions/memory-mode.md` — voice **forces** `memory_mode=async`
- `archetypes/companion.md`, `archetypes/coach-therapist.md` — voice as optional add-on
- `features/capabilities.md` — tier-gating

## Common gotchas

- **Voice forces async memory.** Sync blocks the audio loop; first-token latency goes over budget. Confirm `memory_mode=async` before voice.
- **Token is short-lived.** Mint per-session, don't cache long-term.
- **WebSocket disconnections** — implement reconnect logic; voice live is fragile on flaky networks.
- **Sample rate mismatch** — PCM in must match the expected rate; resample if your source is different.
- **Voice not unlocked** — `voiceGeneration` is read-only via API; check first, fail loudly if absent.
- **`voiceId`** selects which voice from `voices.list()`. Defaults to project default.
- **Latency budget** — live voice TTFC includes STT → context build → generation → TTS. Plan for ~500ms-1500ms.
- **Privacy** — STT may transcribe sensitive content; ensure your audit trail covers it.
