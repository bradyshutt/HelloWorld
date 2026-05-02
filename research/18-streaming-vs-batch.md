# Streaming vs Batch Transcription

## Summary

Streaming STT delivers partial transcripts in 150-300 ms over a persistent connection, which is essential when the voice journal needs to show words on screen as the user speaks. Batch (pre-recorded) transcription is roughly 40-50% cheaper and consistently 1-2 absolute WER points more accurate (often 10-17% relative) because the model sees full audio context and can run multiple passes. A practical pattern for a journaling app is hybrid: use streaming for the live "speak and see" UI, then re-transcribe the saved audio in batch to produce the canonical, high-accuracy entry that the AI narrator reads back later.

## Streaming Architecture

A streaming pipeline opens a persistent connection (typically a WebSocket carrying 16 kHz PCM frames) that stays alive for the whole utterance. The server returns two message types: **interim/partial** results that may change as more audio arrives, and **final** results (`is_final: true`) that are locked in. Partial transcripts arrive every 100-300 ms, which is the budget needed for live captions and conversational agents; delays beyond ~300 ms feel laggy in interactive contexts, while passive captions tolerate 1-3 seconds ([AssemblyAI streaming](https://www.assemblyai.com/products/streaming-speech-to-text), [Deepgram live audio](https://developers.deepgram.com/reference/speech-to-text/listen-streaming)).

For browser-to-server media transport there are two real options:

- **WebSockets (over TCP)** — simple to implement, works everywhere, and is what most STT vendors document. Downside: a single dropped packet stalls the stream while TCP retransmits.
- **WebRTC (over UDP/SRTP)** — designed for realtime media. Includes adaptive jitter buffers, acoustic echo cancellation, automatic gain control, noise suppression, and congestion control (GCC) tuned for audio. Recommended by OpenAI and LiveKit for client-to-agent links, especially on mobile or flaky networks ([LiveKit](https://livekit.com/blog/why-webrtc-beats-websockets-for-voice-ai-agents), [OpenAI Realtime](https://developers.openai.com/api/docs/guides/realtime-webrtc)).

A common production layout uses WebRTC from the user's device to an edge gateway and WebSockets server-to-server into the STT provider.

## Batch Architecture

Batch (a.k.a. pre-recorded or async) transcription uploads a finished audio file (or a URL/object-store reference) and polls or webhooks for a completed transcript. There is no latency budget, so the model can run beam search wider, do a second pass with full-utterance context, and apply heavier punctuation/diarization/formatting models. A one-hour file typically returns in a few minutes ([AssemblyAI: real-time vs batch](https://www.assemblyai.com/blog/real-time-vs-batch-transcription)).

Microsoft and others now offer a middle path called **post-stream refinement**: stream during capture, then automatically re-run a higher-accuracy batch pass on the saved audio so the final stored transcript matches batch quality ([Azure post-stream refinement](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/introducing-post-stream-refinement-higher-accuracy-real-time-transcription/4510038)).

## Cost & Accuracy Trade-offs

Representative 2026 pricing and quality (English, general domain):

| Mode | Deepgram Nova-3 | AssemblyAI Universal-3 |
|---|---|---|
| Streaming | $0.0077/min, ~6.84% WER, sub-300 ms p50 | ~$0.0075/min ($0.45/hr), ~150 ms p50 |
| Batch | $0.0043/min, ~5.26% WER | $0.0025/min ($0.15/hr) |

Batch is typically **40-50% cheaper** and **10-17% relatively more accurate** than streaming for the same model family because it can use full bidirectional context; chunked streaming configurations can degrade WER by up to ~46% in adversarial cases ([Deepgram 2026 comparison](https://deepgram.com/learn/best-speech-to-text-apis-2026), [BuildMVPFast pricing](https://www.buildmvpfast.com/api-costs/transcription), [Picovoice guide](https://picovoice.ai/blog/complete-guide-to-streaming-speech-to-text/)). Streaming also struggles more with proper nouns, brand terms, and uncommon vocabulary because it cannot peek ahead.

## Recommendations

For the voice journal's three goals, a hybrid approach fits best:

1. **Live transcript while speaking (goal 2: speaking practice).** Use streaming STT over a WebSocket (or WebRTC if mobile/poor networks matter). Render partials in a lighter color and "lock" final segments as they arrive — this gives the immediate idea-to-voice feedback that builds the speaking habit.
2. **Canonical journal entry (goal 1: durable log).** When the user stops recording, kick off a batch transcription on the full uploaded audio. Replace the streamed text with the higher-accuracy batch result before saving. This costs roughly half as much per minute and produces noticeably cleaner text for search and for downstream LLM use.
3. **AI narration (goal 3: epic story playback).** Always feed the batch transcript (not the streamed one) into the narrator LLM — proper-noun accuracy matters when the story references the user's friends, places, and projects.

If cost or simplicity dominates and a live transcript is not strictly required, batch-only is the cheapest, highest-accuracy option and is acceptable for short (3-10 minute) journal sessions where waiting 30-60 seconds after stopping is fine.

Sources:
- [AssemblyAI: Real-time vs batch transcription](https://www.assemblyai.com/blog/real-time-vs-batch-transcription)
- [AssemblyAI: Real-Time Speech to Text Guide](https://www.assemblyai.com/blog/real-time-speech-to-text)
- [Deepgram: Best Speech-to-Text APIs in 2026](https://deepgram.com/learn/best-speech-to-text-apis-2026)
- [Deepgram: Streaming TTS Latency / Accuracy Tradeoff 2026](https://deepgram.com/learn/streaming-tts-latency-accuracy-tradeoff-2026)
- [Picovoice: Complete Guide to Streaming Speech-to-Text (2026)](https://picovoice.ai/blog/complete-guide-to-streaming-speech-to-text/)
- [LiveKit: Why WebRTC beats WebSockets for realtime voice AI](https://livekit.com/blog/why-webrtc-beats-websockets-for-voice-ai-agents)
- [OpenAI Realtime API with WebRTC](https://developers.openai.com/api/docs/guides/realtime-webrtc)
- [Microsoft: Post-Stream Refinement for higher-accuracy real-time transcription](https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/introducing-post-stream-refinement-higher-accuracy-real-time-transcription/4510038)
- [BuildMVPFast: STT API Pricing (Feb 2026)](https://www.buildmvpfast.com/api-costs/transcription)
- [Deepgram live audio API reference](https://developers.deepgram.com/reference/speech-to-text/listen-streaming)
