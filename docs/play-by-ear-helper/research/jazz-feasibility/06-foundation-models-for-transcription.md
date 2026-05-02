# Foundation / Self-Supervised Audio Models for Transcription

## Summary

Foundation audio models (MERT, MuQ, MusicFM, Jukebox, etc.) are **strong general-purpose
representation extractors** for music understanding tasks (tagging, key, beat, genre, instrument
classification, single-note pitch). They are **not, by themselves, transcribers**. None of the
public models ships a multi-instrument note-level head that produces something usable for our
"trumpet notes / piano chords from a 15s jazz quartet" use case.

The realistic 2026 path is:

- **Use a specialized transcription model** (MT3, YourMT3+, hFT-Transformer, Basic Pitch) as the
  primary engine, and only consider foundation-model features as an **auxiliary input** to a
  trained AMT head.
- **Foundation models excel at the "what is this"** layer (instruments, chords as labels, key,
  tempo) and are quite useful for chord-symbol recognition on jazz; they are **weak at**
  frame-level note transcription.
- **Audio LLMs (Gemini 2.5/3, GPT-4o, Claude)** as of 2026 can describe music, identify chords,
  estimate key/tempo, and sometimes name instruments, but they **cannot reliably emit a correct
  note sequence** from real audio. They perform near ceiling on MIDI inputs and crash on raw audio
  for anything pitch-precise. They are not a drop-in transcriber.
- For our jazz quartet use case, the practical recipe in 2026 is: specialized AMT model (MT3-class)
  + an LLM/MERT-derived chord-recognition head, not a single foundation model.

---

## Models

### MERT (Li et al., ICLR 2024)
- **License:** Code Apache-2.0; weights on Hugging Face under CC-BY-NC 4.0 (research only).
- **Sizes:** 95M and 330M parameters (`m-a-p/MERT-v1-95M`, `m-a-p/MERT-v1-330M`).
- **Training:** ~160k hours of music audio at 24 kHz; HuBERT-style masked prediction with two
  teachers (RVQ-VAE acoustic teacher + Constant-Q Transform musical teacher to inject pitch /
  harmonic bias).
- **What it's good at (MARBLE benchmark):** beat tracking, key detection, music tagging,
  genre / emotion classification, **NSynth single-note pitch (~94.4% on MERT-330M)**, NSynth
  instrument classification, singer ID. Strong on local-level tasks.
- **What it's bad at:** It is a representation model, not a transcriber. No public head outputs
  a polyphonic piano roll or note-level events. MARBLE has chord recognition (HookTheory, MIREX
  rules) and melody extraction tasks that MERT does well on as a probed embedding, but those are
  classification/labelling, not symbolic note output.
- **Jazz OOD:** Training corpus is dominated by Western pop. Anecdotally fine on jazz solos
  (it sees pitch and timbre well), but no published jazz-specific evaluation. PiJAMA / JAAH are
  the obvious fine-tuning targets but I found no published MERT-on-jazz transcription paper.

### MuQ (Tencent AI Lab, 2025, IEEE TASLP)
- **License:** Code MIT; weights (`OpenMuQ/MuQ-large-msd-iter`) CC-BY-NC 4.0.
- **Size:** ~300M parameters; companion `MuQ-MuLan` is a ~700M music-text contrastive model
  (CLAP-style, EN+ZH).
- **Training:** Pretrained on the Million Song Dataset (open-source MuQ saw ~0.9k hours, paper
  reports larger internal runs). Uses Mel Residual Vector Quantization targets.
- **What it's good at:** Currently SOTA on MARBLE — beats MERT and MusicFM on the 9 evaluated
  downstream tasks. Best-reported model is "MuQiter" with avg MARBLE score 77.0.
  Particularly strong on genre, singer ID, vocal technique, music structure analysis,
  instrument classification.
- **Transcription head:** None published. Same caveat as MERT — it is a feature extractor.
- **Jazz OOD:** MSD and Music4All are pop-leaning; no jazz-specific evaluation reported.

### Jukebox / JukeMIR (Castellon et al., ISMIR 2021; Donahue, ISMIR 2022 "Sheet Sage")
- **License:** Jukebox under Noncommercial Use License (research only); ~5B params.
- **Approach:** Extract activations from Jukebox-5B's middle layers as features, then train a
  shallow probe.
- **Headline result:** "27% stronger performance on melody transcription" (Sheet Sage / Donahue
  ISMIR 2022) when using Jukebox features as input to a Transformer — best published evidence
  that a generative pretrained model helps **monophonic** transcription. Probes also gain ~30%
  on tagging / key / genre / emotion vs. handcrafted features.
- **Caveats:** Jukebox-5B is *huge* and slow to extract features from (a few seconds of audio →
  many seconds on a GPU). Used by Spotify's LLark as the audio frontend for the same reason —
  it's the strongest open music representation, but expensive.

### MULE (Pandora Media, ISMIR 2022) — *not ByteDance*
- **License:** BSD-3 (open repo at `PandoraMedia/music-audio-representations`).
- **Approach:** Slowfast-based supervised + self-supervised audio embeddings learned from a
  large internal music corpus.
- **Strengths:** Good general-purpose music tagging / similarity embedding; competitive on MIR
  tagging benchmarks.
- **Transcription:** Not designed for it; treat as a tagging embedding only.

### CLAP / LAION-CLAP (Wu et al., ICASSP 2023; Microsoft CLAP 2022)
- **License:** LAION-CLAP code CC0; weights CC-BY-4.0. Also `laion/larger_clap_music`
  fine-tuned on music+text.
- **Architecture:** Two-tower contrastive (HTS-AT audio encoder + RoBERTa text encoder).
- **What it's good at:** Zero-shot music tagging, instrument classification by text prompt
  ("a recording of a muted trumpet"), captioning retrieval. Recent work shows LAION-CLAP has
  the strongest alignment with human-perceived timbre on instrument and DSP-effect axes.
- **Transcription:** Not directly applicable — CLAP gives one global embedding per clip, no time
  resolution. Useful for the **"what instruments are present"** sub-task in our pipeline, not for
  notes.

### AudioMAE (Meta, NeurIPS 2022)
- **License:** Apache-2.0 (FAIR); weights public.
- **Approach:** ViT-style masked-autoencoder on log-mel spectrograms, trained primarily on
  AudioSet (general audio, not music).
- **Strengths:** SOTA on AudioSet/ESC-50/SPC-2 classification; good general audio features.
- **Transcription:** Not specialized for music; the constant-Q / pitched signal is not its
  inductive bias. Underperforms MERT/MuQ on MARBLE pitched tasks. AudioMAE++ (2025) improved on
  it but still general-audio.

### MusicFM (Won, Hung, Le — published at ICASSP 2024)
- **License:** Apache-2.0; weights public on GitHub `minzwon/musicfm`.
- **Size:** Conformer encoder, BEST-RQ-style training.
- **Training:** 8k hours from FMA (Creative Commons) plus optional MSD.
- **Strengths:** Beats MERT on several MARBLE tasks at the time; explicitly designed as a
  general music foundation model. Good base for adding task heads.
- **Transcription:** Used as a frozen encoder + simple head for beat tracking (BeatFM, 2025).
  No published note-level AMT head, but its frame-rate output (~25 Hz) is compatible with
  transcription decoders.
- **Jazz OOD:** FMA has a real jazz subset (~500h CC tracks), so MusicFM is the most "jazz-aware"
  foundation model by training data, though still pop-dominant.

### w2v-BERT (Google, 2021)
- **Origin:** Speech (Libri-Light 60k hours), not music. Combines contrastive + MLM.
- **Music relevance:** Mostly used as a *speech* foundation; the architectural pattern (BEST-RQ
  derivative) is what MusicFM and MuQ build on. There's no production "w2v-BERT-for-music"
  release we could find — that role is filled by MusicFM and MuQ.

### EnCodec / SoundStream (Meta 2022 / Google 2021)
- **License:** EnCodec under MIT (Meta); SoundStream not openly released, but the architecture
  is replicated in DAC (Descript Audio Codec, MIT).
- **Role:** Neural audio codecs producing RVQ tokens. Used as the **acoustic teacher inside
  MERT and AudioLM/MusicGen**. Their codes are useful as targets for SSL but as standalone
  features are weaker than MERT/MuQ for MIR tasks (the codec is optimized for reconstruction,
  not semantics).
- **Transcription:** Not directly. AMT systems have used SoundStream/EnCodec embeddings as
  inputs but with mixed results.

### BEATs (Microsoft, ICML 2023)
- **License:** Code MIT (in `microsoft/unilm/beats`); weights public.
- **Size:** ~90M params; iterative training of acoustic tokenizer + audio SSL model.
- **Strengths:** SOTA on AudioSet; strong general audio classifier.
- **Transcription / music:** Like AudioMAE — general audio, not music-pitched. Underperforms
  music-specific FAEs on MARBLE pitch and chord tasks.

---

## Building Transcription Heads

### Has anyone built an instrument-specific note transcription head on MERT/MuQ?

Short answer: **No public, peer-reviewed system that takes MERT/MuQ features and outputs
multi-instrument MIDI**, as of late 2025 / early 2026. What does exist:

- **MARBLE chord-estimation probe** uses MERT features → small MLP, achieves competitive chord
  recognition on HookTheory. This is the closest off-the-shelf "harmonic transcription" use of
  MERT.
- **Sheet Sage / Donahue (ISMIR 2022)** uses Jukebox features (not MERT) with a Transformer head
  for **melody transcription** — a single monophonic line. Result: +27% over CLMR/musicnn
  baselines.
- **Piano Transcription by Hierarchical Language Modeling with Pretrained Roll-based Encoders**
  (Li, Zang, Kong — arXiv 2501.03038, Jan 2025): uses pretrained piano-roll encoders (not generic
  music FAEs) plus an LM decoder. Confirms that "the choice of encoder has a much more substantial
  effect on overall performance than the size of the language model."
- **"Do Foundational Audio Encoders Understand Music Structure?"** (Sony, ICASSP 2026,
  arXiv 2512.17209): benchmarks 11 FAEs on music structure analysis. Conclusion: SSL+MLM on music
  data (MERT-, MuQ-, MusicFM-class) is the best FAE family for structure tasks. Notes that FAE
  features improve "music tagging, piano transcription, music source separation, music
  super-resolution" — but always in combination with a task-specific head, not standalone.
- **YourMT3+** (2024) and the **2025 AMT Challenge** (OpenReview NG187AZ71W) keep MT3-style
  T5 encoder-decoders trained from scratch on synthetic + real data; foundation-model frontends
  did not dominate the leaderboard. Two of eight teams beat MT3 baseline; jazz / popular music
  were called out as future work, not current strengths.

### Could we fine-tune a foundation model on jazz to get a jazz-aware transcriber?

Plausible, but **not a weekend project**:

- **Data:** PiJAMA (~244 albums, jazz piano transcriptions, MIDI), JAAH (113 tracks, chord
  annotations), Weimar Jazz Database (456 monophonic solo transcriptions), URMP (multi-instrument
  but classical, includes trumpet + sax). For trumpet-only or quartet work, real
  multi-instrument jazz with aligned MIDI is *very* scarce; you'd likely have to synthesize
  (NES-MDB-style or RWC + jazz MIDI rendered through soundfonts).
- **Architecture:** Frozen MERT-330M or MusicFM as encoder → conv/LSTM/Transformer head producing
  pitch + onset + offset per instrument channel (Onsets-and-Frames or MT3 token-stream style).
  Spotify's LLark already proves the encoder-frozen-LLM-head pattern works for music QA.
- **Compute:** Training the head with frozen MERT features fits comfortably on a single A100
  (24 hrs-ish for a small head); full fine-tune of MERT-330M on jazz data is doable with
  LoRA / partial unfreezing on 1-2 A100s. *Pretraining from scratch* is out of the question
  (160k hours took ~64 GPUs for weeks).

### Compute cost: feature extractor vs. training from scratch

| Approach | Training compute | Inference cost |
|---|---|---|
| MERT-95M frozen + small head | hours-days on 1 GPU | ~50ms per second of audio on A100 |
| MERT-330M frozen + head | days on 1-2 GPUs | ~150ms per second of audio |
| MuQ-large frozen + head | days on 1-2 GPUs | similar to MERT-330M |
| Jukebox-5B (JukeMIR) features | extraction is the bottleneck | seconds per second of audio, A100-class GPU required |
| MusicFM frozen + head | hours-days on 1 GPU | similar to MERT-95M |
| MT3 trained from scratch | ~16 TPU-v3 days | ~real-time on a T4 |
| MERT pretraining from scratch | infeasible without a research budget (~weeks on 64 A100s) | n/a |

For a hobby project, a **MERT-95M or MusicFM frozen encoder + small CNN/Transformer head trained
on PiJAMA/JAAH** is the best ROI. It does **not** require pretraining.

### Recent (2024-2026) papers building on FAEs for AMT / chord recognition

- Li et al., **Piano Transcription by Hierarchical Language Modeling with Pretrained Roll-based Encoders**, arXiv 2501.03038, Jan 2025.
- Sony, **Do Foundational Audio Encoders Understand Music Structure?**, arXiv 2512.17209, ICASSP 2026.
- **YourMT3+** (Chang et al., 2024), arXiv 2407.04822.
- **MR-MT3: Memory Retaining Multi-Track Music Transcription**, arXiv 2403.10024, 2024.
- **2025 ICME AMT Challenge results** (OpenReview NG187AZ71W).
- **MuQ** paper (Zhu et al., arXiv 2501.01108, IEEE TASLP 2025).
- **BeatFM** (arXiv 2508.09790, 2025) — beat tracking head on MusicFM; demonstrates the pattern.
- **CMI-Bench** (arXiv 2506.12285, ISMIR 2025) — evaluates audio LLMs on MIR tasks including
  pitch estimation and instrument classification.

---

## Audio LLMs (Gemini, GPT-4o, Claude, Qwen2-Audio)

### Gemini 2.5 Pro / 3 Pro
- Native audio in. Excellent ASR, decent music event detection (can name a song, identify
  instruments, describe genre).
- **Music-perception evaluation (Donahue et al., arXiv 2510.22455, 2025):** Gemini 2.5 Flash
  and Pro tested on Syncopation Scoring, Transposition Detection, Chord Quality Identification.
  **Near-ceiling on MIDI input, large accuracy drops on raw audio.** Explicit conclusion:
  "transcription/onset tracking and pitch-salience are the primary bottlenecks. Multimodal LLMs
  reason effectively over symbolic music data, yet still fail to listen reliably."
- **Will not give you reliable note sequences** for a real jazz quartet.

### GPT-4o
- Native audio in. CMI-Bench / WavLLM evaluations show GPT-4o **refuses many music tasks**
  ("pitch extraction by lyrics") and is outperformed by Qwen2-Audio-7B-Instruct, WavLLM on most
  music-specific tasks. Will do "instrument pitch classification" (a multiple-choice setup) but
  refuses open-ended pitch sequence transcription. Verdict: not reliable for our use case.

### Claude (3.5 / 3.7 / 4.x)
- As of early 2026, Claude **does not natively accept audio inputs** in the public API. Audio
  must be transcribed externally first. Cannot listen to a clip and tell you trumpet notes.

### Qwen2-Audio-7B-Instruct (Alibaba)
- Open weights, Apache-2.0. Does have an explicit "music note analysis" task in its training mix
  inherited from Qwen-Audio (MNA). On CMI-Bench, performs better than GPT-4o on music tasks but
  **still well below specialized supervised models** on pitch estimation and instrument
  classification. Likely overfits to MTG-Jamendo-style pop. Not a credible jazz transcriber out
  of the box.

### LLark (Spotify, 2023)
- LLama-2-7B + frozen Jukebox-5B audio frontend; instruction-tuned for music QA. Good at
  captioning / reasoning, not at frame-level transcription. Code/weights open.

### Bottom line on audio LLMs

For "tell me what notes the trumpet just played": **none of the off-the-shelf audio LLMs are
trustworthy in 2026.** They will hallucinate notes. They are useful for high-level chord names,
key, tempo, instrument inventory, mood — and as a *re-ranker* / *consistency check* on the output
of a real AMT system.

---

## Verdict

For the play-by-ear-helper use case (15s jazz quartet → trumpet notes / piano chords):

1. **Don't build the transcriber on top of MERT/MuQ alone.** Use a specialized AMT model (MT3,
   YourMT3+, hFT-Transformer for piano, Basic Pitch for monophonic) as the workhorse.
2. **Use MERT-330M or MusicFM features as auxiliary input** to a small head if you want to
   improve chord / instrument-identification reliability, especially on jazz harmonies that MT3
   was not trained on. CC-BY-NC license is fine for a research demo, problematic for commercial.
3. **For chord symbol recognition specifically**, MERT + MARBLE-style probe trained on JAAH
   (jazz chord dataset) is a credible 1-2 week build. This is probably the single best
   foundation-model use case for jazz in our pipeline.
4. **For audio LLMs**, expect them to give you genre / instruments / chords-as-text /
   tempo / mood — **not** a note sequence. Use Gemini 2.5/3 Pro or Qwen2-Audio for the
   "describe what's happening" sidebar, not the transcription engine.
5. **Fine-tuning on jazz is feasible** but data-limited. PiJAMA covers piano well; trumpet/sax
   transcription requires synthesizing training data from MIDI-rendered soundfonts or relying on
   URMP's small classical multi-instrument set. Don't expect SOTA results; expect
   "good enough for a hobbyist hint system."

The bet for 2026 is: **MT3-class transcriber as primary + MERT/CLAP-derived chord & instrument
heads as secondary + Gemini for human-readable description.** No single foundation model solves
this end-to-end yet.

---

## Sources

- [MERT paper (arXiv 2306.00107)](https://arxiv.org/abs/2306.00107)
- [MERT GitHub](https://github.com/yizhilll/MERT)
- [MERT-v1-330M on Hugging Face](https://huggingface.co/m-a-p/MERT-v1-330M)
- [MERT-v1-95M on Hugging Face](https://huggingface.co/m-a-p/MERT-v1-95M)
- [MuQ paper (arXiv 2501.01108)](https://arxiv.org/abs/2501.01108)
- [MuQ GitHub](https://github.com/tencent-ailab/MuQ)
- [OpenMuQ on Hugging Face](https://huggingface.co/OpenMuQ/MuQ-large-msd-iter)
- [JukeMIR](https://github.com/p-lambda/jukemir)
- [JukeMIR paper (arXiv 2107.05677)](https://arxiv.org/abs/2107.05677)
- [Sheet Sage / Donahue ISMIR 2022, melody transcription via generative pre-training](https://archives.ismir.net/ismir2022/paper/000058.pdf)
- [MULE (Pandora) repo](https://github.com/PandoraMedia/music-audio-representations)
- [LAION-CLAP](https://github.com/LAION-AI/CLAP)
- [larger_clap_music](https://huggingface.co/laion/larger_clap_music)
- [AudioMAE (Meta)](https://github.com/facebookresearch/AudioMAE)
- [AudioMAE paper (arXiv 2207.06405)](https://arxiv.org/abs/2207.06405)
- [MusicFM paper (arXiv 2311.03318)](https://arxiv.org/abs/2311.03318)
- [MusicFM GitHub](https://github.com/minzwon/musicfm)
- [BEATs paper (arXiv 2212.09058)](https://arxiv.org/abs/2212.09058)
- [BEATs in microsoft/unilm](https://github.com/microsoft/unilm/blob/master/beats/README.md)
- [w2v-BERT (arXiv 2108.06209)](https://arxiv.org/abs/2108.06209)
- [SoundStream (arXiv 2107.03312)](https://arxiv.org/abs/2107.03312)
- [MARBLE benchmark (NeurIPS 2023)](https://proceedings.neurips.cc/paper_files/paper/2023/file/7cbeec46f979618beafb4f46d8f39f36-Paper-Datasets_and_Benchmarks.pdf)
- [MARBLE GitHub](https://github.com/a43992899/MARBLE-Benchmark)
- [Do Foundational Audio Encoders Understand Music Structure? (arXiv 2512.17209)](https://arxiv.org/abs/2512.17209)
- [Piano Transcription by Hierarchical LM with Pretrained Roll-based Encoders (arXiv 2501.03038)](https://arxiv.org/abs/2501.03038)
- [YourMT3+ (arXiv 2407.04822)](https://arxiv.org/html/2407.04822v1)
- [MR-MT3 (arXiv 2403.10024)](https://arxiv.org/html/2403.10024v1)
- [2025 AMT Challenge writeup](https://openreview.net/pdf?id=NG187AZ71W)
- [MT3: Multi-Task Multitrack Music Transcription](https://arxiv.org/abs/2111.03017)
- [BeatFM (arXiv 2508.09790)](https://arxiv.org/html/2508.09790)
- [CMI-Bench (arXiv 2506.12285)](https://arxiv.org/abs/2506.12285)
- [Evaluating MLLMs on Core Music Perception Tasks (arXiv 2510.22455)](https://arxiv.org/html/2510.22455)
- [LLark (Spotify Research)](https://research.atspotify.com/2023/10/llark-a-multimodal-foundation-model-for-music)
- [LLark paper (arXiv 2310.07160)](https://arxiv.org/abs/2310.07160)
- [Qwen2-Audio Technical Report (arXiv 2407.10759)](https://arxiv.org/html/2407.10759v1)
- [Qwen2-Audio-7B-Instruct on Hugging Face](https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct)
- [Gemini Audio (DeepMind)](https://deepmind.google/models/gemini-audio/)
- [PiJAMA (jazz piano transcriptions)](https://transactions.ismir.net/articles/10.5334/tismir.162)
- [Weimar Jazz Database](https://jazzomat.hfm-weimar.de/dbformat/dboverview.html)
- [JAAH (jazz chord dataset)](https://mtg.github.io/JAAH/)
