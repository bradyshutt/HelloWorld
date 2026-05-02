# Short Clips and Audio LLMs

## Summary
A 15-second clip is workable for tempo/chord/note detection by dedicated MIR pipelines (Basic Pitch, Demucs, BeatNet, Beat This!) but is on the short side for reliable Krumhansl-style key estimation, which traditionally wants 30s+ — recent "signature of fifths" methods close some of that gap. Audio-capable LLMs in 2026 (Gemini 2.5/3 Pro, GPT-4o audio, Qwen2.5-Omni, Audio Flamingo 3, Music Flamingo) can describe music well and answer high-level questions, but **note-level transcription is not their strong suit**: recent benchmarks (MMAU-music, MMAU-Pro, MUSE, CMI-Bench) show even Gemini Pro lags task-specific MIR models, struggles with abstract pitch/harmony reasoning, frequently refuses pitch-extraction tasks (GPT-4o), and hallucinates notes when pushed beyond captioning. Music Flamingo (NVIDIA, Nov 2025) is the strongest open music LLM with 92% key detection and SOTA on MMAU-Music (76.83), but it still mainly outputs descriptions/chords rather than per-note MIDI. Anthropic's Claude does **not** accept raw audio file input for music as of 2026 — it works on pre-transcribed text, so it is not a candidate for this task. Verdict: use a dedicated AMT pipeline for the trumpet line and chord changes; an audio LLM is useful only as a high-level "second opinion" (key, style, instrumentation), not as the primary transcriber.

## Part 1: Short-Clip Robustness

### Pitch / note transcription
- **Basic Pitch (Spotify, ICASSP 2022)** is instrument-agnostic, polyphonic, and uses 20 ms frames internally with windowing for efficient processing — the model itself operates on short audio chunks, so 15 s is well within its comfortable range. There is no degradation from clip length itself; the limit is what the audio actually contains. Source: https://engineering.atspotify.com/2022/06/meet-basic-pitch , https://github.com/spotify/basic-pitch
- **Demucs** processes audio in ~10 s chunks internally with overlap, so 15 s is fine for source separation (e.g., isolating the trumpet stem).
- **Effect of clip boundaries**: starting/ending mid-phrase truncates note onsets/offsets at the edges. Basic Pitch handles this gracefully (per-frame outputs), but the first/last ~100 ms can have onset detection artifacts; padding the clip with silence is a common mitigation.

### Key estimation
- Classic Krumhansl–Schmuckler key-finding stabilizes "after 15–20 notes" of input. For a 15 s jazz clip at moderate tempo, 15–20 melodic events is realistic but not guaranteed — bebop solos hit it easily, slow ballads may not. Source: https://github.com/Corentin-Lcs/music-key-finder
- Research has explicitly used 15-second segments for key estimation specifically because key tends to remain stable over that horizon without modulation. Source: https://www.mdpi.com/2076-3417/12/21/11261
- The "signature of fifths" approach is competitive with Krumhansl–Kessler / Temperley / Albrecht–Shanahan **on very short fragments**, which is the most directly relevant 2022+ result for this use case.
- **Practical caveat**: jazz harmony with extensions/altered dominants and chromatic passing tones is harder than pop; expect 80–85% accuracy on 15 s jazz vs. 90%+ on 15 s pop.

### Tempo estimation
- Tempo is the most clip-length-tolerant task. Even 8–10 s usually suffices for steady-tempo material. madmom's RNN/DBN tempo estimators and BeatNet are designed for streaming and work fine on 15 s offline. Source: https://madmom.readthedocs.io/en/v0.16/modules/features/beats.html

### Beat / downbeat tracking
- **madmom DBNBeatTrackingProcessor**, **BeatNet** (CRNN + particle filtering, ISMIR 2021), and **Beat This!** (CPJKU) all run on 15 s without issue. BeatNet has both real-time and offline modes; Beat This! uses madmom's DBN parameters as defaults.
- Downbeat estimation gets harder on short clips because it needs ~2 measures of context; a 15 s clip at 120 BPM = 30 beats ≈ 7 bars in 4/4, which is enough.
- Time signature inference from 15 s is **less reliable** than tempo — needs at least one full hypermetric phrase; jazz waltzes vs. 4/4 swing are usually distinguishable, but odd meters (5/4, 7/8) need longer context.
- Sources: https://github.com/CPJKU/beat_this , https://github.com/mjhydri/BeatNet

### Chord recognition
- 15 s is fine for frame-level chord recognition (2–4 frames per chord at typical jazz harmonic rhythm). The main limit is that smoothing/HMM post-processing benefits from longer sequences — short clips give "spikier" outputs that a Viterbi pass would normally clean up.
- Tempo must be detected correctly first; chord-rate inference depends on it.

### Bottom line for Part 1
A 15 s window is **good enough** for tempo, beat, chord, and per-note pitch with dedicated tools; it is **borderline** for key (use signature-of-fifths method) and time signature; it is **fine** for source separation. Padding with 0.5–1 s of silence on each side mitigates edge effects.

## Part 2: Audio LLMs for AMT

### Gemini 2.5 Pro / Gemini 3 Pro (Google, API)
- Native audio input; supports MP3/WAV/M4A/FLAC; consumes 25 tokens/s of audio; up to ~8 hours of audio in context.
- Strongest generalist on music perception in 2025–26 evals. On the **MUSE benchmark** (Oct 2025), Gemini Pro hit 100% on Oddball Detection, 96.67% on Rhythm Matching, but only 46.67% on Meter Identification (vs 73.30% human). CoT raised Pitch Shift Detection from 81.36% → 98.33%. Sources: https://arxiv.org/abs/2510.19055
- On MMAU-music, Gemini Pro is among the top scorers; Music Flamingo (NVIDIA) reports 76.83 on MMAU-Music as SOTA.
- Will not output a clean MIDI/note list. Asked for "what notes is the trumpet playing" it tends to produce a chord-symbol or scale-degree description rather than a precise pitch sequence, and **hallucinates** when pushed for exact transcription.
- Cost: Gemini 2.5 Flash audio input ≈ $1.00/M tokens × 25 tok/s = ~$0.0015/min ≈ ~$0.0004 per 15 s clip. Gemini 3 Pro is ~5–10× more expensive.
- Latency: 1–4 s for a 15 s clip via API.
- Sources: https://ai.google.dev/gemini-api/docs/audio , https://ai.google.dev/gemini-api/docs/pricing , https://simonwillison.net/2025/Nov/18/gemini-3/

### GPT-4o / GPT-4o-audio (OpenAI, API)
- Accepts audio. **Refuses** structured music tasks like Chord Classification, Pitch Extraction by Lyrics, and HEAR Tonic Classification at high rates — refusal is the documented failure mode. Per the preliminary GPT-4o voice study, this is because the model judges the task too hard and refuses to avoid hallucination.
- "Instrument Pitch Classification" (multiple-choice) it will attempt; open-ended pitch sequences it will not.
- Not a viable transcriber for trumpet notes.
- Source: https://arxiv.org/html/2502.09940v1

### Claude (Anthropic) — 2026 status
- As of early 2026, Claude has voice **mode** (talking interface, ElevenLabs voices), but **does not accept raw audio file uploads** for music analysis. The official cookbook pattern is: transcribe with Deepgram/Whisper externally → feed text to Claude.
- **Not a candidate** for "give Claude a 15 s clip."
- Sources: https://www.assemblyai.com/blog/claude-3-5-sonnet-with-audio-data-python , https://platform.claude.com/cookbook/third-party-deepgram-prerecorded-audio

### Qwen2-Audio / Qwen2.5-Omni (Alibaba, open-weights)
- Open audio LLM, can analyze music for genre, instruments, mood; SOTA on AIR-Bench music subset at release.
- On **MMAU-music**, Qwen2.5-Omni-7B currently leads the public leaderboard at 0.692 — strong, but this is multiple-choice/QA, not note transcription.
- On CMI-Bench's structured MIR tasks (pitch estimation on Nsynth-Pitch, key detection), Qwen-Audio "performs far worse than reported in its original paper, likely due to the absence of structured task tokens in the prompt" — confirms it cannot just be asked in plain English to output notes.
- Cost: free if self-hosted (7B model fits on 1× consumer GPU).
- Sources: https://qwenlm.github.io/blog/qwen2-audio/ , https://arxiv.org/html/2506.12285 , https://llm-stats.com/benchmarks/mmau-music

### Audio Flamingo 3 / Audio Flamingo Next (NVIDIA, open-weights)
- Trained on ~50M audio-text pairs, fully open. **MMAU = 72.4%**, MMAU-Pro Music = 61.7%. Sets new SOTA on speech ASR (WER 1.54 on LibriSpeech test-clean).
- Better than Qwen2.5-Omni on MMAU overall, but still QA-flavored — not a per-note transcriber.
- Source: https://research.nvidia.com/labs/adlr/AF3/ , https://arxiv.org/pdf/2507.08128

### Music Flamingo (NVIDIA, Nov 2025, open-weights)
- Built on Audio Flamingo 3 backbone with a music-specific post-training recipe (MF-Skills + chain-of-thought + GRPO RL). The **strongest open music LLM** in 2025–26.
- Reported: **92% key detection accuracy**, 90.86% Medley-Solos-DB instrument recognition, 97.1 on Music Instruct, **76.83 on MMAU-Music** (SOTA), 65.6 on MMAU-Pro-Music, 74.58 on MuChoMusic.
- Identifies chord progressions, modulations, harmonic complexity. Qualitative reviews praise "faithful chord tracking, better localization of structural events."
- Still not a note-level transcriber — outputs descriptive text, chord symbols, structure. Doesn't emit MIDI.
- Sources: https://research.nvidia.com/labs/adlr/MF/ , https://arxiv.org/abs/2511.10289 , https://huggingface.co/nvidia/music-flamingo-hf

### AudioPaLM (Google research, 2023)
- Speech-focused (ASR, S2ST). No music transcription capability documented. Superseded by Gemini's audio path.
- Source: https://google-research.github.io/seanet/audiopalm/examples/

### SALMONN (Tsinghua + ByteDance)
- Whisper + BEATs encoders → Q-Former → LLM. Good at music captioning and audio QA; not a note transcriber. Used as baseline in CMI-Bench, where it falls behind supervised MIR models on pitch/key tasks.
- Source: https://arxiv.org/abs/2310.13289

### MU-LLaMA / MusiLingo (academic)
- MERT encoder + LLaMA/Vicuna; trained on Music Instruct dataset (60k Q&A from MusicCaps). Good at music QA and captioning.
- Per CMI-Bench, "instruction-following LLMs fall significantly short of task-specific supervised MIR models, except in music captioning." Pitch and key tasks are weak.
- Sources: https://github.com/shansongliu/MU-LLaMA , https://arxiv.org/html/2309.08730v3 , https://arxiv.org/html/2506.12285

### GAMA
- Generalist audio LM; not stronger than Audio Flamingo 3 on music. Same caveat: descriptive, not transcriptive.

### "Could 'Gemini 3, please tell me the notes the trumpet plays in this 15s clip' actually work in 2026?"
- **Partially.** Gemini 3 will produce a confident-sounding answer. It will likely identify the instrument (trumpet) correctly, name the key, list a chord progression, describe the contour ("ascending bebop line over a ii-V"), and may emit a short pitch sequence ("Bb4, D5, F5, ...").
- But comparing to ground truth, expect:
  - Instrument ID: ~95% reliable (trumpet vs sax is easy).
  - Key: 70–85% on 15 s of jazz.
  - Chords: roughly accurate at the chord-symbol level but misses extensions/alterations.
  - Note sequence: **unreliable** — the MUSE benchmark and "LLMs can read music but struggle to hear it" (PMLR 2026) show models reason well over MIDI but are "notably brittle" on audio. Expect hallucinated notes, wrong octaves, and missed fast passages.
- For a play-by-ear study tool, this is **not accurate enough** to be the primary transcriber.

### Cost comparison (per 15 s clip)
- Gemini 2.5 Flash audio API: ~$0.0004
- Gemini 3 Pro audio API: ~$0.003–0.01
- GPT-4o audio: ~$0.01 (and frequent refusals on pitch tasks)
- Self-hosted Qwen2.5-Omni / Audio Flamingo 3 / Music Flamingo: marginal cost ≈ free, ~1–3 s GPU time on 1× consumer GPU
- Dedicated AMT pipeline (Demucs + Basic Pitch + madmom): ~0.5–2 s CPU/GPU time, no API cost

### Quality comparison
- Dedicated AMT (Demucs → Basic Pitch on isolated trumpet stem → madmom for beat → chordino/BTC for chords) gives **note-event-level output with timestamps**. Note F1 on jazz solos is typically 70–85% with Basic Pitch, higher with specialized models like MT3.
- Audio LLM gives **prose description plus an unreliable note list**. No timestamps. Hallucination is the dominant error mode.
- LLMs **win** at: key, style, instrumentation, structural description, "what's interesting about this solo."
- LLMs **lose** at: exact pitches, exact onsets, exact chord voicings.

### Recent 2025–2026 evals
- **MMAU / MMAU-Pro** (Aug 2025): Music Flamingo 76.83 on MMAU-Music; AF3 72.4 overall; Qwen2.5-Omni 0.692.
- **MUSE benchmark** (Oct 2025): Gemini Pro near-ceiling on basic perception, but 46.67% on Meter ID vs 73.30% human; CoT inconsistent.
- **CMI-Bench** (ISMIR 2025): all instruction-following LLMs trail supervised MIR models on pitch estimation, key detection, melody extraction.
- **"LLMs can read music, but struggle to hear it"** (PMLR 2026): models near-perfect on MIDI inputs, "notably brittle" on audio inputs for the same tasks. Gemini Pro best of the bunch.

## Verdict
For the play-by-ear use case (15 s jazz quartet → trumpet notes / piano chords), the right architecture in 2026 is:

1. **Dedicated MIR pipeline as primary**: Demucs (separate stems) → Basic Pitch on trumpet stem (notes) → Chordino or BTC on full mix or piano stem (chords) → madmom/BeatNet/Beat This! (beats, tempo) → signature-of-fifths key estimation.
2. **Audio LLM as optional secondary**: Send the 15 s clip plus the MIR output to Gemini 2.5/3 Pro or Music Flamingo for: sanity-check key, name the style ("medium-swing bebop"), describe the harmonic flavor ("ii-V-I with tritone sub"), suggest practice pointers. **Do not** rely on the LLM to enumerate notes.
3. **Skip Claude** for audio (no native audio input for music as of 2026).
4. **Skip GPT-4o** for note tasks (high refusal rate on pitch extraction).

The 15 s clip length is fine for the dedicated pipeline. The bottleneck for accuracy is jazz harmonic complexity, not duration.

## Sources
- Krumhansl-Schmuckler key estimation: https://github.com/Corentin-Lcs/music-key-finder
- Signature-of-fifths key on short fragments: https://www.mdpi.com/2076-3417/12/21/11261
- KeyFinder thesis: https://www.ibrahimshaath.co.uk/keyfinder/KeyFinder.pdf
- Basic Pitch: https://engineering.atspotify.com/2022/06/meet-basic-pitch , https://github.com/spotify/basic-pitch
- madmom beat tracking: https://madmom.readthedocs.io/en/v0.16/modules/features/beats.html
- BeatNet: https://github.com/mjhydri/BeatNet
- Beat This!: https://github.com/CPJKU/beat_this
- Gemini audio API: https://ai.google.dev/gemini-api/docs/audio
- Gemini API pricing: https://ai.google.dev/gemini-api/docs/pricing
- Gemini 3 Pro audio review: https://simonwillison.net/2025/Nov/18/gemini-3/
- GPT-4o voice mode music limitations: https://arxiv.org/html/2502.09940v1
- Claude audio cookbook (text-only): https://platform.claude.com/cookbook/third-party-deepgram-prerecorded-audio
- Qwen2-Audio: https://qwenlm.github.io/blog/qwen2-audio/ , https://arxiv.org/abs/2407.10759
- Audio Flamingo 3: https://research.nvidia.com/labs/adlr/AF3/ , https://arxiv.org/pdf/2507.08128
- Music Flamingo: https://research.nvidia.com/labs/adlr/MF/ , https://arxiv.org/abs/2511.10289 , https://huggingface.co/nvidia/music-flamingo-hf
- SALMONN: https://arxiv.org/abs/2310.13289 , https://github.com/bytedance/SALMONN
- MU-LLaMA: https://github.com/shansongliu/MU-LLaMA
- MusiLingo: https://arxiv.org/html/2309.08730v3
- AudioPaLM: https://google-research.github.io/seanet/audiopalm/examples/
- MMAU benchmark: https://mmaubench.github.io/ , https://llm-stats.com/benchmarks/mmau-music
- MMAU-Pro: https://arxiv.org/html/2508.13992v1
- MUSE benchmark: https://arxiv.org/abs/2510.19055 , https://github.com/brandoncarone/MUSE_music_benchmark
- MARBLE: https://arxiv.org/abs/2306.10548 , https://marble-bm.shef.ac.uk/
- CMI-Bench: https://arxiv.org/abs/2506.12285 , https://github.com/nicolaus625/CMI-bench
- "LLMs can read music, but struggle to hear it" (PMLR 2026): https://proceedings.mlr.press/v303/carone26a.html
- "Evaluating MLLMs on Core Music Perception Tasks": https://arxiv.org/html/2510.22455v1
- "Can LLMs Reason in Music?": https://arxiv.org/html/2407.21531v1
- Audio LLM hallucination survey (Interspeech 2024): https://www.isca-archive.org/interspeech_2024/kuan24_interspeech.pdf
