# Voice Journal — Research Summary

This directory contains 20 research notes covering the technical, product, market, and behavioral-design space around the voice journal idea (a voice-chat daily activity log with optional AI narration in fun styles). Each file has its own summary and recommendations; this document consolidates them.

## Executive Summary

The voice/AI journaling space is crowded but fragmented. Text-first AI journals (Rosebud, Mindsera, Stoic, Day One) treat voice as a transcription side-feature, while voice-first tools (AudioPen, Voicenotes, Otter, Apple Voice Memos) handle dictation well but ignore narrative continuity, daily-log structure, and habit design. The clearest white space sits at the intersection of three under-served product moves: a chronological "what happened today" voice log (not a therapy journal), a stylized "narrate my day" AI playback (only Untold currently does this), and a quiet speaking-fluency loop that rewards verbal articulation rather than written depth.

The supporting tech stack is mature in 2026. Cloud STT is commoditized at ~$0.003–0.006/min, LLMs are now cheap enough that nightly summaries cost cents, and TTS quality has reached the point where "epic story" narration is genuinely entertaining (ElevenLabs v3, Hume Octave 2). The dominant unit-economics constraint is TTS for daily playback, not transcription. Privacy is the single biggest non-technical risk: voice is biometric data under BIPA / GDPR Art. 9, and 2025 saw 100+ class actions, several touching transcription pipelines.

## Recommended Starting Stack

A defensible MVP based on the synthesis of the research:

- **Capture**: Mobile-first using Expo `expo-audio` (or Flutter `record`); web/PWA using `MediaRecorder` + AudioWorklet, encoding to Opus at 24–32 kbps mono. Push-to-talk default with an opt-in eyes-free "just keep talking" mode. ([06](06-browser-audio-recording.md), [07](07-mobile-audio-frameworks.md), [09](09-audio-formats.md), [14](14-voice-first-ux.md))
- **VAD / endpointing**: RNNoise → Silero VAD v6 → STT-based endpointing for the conversational mode. Longer silence threshold (1.0–1.5 s) than typical chat agents because journaling has thinking pauses. ([05](05-voice-activity-detection.md))
- **STT**: Hybrid — on-device first (iOS 26 SpeechAnalyzer, Android SpeechRecognizer, whisper.cpp small.en fallback), cloud fallback on `gpt-4o-mini-transcribe` ($0.003/min) for accuracy-critical entries. Optional batch re-transcription overnight to feed narration. ([01](01-stt-cloud-apis.md), [02](02-stt-on-device.md), [18](18-streaming-vs-batch.md))
- **LLM tier**: Claude Haiku 4.5 or GPT-5.4 Mini for live conversational follow-ups (~600 ms TTFT); Claude Sonnet 4.6 / Gemini Flash for nightly summaries; Claude Opus 4.7 on demand for the "epic story" narration (top creative-writing benchmarks). Optional local Qwen 3 8B path for privacy-first users. ([04](04-llm-options.md))
- **TTS narration**: ElevenLabs v3 default (inline `[whispers]`, `[laughs]` tags for theatrical delivery), `gpt-4o-mini-tts` for cheaper steerable voices, Kokoro-82M as offline fallback. Pre-render and cache nightly. ([03](03-tts-narration.md))
- **Conversational loop**: LiveKit Agents + OpenAI Realtime as the MVP back-and-forth stack; swap TTS to ElevenLabs v3 for narration replay. ([20](20-conversational-voice-agents.md))
- **Architecture**: Local-first hybrid via PowerSync — SQLite metadata + S3/R2 object storage for audio blobs. Audio rides a separate channel from the CRDT/SQLite sync stream. ([16](16-local-vs-cloud.md))
- **Search**: SQLite FTS5 + sqlite-vec embedded, fused with RRF; embeddings via BGE-small-en-v1.5 on-device or `text-embedding-3-small` in the cloud. Word-level timestamps so any hit deep-links to playback. ([17](17-transcript-search.md))
- **Privacy**: On-device capture and transcription first; client-side AES-256-GCM with user-derived keys; voiceprint/diarization disabled by default; tiered retention (raw audio purged within 24–72h); per-purpose consent. DPIA before launch. Avoid OpenAI Whisper API's reported 30-day retention — prefer self-host or vendors with contractual ZDR (Deepgram, AssemblyAI). ([08](08-privacy-security.md))

## Product Direction

- **Differentiator #1 — "Narrate my day" as a first-class output.** Only Untold leans into stylized retelling. User-selectable styles (epic bard, noir, sportscaster, fairy tale, sitcom) with TTS playback is the most defensible wedge. Two-stage pipeline: extract a fact-ledger from the transcript, then restyle, with a faithfulness verification gate before TTS. Style packs as data, not prompts. ([10](10-competitive-landscape.md), [13](13-narrative-ai-prompting.md))
- **Differentiator #2 — Speaking-fluency loop.** A 2025 study found spoken journals are ~4× longer than written ones with larger learning gains; voice journaling also produces *greater* gains in cognitive change and self-esteem than writing in direct comparisons. Surface lightweight metrics (pace, filler rate, MTLD lexical diversity) without nagging — 5 fillers/min is normal. Optional 4/3/2 retell mode for fluency drills. ([11](11-journaling-benefits.md), [12](12-speaking-practice-benefits.md))
- **Differentiator #3 — Activity log, not therapy journal.** Most AI journals push therapeutic prompts; users wanting a chronological "field log" fall back to Voice Memos. A narrative-aware timeline with weekly recaps fills this gap. ([10](10-competitive-landscape.md))
- **Habit design.** Decouple streak (1 entry) from goal (depth/length) — Duolingo's 2024 split delivered +3.3% D14 retention. Open to a prompt, not a blank screen. Allow streak-freezes. AI-narration style packs as variable rewards on milestones. Avoid Snapchat-style forced streaks. ([15](15-habit-formation.md))
- **UX defaults.** One-giant-button + tap-to-toggle (Voice Memos lineage); live waveform + elapsed timer + streaming partial transcript during recording; AI-rewritten glanceable summary post-capture (users prefer this over raw transcripts); never repeat the same fallback twice on errors. ([14](14-voice-first-ux.md))

## Unit Economics & Pricing

A 15–30 min/day power user costs roughly $4–10/mo all-in (STT + LLM + TTS); median user closer to $1–2. TTS for daily "epic narration" is the dominant variable cost (an ElevenLabs Multilingual v2 5-min readback ≈ $0.27–0.54/generation). ([19](19-monetization.md))

Pricing wedge: the mid-market sits at $50–100/yr (Day One Silver $49.99, AudioPen $99, Voicenotes $99, Rosebud Bloom $156). A freemium tier with generous transcription minutes plus paid narration styles could undercut Rosebud while monetizing the "fun layer." Apple Small Business Program (15%) takes a $9.99 sub from $9.99 → $8.49 → ~$5–7 net of AI cost. ([19](19-monetization.md))

## Key Risks & Open Decisions

1. **Privacy / biometric exposure.** Cloud STT vendors that retain audio create real BIPA / GDPR liability. Decision: how aggressive to be on-device (UX cost: slower transcription on weak devices) vs cloud (legal/compliance cost). ([08](08-privacy-security.md))
2. **TTS cost ceiling.** If "narrate my day" becomes the headline feature, daily generation cost can dwarf STT. Decision: cap generations, cache aggressively, or push premium voices to a higher tier. ([03](03-tts-narration.md), [19](19-monetization.md))
3. **Live transcript vs after-the-fact.** Streaming STT is ~40–50% more expensive and ~10–17% relatively less accurate than batch. The right pattern is stream while speaking (for the live UI), then re-batch for the canonical entry. ([18](18-streaming-vs-batch.md))
4. **Conversational depth vs friction.** Rosebud-style follow-ups demand a real-time stack (LiveKit + OpenAI Realtime, ~$0.05–0.10/min). Decision: ship a simpler "monologue + AI summary" v1, or commit to the conversational stack from day one. ([20](20-conversational-voice-agents.md))
5. **Faithfulness in narrative restyling.** "Epic" rewrites must not invent facts. Decision: strict fact-ledger pipeline with verification gate vs. lighter persona prompts (faster, more hallucination-prone). ([13](13-narrative-ai-prompting.md))

## Index of Research Files

| # | File | Topic |
|---|---|---|
| 01 | [stt-cloud-apis](01-stt-cloud-apis.md) | Cloud speech-to-text API comparison |
| 02 | [stt-on-device](02-stt-on-device.md) | On-device / open-source speech recognition |
| 03 | [tts-narration](03-tts-narration.md) | Text-to-speech for narration playback |
| 04 | [llm-options](04-llm-options.md) | LLM options for journaling & narration |
| 05 | [voice-activity-detection](05-voice-activity-detection.md) | VAD & turn-taking |
| 06 | [browser-audio-recording](06-browser-audio-recording.md) | Browser audio recording APIs |
| 07 | [mobile-audio-frameworks](07-mobile-audio-frameworks.md) | Mobile audio recording frameworks |
| 08 | [privacy-security](08-privacy-security.md) | Privacy & security for voice journals |
| 09 | [audio-formats](09-audio-formats.md) | Audio formats & compression |
| 10 | [competitive-landscape](10-competitive-landscape.md) | Competitive landscape |
| 11 | [journaling-benefits](11-journaling-benefits.md) | Mental-health benefits of journaling |
| 12 | [speaking-practice-benefits](12-speaking-practice-benefits.md) | Benefits of speaking practice |
| 13 | [narrative-ai-prompting](13-narrative-ai-prompting.md) | Narrative AI prompting |
| 14 | [voice-first-ux](14-voice-first-ux.md) | Voice-first UX patterns |
| 15 | [habit-formation](15-habit-formation.md) | Habit formation & engagement |
| 16 | [local-vs-cloud](16-local-vs-cloud.md) | Local-first vs cloud architecture |
| 17 | [transcript-search](17-transcript-search.md) | Searchable transcripts & semantic search |
| 18 | [streaming-vs-batch](18-streaming-vs-batch.md) | Streaming vs batch transcription |
| 19 | [monetization](19-monetization.md) | Monetization & pricing |
| 20 | [conversational-voice-agents](20-conversational-voice-agents.md) | Conversational voice agent stacks |
