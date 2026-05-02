# Text-to-Speech for Narration

## Summary

For a voice journal that narrates back the day in fun styles (e.g. "epic story"), the strongest options in 2026 are **ElevenLabs v3** (best-in-class expressive narration via inline audio tags), **OpenAI gpt-4o-mini-tts** (cheap, steerable via natural-language instructions), and **Hume Octave 2** (purpose-built for emotional delivery). Cloud commodity engines (Google, Azure, Polly) are reliable and cheap but less theatrical. For offline or zero-cost playback, **Kokoro-82M** offers the best quality-per-byte, with **Piper** as a reliable lighter fallback.

## Landscape

- **ElevenLabs**: Eleven v3 is their flagship expressive model; Flash v2.5 (~75ms TTFB) and Turbo v2.5 (~250-300ms) handle real-time. v3 itself has higher latency and is not yet recommended for live conversation. Pricing starts at $5/mo (30k chars) up to enterprise tiers; 3,000+ voices and industry-leading cloning ([ElevenLabs models](https://elevenlabs.io/docs/overview/models), [Vapi blog](https://vapi.ai/blog/elevenlabs-vs-openai)).
- **OpenAI TTS (gpt-4o-mini-tts)**: 13 voices, ~$0.60/M input chars (or roughly $0.015/min), accepts a free-form `instructions` parameter to steer tone ("cheerful," "poetic," "noir narrator"). First chunk in ~300-600ms with streaming ([OpenAI announcement](https://openai.com/index/introducing-our-next-generation-audio-models/), [TokenMix](https://tokenmix.ai/blog/gpt-4o-mini-tts-cheapest-tts-api-2026)).
- **PlayHT**: 600+ voices, 140+ languages, strong on conversational long-form; subscription ~$374/yr for 600k words. Generally cheaper than ElevenLabs at scale but lower top-end quality ([Speechify comparison](https://speechify.com/blog/elevenlabs-vs-play-ht/)).
- **Google Cloud TTS**: Standard $4/M chars, WaveNet/Neural2 $16/M, Studio & Chirp 3 HD $30/M; ~300 voices, 50+ languages, sub-800ms latency, very natural prosody ([Speechmatics](https://www.speechmatics.com/company/articles-and-news/best-tts-apis-in-2025-top-12-text-to-speech-services-for-developers)).
- **Azure Neural Voices**: $16/M (custom $24/M), 129 neural voices across 54 locales, robust SSML and style/role tags ([VoiceKeep](https://voicekeep.io/guides/tts-api-comparison)).
- **Amazon Polly**: $4/M standard, $16/M neural, 31 generative voices in 20 languages, 5M chars/mo free first year, very predictable latency ([Awesome Agents](https://awesomeagents.ai/pricing/voice-tts-pricing/)).
- **Hume Octave 2**: Emotionally-intelligent TTS driven by plain-English direction; strong for empathy/excitement/calm ([Hume Octave](https://www.hume.ai/octave)).
- **Cartesia Sonic-3**: 90ms TTFA (40ms Turbo), real-time laughter/emotion - leader for live agents ([Cartesia](https://cartesia.ai/sonic)).
- **On-device**: **Kokoro-82M** (Apache 2.0, ~4.5 MOS, sub-300ms CPU synthesis); **Piper** (MIT, ONNX, runs on a Raspberry Pi 4 at RTF 0.20); plus OS-native (AVSpeechSynthesizer / Web Speech API) for free fallback ([BentoML](https://www.bentoml.com/blog/exploring-the-world-of-open-source-text-to-speech-models), [CodeSOTA](https://www.codesota.com/guides/tts-models)).

## Voice Style & Emotion

For "epic story" narration, expressiveness matters more than raw latency:

- **ElevenLabs v3** uses inline audio tags - `[whispers]`, `[laughs]`, `[shouts]`, `[curious]`, `[mischievously]`, `[door slam]` - giving stage-direction-level control. It has a wide dynamic range that makes narration feel performed, not read ([Audio Tags Guide](https://audio-generation-plugin.com/elevenlabs-v3/), [ElevenLabs Help](https://help.elevenlabs.io/hc/en-us/articles/35869142561297-How-do-audio-tags-work-with-Eleven-v3)).
- **OpenAI gpt-4o-mini-tts** is steered via a single natural-language `instructions` field - perfect for swapping styles ("narrate this like a Norse saga," "deliver as a noir detective") without re-engineering prompts ([PromptLayer](https://blog.promptlayer.com/gpt-4o-mini-tts-steerable-low-cost-speech-via-simple-apis/)).
- **Hume Octave 2** detects emotional context in the journal text itself and adjusts delivery automatically - useful when entries swing between elated and melancholy.
- **Azure** offers SSML `<mstts:express-as>` styles (newscast, cheerful, sad, narration-relaxed) with intensity control - good middle ground.
- **Google Studio / Chirp 3 HD** voices are very natural but offer less explicit style steering.
- **On-device** Kokoro/Piper are flat compared to cloud frontier voices; usable for plain readback but not "epic story."

## Trade-offs

| Concern | Best fit |
|---|---|
| Most theatrical narration | ElevenLabs v3, Hume Octave 2 |
| Cheapest steerable cloud | OpenAI gpt-4o-mini-tts |
| Lowest latency streaming | Cartesia Sonic-3 (~90ms), ElevenLabs Flash v2.5 (~75ms) |
| Privacy / offline / free | Kokoro-82M, Piper, system TTS |
| Enterprise reliability + SSML | Azure Neural, Amazon Polly |
| Cheapest at scale | Polly/Google Standard ($4/M) |

Key tension: the most expressive model (ElevenLabs v3) is also the slowest, while the snappiest models (Flash, Sonic-3) are tuned for live agents and less dramatic. Narration playback is asynchronous, so latency tolerance is higher than in a live chat loop.

## Recommendations for voice journal

1. **Default narration engine: ElevenLabs v3** for the "epic story" recap - audio tags map cleanly onto journal moods (`[excited]`, `[whispers]`, `[sighs]`). Higher latency is acceptable because the recap is generated once per day and can be pre-rendered.
2. **Live "thinking out loud" responses (if any): OpenAI gpt-4o-mini-tts** or **ElevenLabs Flash v2.5** - low cost, low latency, and the OpenAI `instructions` field lets the same voice flip between "supportive coach," "epic narrator," and "noir detective" styles.
3. **Free / offline fallback: Kokoro-82M** (with Piper as a Raspberry-Pi-class fallback) so the journal still plays back without network or budget. Wire it behind a provider abstraction.
4. **Skip for this use case**: Polly/Google Standard voices (too flat for storytelling) and Custom Neural Voice (overkill unless the user wants their own cloned voice - in which case ElevenLabs Instant Voice Cloning is simpler).
5. **Architecture tip**: cache the generated narration audio per day; charge the expressive model only on first play, then serve from disk on replays.

Sources:
- [SurePrompts: Voice Generation Models Compared 2026](https://sureprompts.com/blog/voice-generation-models-compared-2026)
- [Gladia: Best TTS APIs for developers 2026](https://www.gladia.io/blog/best-tts-apis-for-developers-in-2026-top-7-text-to-speech-services)
- [Inworld: ElevenLabs v3 Review](https://inworld.ai/resources/elevenlabs-v3-review)
- [ocdevel: Open-Source TTS Comparison](https://ocdevel.com/blog/20250720-tts)
- [Inferless: 12 Best Open-Source TTS Models](https://www.inferless.com/learn/comparing-different-text-to-speech---tts--models-part-2)
