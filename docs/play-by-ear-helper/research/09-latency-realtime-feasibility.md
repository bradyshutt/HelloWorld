# Latency & Real-Time Feasibility

## Summary
Full-quality offline transcription of a 3-minute clip is fast enough to feel "instant-ish": Spotify Basic Pitch runs faster than real-time on a modern CPU (a 3-minute song typically finishes in well under a minute, often 10-30 seconds), and Demucs source separation adds ~1.5x track length on CPU but only ~5-15 seconds on a recent GPU/Apple Silicon (a 7-minute song separates in ~12 s on an M4 Max, ~5 s on an RTX 3090 with TensorRT). True low-latency live transcription (sub-30 ms, "notes appear as they are played") is feasible only for piano-only streaming models like Mobile-AMT and the 2025 streaming/online AMT systems, which currently sit at 128-320 ms delay; general polyphonic instrument-agnostic streaming below 30 ms is not yet a solved problem. For Play by Ear Helper, the realistic UX is a progress bar of 5-60 seconds for a 3-minute clip (depending on whether Demucs runs and on what hardware), not a real-time scrolling staff. Heavy transformer models like MT3 process audio in non-overlapping segments and are not designed for streaming; they are the slowest of the bunch.

## Inference Times (Typical 3-min Audio)

| Model | CPU | GPU / Accelerator | Notes |
|---|---|---|---|
| **Spotify Basic Pitch** (lightweight CNN, ~17 MB) | "Faster than real-time on most modern computers" - empirically ~10-30 s for 3 min | Negligible advantage; model is too small to benefit much | Ships TF, CoreML, TFLite, ONNX runtimes; 20 ms frames |
| **Demucs v3 (original)** | ~1.5x track length (~4.5 min for a 3-min clip on i5/i7 8th-11th gen) | ~30 s for 3.27 min on a single GPU (issue #1) | CPU is 5-10x slower than GPU |
| **htdemucs (v4 hybrid)** | ~1.5x track length | ~12 s for a 7-min track on M4 Max (~34x faster than real-time); ~5 s on RTX 3090 with TensorRT | Default high-quality model |
| **htdemucs_ft** (fine-tuned, 4 sub-models) | ~6x track length (4x htdemucs) | ~4x htdemucs (~20 s for 3 min on RTX 3090) | Marginal SDR gain for 4x cost |
| **mdx_extra** | similar to htdemucs | similar to htdemucs | MDX-challenge variant, MusDB test in training set |
| **MT3 / YourMT3+** (T5-based transformer) | Multiple minutes for 3-min clip; segment-based | Seconds per segment on A100; not benchmarked publicly for full-track RTF | Splits audio into non-overlapping segments due to memory; not streaming-friendly |
| **Onsets & Frames** (CNN+LSTM) | Roughly real-time; smaller than MT3 | ~real-time | Piano only; older baseline |
| **Mobile-AMT (EUSIPCO 2024)** | Real-time on smartphone (82.9% lower compute than SOTA) | n/a (designed for mobile) | Piano only; in-the-wild robust |
| **Streaming Piano (arXiv 2503.01362, 2025)** | ~real-time, no segmentation needed | n/a | CNN encoder + 2 transformer decoders; chunked |
| **CREPE-full** (pitch only, monophonic) | ~10-30 ms per 10 ms frame depending on size | <1 ms on GPU | 90x slower than SwiftF0 per pitch-benchmark; offers Tiny/Small/Med/Large/Full variants |
| **CREPE-tiny / pYIN / YIN** | <1-3 ms/sec audio | n/a | Real-time capable but **monophonic only** - not for chords |
| **aubio / Praat (DSP pitch)** | 2.8 ms per 1 s of audio (Praat) | n/a | Monophonic; ~30 s for 3-min song with aubio Python |

### What this means for a 3-minute song end-to-end

| Pipeline | Hardware | Wall-clock |
|---|---|---|
| Basic Pitch only | CPU laptop | ~10-30 s |
| Basic Pitch only | M-series Mac / mid GPU | ~5-15 s |
| Demucs (htdemucs) + Basic Pitch | CPU laptop | ~5-7 min (Demucs dominates) |
| Demucs + Basic Pitch | M4 Max / RTX 3090 | ~10-30 s total |
| Demucs htdemucs_ft + MT3 | CPU laptop | tens of minutes - impractical |
| Demucs + MT3 | A100 GPU | ~1-3 min |
| WASM Basic Pitch (browser) | Modern laptop browser | 30-90 s, ~2-3x slower than native (ONNX Runtime Web SIMD+threads gives ~3.4x CPU speedup vs single-thread WASM) |

## Real-Time / Streaming Approaches

### Feasible today
- **Monophonic real-time pitch** (one note at a time): YIN, pYIN, CREPE-tiny, SwiftF0, Praat all run easily under 10 ms latency in the browser via Web Audio API. AudioWorklets give animation-frame-rate (~60 FPS) updates. These cannot handle chords.
- **Piano-only streaming AMT**: A burst of 2024-2025 papers explicitly target streaming:
  - **Mobile-AMT** (EUSIPCO 2024): real-time on phones, 82.9% compute reduction.
  - **Streaming Piano Transcription** (arXiv 2503.01362, 2025): CNN encoder + dual transformer decoders for onset/offset, processes variable-length audio without segmentation.
  - **"Pairing Real-Time Piano Transcription"** (arXiv 2505.05078, 2025) and **"Minimum Latency Real-Time Piano Transcription"** (arXiv 2509.07586, 2025): explicitly targeting <30 ms latency by removing non-causal layers. Today's online models still sit at 128-320 ms delay.
  - **jdasam/online_amt** (GitHub): runnable PyTorch demo with web visualization, based on autoregressive multi-state note model (ISMIR 2020).
- **Real-time low-latency source separation**: HS-TasNet (L-Acoustics, 2024) achieves 23 ms latency with SDR 4.65 on MusDB - usable for vocal removal but quality is well below offline Demucs.

### Not yet feasible
- **Real-time, instrument-agnostic, polyphonic AMT** that matches Basic Pitch / MT3 quality. No production system does this. AnthemScore advertises real-time microphone *input* but transcription is still batch/file-oriented.
- **Streaming MT3 / YourMT3+**. These transformer models depend on full-segment context and split audio into non-overlapping windows because of memory; they are inherently offline.
- **Real-time Demucs htdemucs**. Possible chunked, but CPU users see ~1.5x track length, so streaming would fall behind unless GPU-assisted.

### Chunked inference and the latency-quality tradeoff
- Demucs natively supports chunked streaming (chunked spectrogram passes), but htdemucs has internal transformers requiring meaningful context (~7-second windows are typical). Setting smaller windows degrades SDR.
- Basic Pitch processes 2-second windows with overlap; you can run it on rolling 2-3 s buffers, but the 22 ms hop and CNN context mean per-chunk latency is 100-300 ms even on fast hardware - acceptable for "near-live" but not for instrument-style monitoring.
- Quantized ONNX/TFLite/CoreML versions of Basic Pitch can run on phones; expect roughly real-time on a recent iPhone, ~2-3x real-time on Apple Silicon.

## Recommended UX

For Play by Ear Helper, the realistic pattern is **"upload/record then progress bar, then show full sheet"**, not real-time scrolling:

1. **Default flow (file or recorded clip)**: progress bar with stages
   - "Separating instruments..." (Demucs, 10-90 s on GPU; skip on weaker hardware)
   - "Detecting notes..." (Basic Pitch, 5-30 s)
   - "Quantizing & rendering score..." (instant)
   - Total target: **under 60 s for a 3-minute song on a mid-range laptop**, under 10-15 s on Apple Silicon / GPU.
2. **Optional "live preview" mode** (monophonic): use CREPE-tiny or pYIN in the browser via Web Audio + WASM/AudioWorklet to show a live pitch contour while the user hums or plays a single line. Make it clear this is single-note only.
3. **Stretch goal**: piano-only "live mode" using a streaming AMT model (Mobile-AMT-style) running in WASM / CoreML. Latency 150-300 ms is achievable today; sub-30 ms requires the cutting-edge 2025 papers and is research-grade, not product-ready.
4. **Avoid promising real-time polyphonic instrument-agnostic transcription** in v1. The state of the art does not support it at acceptable quality.
5. **UX copy**: "Analyzing your clip... usually 10-30 seconds" beats a fake real-time animation. If running on CPU-only, set expectations: "This may take up to a minute on slower devices."

## Sources

- Spotify Basic Pitch repo & engineering blog: https://github.com/spotify/basic-pitch , https://engineering.atspotify.com/2022/06/meet-basic-pitch , https://basicpitch.spotify.com/about
- Basic Pitch TS (browser/JS port): https://github.com/spotify/basic-pitch-ts
- Demucs original timing issue (CPU/GPU numbers): https://github.com/facebookresearch/demucs/issues/1
- Demucs streaming inference issue: https://github.com/facebookresearch/demucs/issues/515
- Demucs ONNX/TensorRT port (RTX 3090 ~5 s/song): https://github.com/sevagh/demucs.onnx , https://huggingface.co/MansfieldPlumbing/Demucs_v4_TRT
- Demucs on Apple Silicon (M4 Max, 12 s for 7 min): https://medium.com/@andradeolivier/i-ported-demucs-to-apple-silicon-it-separates-a-7-minute-song-in-12-seconds-6c4e5cffb5c3
- Demucs-GUI usage notes (~1.5x CPU RTF): https://github.com/CarlGao4/Demucs-Gui/blob/main/usage.md
- HTDemucs variant comparison: https://stemsplitter.github.io/research/model-comparison/
- Real-time low-latency separation HS-TasNet (23 ms): https://www.l-acoustics.com/wp-content/uploads/2024/04/real_time_demixer_2024_04_19.pdf , https://arxiv.org/html/2402.17701v1
- MT3 paper: https://arxiv.org/abs/2111.03017 , https://github.com/magenta/mt3
- YourMT3+: https://arxiv.org/html/2407.04822v1
- Onsets & Frames: https://arxiv.org/abs/1710.11153 , https://magenta.tensorflow.org/onsets-frames
- Mobile-AMT (EUSIPCO 2024): https://eurasip.org/Proceedings/Eusipco/Eusipco2024/pdfs/0000036.pdf , https://openreview.net/forum?id=1QTsNlmlDk
- Streaming Piano Transcription (2025): https://arxiv.org/pdf/2503.01362
- Pairing Real-Time Piano Transcription (2025): https://arxiv.org/pdf/2505.05078
- Minimum-Latency Real-Time Piano Transcription (2025): https://arxiv.org/abs/2509.07586
- Online AMT live demo: https://github.com/jdasam/online_amt
- CREPE: https://github.com/marl/crepe , https://arxiv.org/abs/1802.06182
- Pitch detection benchmarks (SwiftF0 vs CREPE vs Praat): https://github.com/lars76/pitch-benchmark
- pYIN: https://code.soundsoftware.ac.uk/projects/pyin
- ONNX Runtime Web (WASM SIMD + threads, 3.4x): https://opensource.microsoft.com/blog/2021/09/02/onnx-runtime-web-running-your-machine-learning-model-in-browser/ , https://onnxruntime.ai/docs/tutorials/web/
- AnthemScore (real-time mic input, batch transcription): https://www.lunaverus.com/ , https://www.lunaverus.com/documentation
