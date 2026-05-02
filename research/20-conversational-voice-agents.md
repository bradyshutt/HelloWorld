# Conversational Voice Agent Stacks

## Summary

For a back-and-forth voice journaling app, the 2026 landscape splits into three layers: **speech-to-speech model APIs** (OpenAI Realtime, Gemini Live), **managed voice-agent platforms** (ElevenLabs Conversational AI, Vapi, Retell), and **open-source orchestration frameworks** (LiveKit Agents, Pipecat). Speech-to-speech models give the lowest end-to-end latency (300–500 ms steady-state) and the most natural follow-up question delivery, while orchestrators add WebRTC transport, turn-taking, and observability around them. For a journaling project with modest volume and a need for fast iteration, a managed stack (ElevenLabs Agents or Vapi) or LiveKit Agents wrapping OpenAI Realtime is the sweet spot.

## Stack Options

- **OpenAI Realtime API (`gpt-realtime`, `gpt-realtime-mini`)** — true speech-to-speech with built-in interruption handling, function calling, and a WebSocket interface. Audio pricing is roughly $32/1M input and $64/1M output tokens, with cached-input discounts ([OpenAI](https://openai.com/index/introducing-gpt-realtime/), [TokenMix](https://tokenmix.ai/blog/voice-ai-api-realtime-vs-gemini-live-vs-elevenlabs-2026)).
- **Gemini Live API (`gemini-3.1-flash-live`)** — native barge-in, affective dialog, transcripts, 24+ languages, and the cheapest output at ~$0.018/min, ~7–12× cheaper than alternatives at high volume ([Google](https://ai.google.dev/gemini-api/docs/live-api), [LaoZhang](https://blog.laozhang.ai/en/posts/gemini-3-1-flash-live-api)).
- **ElevenLabs Conversational AI** — billed per minute ($0.08 Standard / $0.10 Turbo / $0.12 Premium), bundles best-in-class TTS, turn-taking models, telephony, and RAG. Targets sub-100 ms TTS latency on Flash v2.5 ([ElevenLabs pricing](https://elevenlabs.io/pricing), [Cekura breakdown](https://www.cekura.ai/blogs/elevenlabs-pricing)).
- **Vapi** — developer-first managed platform; per-minute cost $0.05–0.13 plus separate platform/telephony/voice/model/STT fees, which can stack to ~$0.33/min ([Vapi pricing](https://vapi.ai/pricing), [Retell comparison](https://www.retellai.com/blog/vapi-ai-review)).
- **Retell** — flat $0.07/min platform fee + LLM cost, visual workflow builder, fastest time-to-first-call (~3 hrs) but adds 50–100 ms latency overhead ([Retell vs Vapi](https://www.retellai.com/comparisons/retell-vs-vapi)).
- **LiveKit Agents** — open-source Python/Node framework over WebRTC; clean abstractions, plugin-based integration with OpenAI Realtime and Gemini Live, recommended once volume grows ([LiveKit docs](https://docs.livekit.io/agents/integrations/openai/realtime/), [Forasoft playbook](https://www.forasoft.com/blog/article/livekit-ai-agents-guide)).
- **Pipecat** — open-source Python framework from Daily.co; more granular control of the pipeline but more verbose configuration and you must wire up turn-taking and telephony yourself ([AssemblyAI comparison](https://www.assemblyai.com/blog/vapi-vs-pipecat-vs-livekit)).

## Latency & Interruption

OpenAI Realtime and Gemini 3.1 Flash Live both deliver 300–500 ms steady-state end-to-end latency; Gemini reports ~960 ms time-to-first-token on the opening response ([TokenMix](https://tokenmix.ai/blog/voice-ai-api-realtime-vs-gemini-live-vs-elevenlabs-2026)). Modern stacks have moved past pure VAD to a two-signal approach: acoustic VAD (Silero) plus a semantic turn-detection model that looks at prosody and content to ignore backchannels like "mhm" — when a real interrupt fires, TTS is cancelled and the in-flight LLM turn is rolled back. Target interruption response is <200 ms from speech onset to TTS suppression ([CallBotics](https://callbotics.ai/blog/ai-voice-agent-interruption-handling), [AssemblyAI orchestration](https://www.assemblyai.com/blog/orchestration-tools-ai-voice-agents)). LiveKit, Pipecat, ElevenLabs Agents, and Retell all ship this; Retell is specifically noted for natural mid-sentence interruption.

## Trade-offs

- **Speed to ship vs. control**: Managed (Vapi/Retell/ElevenLabs) = days to a working agent; OSS (LiveKit/Pipecat) = more code but better unit economics past ~10–50k min/month ([Hamming](https://hamming.ai/resources/best-voice-agent-stack)).
- **Cost predictability**: Retell and ElevenLabs publish flat per-minute prices; Vapi's stacked fees and OpenAI/Gemini's token-based audio billing are harder to forecast.
- **Voice quality vs. latency**: ElevenLabs wins on TTS expressiveness (matters for the "epic narrator" feature); Gemini Live wins on cost; OpenAI Realtime balances both with the most mature speech-to-speech model.
- **Lock-in**: Speech-to-speech APIs tie you to one model; LiveKit/Pipecat let you swap STT, LLM, and TTS independently.

## Recommendations for voice journal

For an MVP, start with **LiveKit Agents + OpenAI Realtime API**: clean Python SDK, WebRTC handles the mobile/web client, and Realtime's native interruption + function calling makes "ask follow-up" prompts trivial. This setup also hides the WebSocket plumbing that's awkward to expose directly to a phone or browser ([LiveKit](https://docs.livekit.io/agents/integrations/openai/realtime/)). If TTS expressiveness for the "epic narrator" replay mode is the differentiator, swap the TTS leg to **ElevenLabs v3** (or use ElevenLabs Conversational AI end-to-end for fastest setup). Keep **Gemini Live** in mind as a cost-down option once usage grows — it's the cheapest path for a daily-use journaling app where minutes add up. Avoid Vapi's stacked pricing unless you need its telephony depth, and avoid raw Pipecat unless you want to hand-tune turn-taking.

Sources are linked inline above.
