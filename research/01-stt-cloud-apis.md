# Cloud Speech-to-Text APIs

## Summary

For a voice journal app, the strongest 2025/2026 options are **Deepgram Nova-3**, **AssemblyAI Universal-2**, and **OpenAI's gpt-4o-transcribe** family — all reaching sub-7% WER on real-world audio. Hyperscalers (Google, Azure, AWS) are still competitive on language breadth and enterprise compliance but tend to be pricier per minute and slower to iterate on accuracy. Because journaling is mostly single-speaker, monologue-style dictation, diarization and ultra-low streaming latency matter less than accuracy, cost on short clips, and clean punctuation. A pragmatic default is **gpt-4o-mini-transcribe** for batch (cheap, accurate, already in the OpenAI stack) with **Deepgram** as a streaming fallback if live transcription is added later.

## Landscape

The market has bifurcated into two camps. Specialized STT vendors — Deepgram, AssemblyAI, Speechmatics, and OpenAI's transcription endpoints — compete on accuracy/latency/price and ship new models every few months. Hyperscalers — Google Cloud Speech-to-Text, Azure Speech, AWS Transcribe — bundle STT with broader cloud ecosystems, prioritize enterprise features (compliance, regional residency, IAM), and lag on per-minute price ([Deepgram 2026 guide](https://deepgram.com/learn/best-speech-to-text-apis-2026), [AssemblyAI startups guide](https://www.assemblyai.com/blog/best-speech-to-text-apis-startups)).

In March 2025, OpenAI released `gpt-4o-transcribe` and `gpt-4o-mini-transcribe`, surpassing original Whisper on WER (4.1% vs 5.3% on FLEURS), plus a `gpt-4o-transcribe-diarize` variant ([TokenMix review](https://tokenmix.ai/blog/gpt-4o-transcribe-vs-whisper-review-2026)). Deepgram's Nova-3 became the first model to support 10-language *real-time multilingual* streaming in a single session ([Deepgram models docs](https://developers.deepgram.com/docs/models-languages-overview)). AssemblyAI's Universal-2 cut prices ~43% to $0.37/hr while improving alphanumeric and formatting accuracy ([AssemblyAI vs Deepgram](https://www.assemblyai.com/blog/assemblyai-vs-deepgram)).

## Comparison

| Provider | Batch $/min | Streaming $/min | Streaming latency | Languages | Diarization | Notes |
|---|---|---|---|---|---|---|
| **OpenAI gpt-4o-transcribe** | $0.006 | HTTP-chunked, 500–1500ms first chunk | ~50+ | Add-on (`-diarize`, 2.5×) | 1–2 min file minimum can inflate cost on short clips |
| **OpenAI gpt-4o-mini-transcribe** | $0.003 | Same as above | ~50+ | Via diarize variant | Recommended default by OpenAI |
| **Deepgram Nova-3** | $0.0043 | $0.0077 | sub-300ms | 10 real-time, more batch | Yes | 500 concurrent streams default |
| **AssemblyAI Universal-2** | ~$0.0042 effective | ~$0.0025 base | sub-300ms | 99+ batch, 6 streaming | Yes | Bills on session duration (~65% overhead on short calls) |
| **Google Cloud STT** | ~$0.016–0.024 | Streaming supported | Fast | 125+ | Yes | 300 concurrent streams/region cap |
| **Azure Speech** | $0.006 batch / $0.0167 real-time | WebSocket streaming | Moderate | 100+ | $0.30/hr add-on | Three modes: real-time, fast REST, batch |
| **AWS Transcribe** | $0.015 batch / $0.024 real-time | Streaming supported | Moderate | 54 | Up to 10 speakers (+20–40%) | 15-second billing blocks penalize short utterances |

Sources: [Deepgram pricing breakdown](https://deepgram.com/learn/speech-to-text-api-pricing-breakdown-2025), [VocaFuse 2025 comparison](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/), [BrassTranscripts AWS pricing](https://brasstranscripts.com/blog/aws-transcribe-pricing-per-minute-2025-better-alternative), [Azure Speech pricing](https://azure.microsoft.com/en-us/pricing/details/speech/), [OpenAI pricing](https://openai.com/api/pricing/).

## Trade-offs

- **Short-clip billing traps.** OpenAI enforces a 1–2 min minimum per file; AWS bills in 15-second blocks. For 8–30s journal snippets this can 2–5× the effective rate. Per-second billing (Deepgram, AssemblyAI) is friendlier ([George Mandis](https://george.mand.is/2025/06/openai-charges-by-the-minute-so-make-the-minutes-shorter/)).
- **Streaming vs batch.** Voice agents need <300ms; a journal app does not. Batch endpoints are 30–60% cheaper and more accurate.
- **Diarization need.** Mostly irrelevant for solo journaling — skip the add-on cost.
- **Vendor lock-in.** Hyperscalers integrate well with existing AWS/GCP/Azure stacks but are pricier per minute and slower-moving on model quality.
- **Multilingual.** If users journal in non-English languages, AssemblyAI (99+ batch) or Azure (100+) win on breadth; Deepgram leads on real-time multilingual.

## Recommendations for voice journal

1. **Default: `gpt-4o-mini-transcribe` for batch transcription.** $0.003/min, strong accuracy, simple API, and consolidates with the same OpenAI key likely used for the LLM narration step. Pad clips to ≥60s to avoid the minimum-file penalty, or batch multiple short entries into one upload.
2. **If cost-sensitive at scale: switch to Deepgram Nova-3 batch ($0.0043/min)** — per-second billing is gentler on short journal entries and accuracy is at parity.
3. **If/when adding live "talk-to-journal" mode**, add Deepgram streaming ($0.0077/min, sub-300ms) — it gives a responsive feel that helps users practice fluid speaking, which is one of the project's stated goals.
4. **Skip diarization** for the v1 single-user use case; revisit only if the app gains shared/family journals.
5. **Avoid AWS Transcribe and Google STT initially** — their pricing and short-clip billing don't favor a consumer journaling cadence.
6. **Punctuation + smart formatting matter more than raw WER** for the "AI narrates your day" feature. AssemblyAI Universal-2 and gpt-4o-transcribe both score well here ([AssemblyAI blog](https://www.assemblyai.com/blog/assemblyai-vs-deepgram)).

Sources:
- [Best Speech-to-Text APIs in 2026 — Deepgram](https://deepgram.com/learn/best-speech-to-text-apis-2026)
- [Speech-to-Text API Pricing Breakdown 2025 — Deepgram](https://deepgram.com/learn/speech-to-text-api-pricing-breakdown-2025)
- [AssemblyAI vs Deepgram: Accuracy & Speed](https://www.assemblyai.com/blog/assemblyai-vs-deepgram)
- [Top APIs for real-time speech recognition 2026 — AssemblyAI](https://www.assemblyai.com/blog/best-api-models-for-real-time-speech-recognition-and-transcription)
- [GPT-4o-Transcribe vs Whisper Review — TokenMix](https://tokenmix.ai/blog/gpt-4o-transcribe-vs-whisper-review-2026)
- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [Best Speech-to-Text APIs 2025 — VocaFuse](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/)
- [Azure Speech Pricing](https://azure.microsoft.com/en-us/pricing/details/speech/)
- [AWS Transcribe Pricing 2026 — BrassTranscripts](https://brasstranscripts.com/blog/aws-transcribe-pricing-per-minute-2025-better-alternative)
- [Cheapest Audio Transcription APIs 2025 — DEV](https://dev.to/fredpsantos33/cheapest-audio-transcription-apis-in-2025-whisper-via-api-vs-assemblyai-vs-deepgram-2f2a)
