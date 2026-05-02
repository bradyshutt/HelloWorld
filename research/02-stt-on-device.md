# On-Device Speech Recognition

## Summary

On-device speech-to-text (STT) is mature enough in 2025/2026 to be the default for a daily voice journal: Whisper-family models cover desktop and mobile, and platform-native APIs (Apple's new `SpeechAnalyzer` in iOS 26 and Android's `SpeechRecognizer`) provide low-friction, free, fully-local transcription. The main trade-off is accuracy vs. footprint: tiny/base Whisper models (~75-150 MB) run anywhere but yield 10-15% WER, while small/medium variants approach cloud-quality (~3-6% WER) at the cost of RAM and battery. For a privacy-respecting journal, a hybrid stack — native OS APIs on phone, `whisper.cpp`/MLX on desktop, Transformers.js for web — is the most pragmatic path.

## Landscape

The on-device STT space in 2026 is dominated by ports of OpenAI's Whisper, supplemented by older lightweight engines (Vosk) and resurgent first-party OS frameworks. Coqui STT was discontinued in late 2023/early 2024 and is no longer a viable option ([AssemblyAI](https://www.assemblyai.com/blog/top-open-source-stt-options-for-voice-applications)). Apple's WWDC25 introduced `SpeechAnalyzer`/`SpeechTranscriber`, a long-form, low-latency on-device model that reportedly transcribes 2.2x faster than Whisper with comparable quality ([MacStories](https://www.macstories.net/stories/hands-on-how-apples-new-speech-apis-outpace-whisper-for-lightning-fast-transcription/), [Apple Developer](https://developer.apple.com/documentation/speech/speechanalyzer)). The previous `SFSpeechRecognizer` API still works but is rate-limited (1000 requests/hour per device, ~1 minute audio per request) ([Apple Developer Forums](https://developer.apple.com/forums/thread/105405)).

## Models & Frameworks

- **whisper.cpp** — pure C/C++ port using ggml; runs on iOS via XCFramework, Android via NDK, plus desktop. Tiny model is 75 MB; Large-v3 needs ~3.9 GB RAM (down from ~10 GB in PyTorch). Small runs near real-time on recent iPhones, Medium near real-time on newer hardware ([whisper.cpp GitHub](https://github.com/ggml-org/whisper.cpp), [whisper.cpp #1093](https://github.com/ggml-org/whisper.cpp/discussions/1093)). Quantization shrinks size ~75% with negligible accuracy loss ([TildAlice](https://tildalice.io/whisper-quantization-mobile/)).
- **faster-whisper** — CTranslate2-based; ~5x faster than whisper.cpp on CPU for small.en (14 s vs 46 s on the same clip), but desktop/server-oriented, not mobile ([Modal blog](https://modal.com/blog/choosing-whisper-variants)).
- **MLX Whisper** — Apple Silicon only; reportedly 30-40% faster than whisper.cpp on M-series chips ([PyPI](https://pypi.org/project/mlx-whisper/), [Hylke Rozema](https://www.hylkerozema.nl/2026/02/24/local-audio-transcription-with-mlx-whisper-and-claude-on-apple-silicon/)).
- **Apple SpeechAnalyzer (iOS 26+)** — system-managed models, multi-language, designed for long-form/meeting audio, fully on-device ([Callstack](https://www.callstack.com/blog/on-device-speech-transcription-with-apple-speechanalyzer), [iOS 26 SpeechAnalyzer Guide](https://antongubarenko.substack.com/p/ios-26-speechanalyzer-guide)).
- **Android SpeechRecognizer** — built-in, free, works offline once language packs are downloaded; lower accuracy than Whisper-small and limited language coverage, but zero deployment cost ([Android Developers](https://developer.android.com/reference/android/speech/SpeechRecognizer), [WebRTC.ventures](https://webrtc.ventures/2025/03/real-time-speech-transcription-on-android-with-speechrecognizer/)).
- **Vosk** — Kaldi-based, 50 MB models, 20+ languages, official Android/iOS bindings; lighter and faster than Whisper-tiny but noticeably less accurate ([alphacephei.com](https://alphacephei.com/vosk/android), [VideoSDK](https://www.videosdk.live/developer-hub/stt/vosk-speech-recognition)).
- **Transformers.js + WebGPU** — Whisper-base (73M params) runs in browser via ONNX/WebGPU; WebGPU support is ~70% globally as of late 2024 ([HF blog](https://huggingface.co/blog/transformersjs-v3), [whisper-web](https://github.com/xenova/whisper-web)).

## Trade-offs

| Concern | Best option |
|---|---|
| Lowest latency on phone | Native OS API (SpeechAnalyzer / SpeechRecognizer) |
| Highest accuracy offline | Whisper large-v3 / turbo via whisper.cpp or MLX |
| Smallest footprint | Vosk (50 MB) or Whisper tiny.en quantized (~40 MB) |
| Cross-platform single codebase | whisper.cpp (C/C++ portable to iOS/Android/desktop) |
| Browser/PWA | Transformers.js with WebGPU |

Whisper tiny.en achieves ~5.6% WER on LibriSpeech clean and ~14.9% on noisier audio; Large-v3 hits 2.7% clean ([NovaScribe](https://novascribe.ai/how-accurate-is-whisper)). All on-device options preserve privacy (no audio leaves the device), reduce ongoing cost to zero, and work offline — a strong fit for a personal journal containing private thoughts.

## Recommendations for voice journal

1. **Mobile MVP**: use `SpeechAnalyzer` on iOS 26+ and `SpeechRecognizer` on Android. Both are free, low-latency, fully local, and need no model bundling — ideal for the speaking-practice loop where instant feedback matters more than perfect accuracy.
2. **Fallback / older iOS**: ship `whisper.cpp` with the small.en quantized model (~150 MB) for iOS <26 and for Android devices where the system recognizer is weak. Tiny.en is acceptable for short journal entries if storage is tight.
3. **Desktop/web companion**: MLX Whisper (Apple Silicon) or faster-whisper (other) for higher-accuracy re-transcription/summarization passes; Transformers.js + WebGPU for an in-browser entry point.
4. **Re-transcription pipeline**: run a higher-quality Whisper pass overnight on charger to refine the day's entries before the AI narration step — this hides large-model latency from the live UX.
5. **Avoid**: Coqui STT (discontinued) and any cloud-only API as the default path, given the journal's privacy-sensitive content.

Sources:
- [whisper.cpp (GitHub)](https://github.com/ggml-org/whisper.cpp)
- [Whisper accuracy WER data 2026 — NovaScribe](https://novascribe.ai/how-accurate-is-whisper)
- [Whisper quantization for mobile — TildAlice](https://tildalice.io/whisper-quantization-mobile/)
- [Choosing Whisper variants — Modal](https://modal.com/blog/choosing-whisper-variants)
- [mlx-whisper PyPI](https://pypi.org/project/mlx-whisper/)
- [Local transcription with MLX Whisper on Apple Silicon](https://www.hylkerozema.nl/2026/02/24/local-audio-transcription-with-mlx-whisper-and-claude-on-apple-silicon/)
- [Apple SpeechAnalyzer documentation](https://developer.apple.com/documentation/speech/speechanalyzer)
- [iOS 26 SpeechAnalyzer Guide — Anton Gubarenko](https://antongubarenko.substack.com/p/ios-26-speechanalyzer-guide)
- [On-Device Speech Transcription with Apple SpeechAnalyzer — Callstack](https://www.callstack.com/blog/on-device-speech-transcription-with-apple-speechanalyzer)
- [Apple's New Transcription APIs vs Whisper — MacStories](https://www.macstories.net/stories/hands-on-how-apples-new-speech-apis-outpace-whisper-for-lightning-fast-transcription/)
- [SFSpeechRecognizer rate limits — Apple Developer Forums](https://developer.apple.com/forums/thread/105405)
- [Android SpeechRecognizer reference](https://developer.android.com/reference/android/speech/SpeechRecognizer)
- [Real-Time Speech Transcription on Android — WebRTC.ventures](https://webrtc.ventures/2025/03/real-time-speech-transcription-on-android-with-speechrecognizer/)
- [Vosk Android documentation](https://alphacephei.com/vosk/android)
- [Vosk Speech Recognition Guide — VideoSDK](https://www.videosdk.live/developer-hub/stt/vosk-speech-recognition)
- [Top open source STT options 2026 — AssemblyAI](https://www.assemblyai.com/blog/top-open-source-stt-options-for-voice-applications)
- [Transformers.js v3 WebGPU — Hugging Face](https://huggingface.co/blog/transformersjs-v3)
- [whisper-web (Xenova)](https://github.com/xenova/whisper-web)
- [whisper.cpp ScribeAI iOS discussion](https://github.com/ggml-org/whisper.cpp/discussions/1093)
