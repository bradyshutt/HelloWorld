# Voice Activity Detection & Turn-Taking

## Summary

For a voice journal where users describe their day in free-form monologue, Voice Activity Detection (VAD) and end-of-turn detection are what make the experience feel natural rather than walkie-talkie. The 2025/2026 consensus is that small DNN-based VADs (Silero v5/v6, Picovoice Cobra) substantially outperform the legacy WebRTC GMM VAD in noisy or real-world conditions, while pure VAD-only endpointing should be supplemented with an STT- or LLM-based "end-of-turn" model for conversational use. For a journaling app specifically, push-to-talk is the safest default UX, with optional hands-free continuous mode gated by VAD plus generous silence thresholds (700–1500 ms) to tolerate the natural pauses people take while thinking.

## Techniques

- **Frame-level VAD** classifies short audio frames (10–32 ms) as speech or non-speech. WebRTC VAD uses Gaussian Mixture Models; Silero, Cobra, and TEN-VAD use small neural nets. Silero v5/v6 operate on fixed 512-sample (32 ms) windows at 16 kHz ([Silero VAD repo](https://github.com/snakers4/silero-vad)).
- **Endpointing** decides when the user has finished a turn. The simplest approach is "speech then N ms of silence." LiveKit notes pure VAD endpointing tends to add latency and miss intent ([LiveKit turn detection](https://docs.livekit.io/agents/build/turns/)).
- **STT-based endpointing** uses the recognizer's own silence/finality signals (e.g., Deepgram, AssemblyAI, Google STT). LiveKit recommends STT endpointing as the default for production agents ([AssemblyAI on endpointing](https://www.assemblyai.com/blog/turn-detection-endpointing-voice-agent)).
- **Model-based end-of-turn detection** uses a small transformer that looks at partial transcripts plus prosody to decide if the user is "done." LiveKit's open-weights turn detector hits ~85% TPR and runs locally in <500 MB RAM ([LiveKit turn detector](https://docs.livekit.io/agents/logic/turns/turn-detector/)).
- **Noise suppression** improves VAD reliability. RNNoise (Xiph) is a tiny RNN that runs in real time on a Raspberry Pi and is shipped in OBS, Mumble, and Discord-style stacks; dataset was refreshed in Jan 2025 ([RNNoise project](https://jmvalin.ca/demo/rnnoise/), [xiph/rnnoise](https://github.com/xiph/rnnoise)).

## Libraries Compared

| Library | Type | Strengths | Weaknesses |
|---|---|---|---|
| **Silero VAD v6** (Aug 2025) | Open-source DNN, MIT, ONNX/TorchScript | ~87% TPR @ 5% FPR, 6000+ languages, ~2 MB model, streaming iterator, 3× faster than v4 ([release notes](https://github.com/snakers4/silero-vad/releases)) | Slightly heavier than WebRTC; fixed 32 ms window |
| **WebRTC VAD** | GMM, C, BSD | Tiny, ubiquitous, near-zero CPU | ~50% TPR @ 5% FPR, lots of false positives in noise, unmaintained ([Picovoice 2025 comparison](https://picovoice.ai/blog/best-voice-activity-detection-vad-2025/)) |
| **Picovoice Cobra v2.1** (Sep 2025) | Commercial DNN, on-device | ~99% accuracy claim, cross-platform SDKs, .NET added in 2025, 1/5 the false alarms of v2.0 ([Cobra v2.1 announcement](https://picovoice.ai/blog/voice-activity-detection-accuracy-improvement/)) | Paid license for production |
| **TEN-VAD** | Open-weights DNN (Hugging Face) | Newer entrant, competitive accuracy ([TEN-VAD model card](https://huggingface.co/TEN-framework/ten-vad)) | Smaller community |
| **RNNoise** | Noise suppressor (not a VAD) | Real-time denoise, runs on Pi, GPL/BSD-style ([RNNoise](https://github.com/xiph/rnnoise)) | Speech-only training; not a VAD by itself |

A common production stack is **RNNoise → Silero VAD → STT endpointing → optional LLM turn-detector**.

## UX patterns: push-to-talk vs continuous

- **Push-to-talk (PTT)** — user holds or taps a mic button. Best for journaling because users often pause to think; PTT eliminates premature cutoffs, gives explicit privacy ("mic is off when I'm not pressing"), and avoids hot-mic anxiety. It is the recommended default for explicit-control voice flows ([Lollypop Studio VUI best practices 2025](https://lollypop.design/blog/2025/august/voice-user-interface-design-best-practices/)).
- **Tap-to-start, VAD-to-stop** — user taps once, app records until VAD detects N ms of silence. Good middle ground for hands-free journaling while cooking/commuting. Use a longer silence threshold (1.0–1.5 s) than chat agents (typically 500–800 ms) because journaling has more reflective pauses ([LiveKit turns](https://docs.livekit.io/agents/build/turns/)).
- **Continuous / wake-word** — always-listening mode behind a wake word ("Hey Journal"). Highest convenience, highest privacy cost. Show a persistent recording indicator ([Naskay VUI 2025](https://naskay.com/blog/voice-user-interfaces-2025-smarter-touchless-design/)).
- **Barge-in / interruption** — when the AI narrates back the day, VAD on the input stream lets the user cut in. Required for any "epic story narration" replay mode.

## Recommendations

1. **Default to Silero VAD v6** in ONNX form. It is free, MIT-licensed, ~2 MB, runs on CPU, and gives DNN-grade accuracy. WebRTC VAD is only worth using for ultra-constrained embedded targets.
2. **Ship two modes:** PTT as the default for journaling sessions, plus an optional hands-free mode using Silero with a 1000–1500 ms `min_silence_duration_ms` to respect thinking pauses.
3. **Layer endpointing:** VAD for "speech started/ended" segmentation, STT finality for sentence boundaries, and (optionally) a turn-detector model when the AI is in conversational follow-up mode.
4. **Pre-process with RNNoise** if users record on phones in cafés/streets — VAD precision improves measurably.
5. **For commercial/enterprise builds**, evaluate Picovoice Cobra v2.1; its accuracy is the headline benchmark and they offer signed SDKs.
6. **UX guardrails:** always show a clear recording indicator, allow the user to scrub/delete the last segment, and never auto-send without a brief "ending in 3…2…1" or visible stop affordance.

Sources:
- [Picovoice — Best VAD 2025/2026](https://picovoice.ai/blog/best-voice-activity-detection-vad-2025/)
- [Picovoice — Cobra v2.1 release](https://picovoice.ai/blog/voice-activity-detection-accuracy-improvement/)
- [Silero VAD GitHub](https://github.com/snakers4/silero-vad)
- [Silero VAD releases (v5 / v6)](https://github.com/snakers4/silero-vad/releases)
- [LiveKit — Turn detection docs](https://docs.livekit.io/agents/build/turns/)
- [LiveKit — Turn detector plugin](https://docs.livekit.io/agents/logic/turns/turn-detector/)
- [AssemblyAI — Turn detection / endpointing](https://www.assemblyai.com/blog/turn-detection-endpointing-voice-agent)
- [RNNoise (Xiph)](https://jmvalin.ca/demo/rnnoise/)
- [RNNoise GitHub](https://github.com/xiph/rnnoise)
- [TEN-VAD model](https://huggingface.co/TEN-framework/ten-vad)
- [Lollypop — VUI best practices 2025](https://lollypop.design/blog/2025/august/voice-user-interface-design-best-practices/)
- [Naskay — Voice UI 2025](https://naskay.com/blog/voice-user-interfaces-2025-smarter-touchless-design/)
