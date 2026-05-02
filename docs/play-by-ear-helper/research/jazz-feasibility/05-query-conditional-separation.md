# Query-Based and Conditional Separation / Transcription

## Summary

**Verdict: Research-grade feasible, production-grade marginal-to-poor for the specific use case of "isolate the trumpet from a 15s jazz quartet."**

Query-based and language-conditioned separation has matured rapidly (2022-2026) and now beats fixed-stem separators in flexibility — you really can prompt "the sound of a trumpet" or pass a 3s trumpet sample and get an audio output. However, all evaluated models report SI-SDR in the **5-10 dB** range on instrument-style benchmarks (MUSIC, AudioCaps-derived sets), which corresponds to clearly audible artifacts and partial leakage of piano/bass/drums into the trumpet stem. None of these systems is trained or benchmarked on real jazz quartet recordings; the dominant training data are AudioSet/AudioCaps-style clips and FUSS/MUSIC, which contain isolated instruments rather than tightly-arranged ensembles with overlapping harmonic content (piano comping under a trumpet line is a near-worst-case for these models).

For the practical problem ("get notes of the trumpet line"), the evidence points to two more reliable paths: (1) use **AudioShake's commercial wind-instruments stem** (the only system with documented production-quality jazz wind separation) feeding any AMT, or (2) **skip separation entirely** using a multi-instrument transcriber (MT3 / YourMT3+ / Jointist) that already emits per-instrument MIDI from the full mix. Open-source query-conditioned separators are useful as a fallback, a research baseline, or for sounds outside the standard stem vocabulary, but they are not the strongest available choice for trumpet specifically.

## Models

### AudioSep (Liu et al., 2023 — "Separate Anything You Describe")
- **Approach:** CLAP text encoder (QueryNet) + ResUNet SeparationNet, mask-based time-frequency separation. Trained on 14k hours of weakly-labeled audio (AudioSet, VGGSound, AudioCaps, WavCaps, Clotho, MUSIC, FSD50k).
- **License:** MIT (code), checkpoint `audiosep_base_4M_steps.ckpt` released; runs at 32 kHz.
- **Quality on instruments:** SI-SDR ~9.4 dB, SDRi ~10.5 dB on MUSIC dataset (single-instrument-vs-mixture benchmark, not jazz ensemble). AudioSep-CLIP variant ~9.75 SI-SDR. Strong vs. prior baselines, but MUSIC is an "easy" dataset of isolated YouTube instrument solos mixed pairwise — not representative of dense quartet arrangements.
- **"Separate the trumpet" prompt:** Yes, supported syntactically; the paper's demo page shows queries like "the sound of trumpet" and "violin and piano playing together." Quality varies sharply with how close the source resembles training distribution. Real jazz with simultaneous piano, bass, and drums shows audible bleed in informal listening.
- **Production-readiness:** Research-grade. Easy to run (HF + Replicate `cjwbw/audiosep`), but expect leakage and spectral holes on real ensembles.
- **Repo:** https://github.com/Audio-AGI/AudioSep

### LASS-Net (Liu et al., Interspeech 2022 — original LASS task)
- **Approach:** BERT text query + ResUNet separator. Predecessor to AudioSep; trained on AudioCaps captions only.
- **License:** Research code (MIT-style), small-scale.
- **Quality:** Substantially weaker than AudioSep. Mostly historical interest; superseded.
- **Note:** The "ICASSP 2023" framing in the prompt is approximate — the canonical paper is Interspeech 2022; ICASSP/DCASE picked up the task in 2023-2024.

### CLIPSep (Dong et al., ICLR 2023, Sony AI)
- **Approach:** CLIP image-text encoder conditions a separator. Trained on **unlabeled video** (audio-image pairs) with "noise-invariant training" to handle off-screen sounds.
- **License:** Research code on Sony's GitHub; demos at https://sony.github.io/CLIPSep/
- **Quality on music:** Designed for general sounds, not music. MUSIC-dataset performance trails AudioSep. Useful as a self-supervised baseline but not a serious candidate for trumpet extraction.

### Universal Source Separation / USS (Kong et al., 2023, "Universal Source Separation with Weakly Labelled Data", arXiv 2305.07447)
- **Approach:** Audio-tagging model (PANN/HTSAT) acts as the QueryNet; output class probabilities (or class-embedding) condition a separator. Supports all 527 AudioSet classes including "Trumpet," "Saxophone," "Piano," etc.
- **Quality:** Average SDRi 5.57 dB across 527 classes. Per-class numbers are uneven; instruments tend to land 4-8 dB SDRi.
- **License:** Apache-2.0; code in bytedance/music_source_separation.
- **Practical note:** This is essentially AudioSep's older cousin with a class-token query instead of free text. For "trumpet" the class is in AudioSet's vocabulary, so you can query it directly.

### CLAPSep (Ma et al., 2024, arXiv 2402.17455)
- **Approach:** Reuses CLAP for **multi-modal** queries — accepts text **or** an audio example (query-by-example). FiLM conditioning on a transformer masker.
- **License:** Research/MIT-style.
- **Quality:** Outperforms AudioSep on AudioCaps separation; supports negative queries ("everything except trumpet").
- **Why it matters for jazz:** Audio-query mode lets you provide a 2-3s isolated trumpet clip from the same player as a stronger conditioning signal than text — this is theoretically a better fit than language alone for timbre-specific separation.

### FlowSep (Yuan et al., ICASSP 2025, arXiv 2409.07614)
- **Approach:** **Generative** rectified-flow-matching separation in a VAE latent space, conditioned on FLAN-T5 text embeddings, BigVGAN vocoder for waveform.
- **License:** Code at https://github.com/Audio-AGI/FlowSep
- **Quality:** Beats AudioSep on AudioCaps benchmarks; reduces the "spectral hole / artifact" problem of mask-based methods. Trained on 1,680 hours.
- **Caveat:** Generative — it can hallucinate plausible-but-wrong notes. **For transcription this is dangerous**; you may get a clean-sounding trumpet stem with notes that weren't actually played. Mask-based separators distort but don't invent.

### Hyperellipsoidal Queries (Watcharasupat & Lerch, 2025, arXiv 2501.16171)
- **Approach:** Query is a region in embedding space (center + spread), letting you ask for "this trumbre and similar things." Trained on MoisesDB (multi-stem music dataset including brass).
- **Quality:** SOTA on MoisesDB query-based separation.
- **Best fit for music** of the recent batch — actually evaluated on music data (not AudioSet) and has a brass category in MoisesDB. Code availability uncertain (preprint).

### TUSS — Task-Aware Unified Source Separation (Saijo et al., MERL, ICASSP 2025, arXiv 2410.23987)
- **Approach:** Variable number of **learnable prompt tokens** specify which sources to separate; one model handles speech enhancement, music sep, sound-event sep, even contradictory tasks.
- **License:** Released by MERL at https://github.com/merlresearch/unified-source-separation (research license — check before commercial use).
- **Quality:** Strong unified results; a follow-up FasTUSS (2025) is faster.
- **Trumpet relevance:** Prompts are **learnable embeddings**, not free-text. So you'd need to add and train a "trumpet" prompt — not a zero-shot solution out of the box.

### Bandit-v2 (Watcharasupat et al.)
- **Approach:** Cinematic source separation: speech / music / SFX. Generalized bandsplit network (similar lineage to BS-RoFormer).
- **Use case:** Cinematic, **not** instrument-level. Irrelevant for trumpet-from-quartet.
- **Repo:** https://github.com/kwatcharasupat/bandit-v2 ; available on MVSep.

### Zero-Shot QbE (Chen et al., AAAI 2022)
- **Approach:** Provide a sample of the source you want as a **reference clip**; model separates similar timbre from a mixture. Works zero-shot on unseen classes.
- **License:** Open (MIT).
- **Quality:** Pioneering but modest SDR; pre-CLAP era. Useful conceptually — the QbE paradigm is a strong fit for jazz where you can grab a 3s isolated trumpet phrase from elsewhere in the recording.
- **Repo:** https://github.com/RetroCirce/Zero_Shot_Audio_Source_Separation

### iQuery (CVPR 2023)
- **Approach:** Audio-visual: instruments-as-queries with video frames. Requires synchronized video.
- **Relevance:** Only useful if you have video alongside audio.

### CodecSep, Hybrid-Sep, MARS-Sep, ClearSep (2025-2026 follow-ups)
- All use CLAP-derived embeddings + various improvements (codec latents, adversarial training, RL).
- ClearSep claims Re-SDR 18.4 dB vs AudioSep's 14.7 on real-world separation — the strongest recent number, but still a research result, not a packaged product.

### DCASE 2024 Task 9 (LASS challenge)
- Top system: **AudioSep-DP** — SI-SDR 7.35 dB at 32 kHz on real audio. Baseline 5.7 dB.
- These numbers reflect realistic-difficulty open-domain separation. **A 7 dB SI-SDR trumpet stem will have audible piano/drums leakage**; pitch detectors will pick up phantom notes.

## Instrument-Conditioned Transcription

### MT3 (Magenta, 2021)
- Multi-task multi-track AMT outputting per-instrument MIDI (program-token-based). **Not prompted** — it transcribes everything and assigns programs. You can post-filter for "trumpet" tokens.
- Jazz-quartet capability: limited; trained mostly on classical/Slakh.
- License: Apache-2.0.

### MR-MT3 (2024)
- Memory-retaining MT3 variant; mitigates instrument leakage between programs. Same query model — implicit per-instrument output, not user-prompted.

### YourMT3+ (Chang et al., MLSP 2024, arXiv 2407.04822)
- **Multi-channel decoder** + task queries. Adds **task-query-based training** — closer to a "transcribe instrument X" prompt model than vanilla MT3, though still operates over a fixed program vocabulary.
- License: MIT, repo https://github.com/mimbres/YourMT3
- Best open-source choice if you want "MIDI for trumpet" without explicit separation.

### Jointist (Cheuk, Choi, Kong et al., 2022/2023, arXiv 2302.00286)
- **Instrument-aware joint separation + transcription.** Detects instruments first, then conditions both a transcription head and a separation head on detected instrument set.
- Reports **+5 dB SDR** improvement on separation and >1 ppt transcription gain over MT3 by joint training.
- This is conceptually the closest thing to a turnkey "give me trumpet notes from a mix" pipeline in open research. License: research/MIT.

### Cerberus (Manilow et al., ICASSP 2020)
- Joint separation + transcription with a **third "head"** on a Chimera-style network. Tested on piano+guitar, piano+guitar+bass, +drums, +strings — **no trumpet/horn evaluation**.
- License: research; PyTorch reimpl at sweetcocoa/cerberus-pytorch.
- Older but proves the joint-task synergy. Superseded by Jointist for multi-instrument scenarios.

## End-to-End Approaches ("audio + instrument query → notes")

There is **no published model** that does exactly "give me a 15s jazz mix and the prompt 'trumpet' and outputs MIDI of just the trumpet" as a single neural net. The closest paradigms:

1. **Jointist** — implicitly does this: detects instruments → emits per-instrument piano rolls. You take the trumpet roll. Closest to end-to-end.
2. **YourMT3+** with task queries — emits multi-track MIDI; filter to the trumpet program.
3. **AudioSep → AMT pipeline** — two-stage, but you can compose them today.
4. **Score-informed source separation** (e.g., Bespoke Neural Networks, Ewert et al.) — flips the problem: given an approximate score, refine separation. Useful only after a first transcription pass.

No "language prompt → MIDI" end-to-end model targeting a specific instrument exists in the public literature as of May 2026.

## Practical Notes on Real-World Jazz

- **AudioShake commercially supports wind-instrument stems** (flute, sax, trumpet) and is the only documented production-quality wind separator. Commercial API; paid. Their messaging explicitly calls out wind instruments as historically hard for source separation and claims tuned analysis modules.
- **MUSIC dataset benchmarks overstate real-world performance** — they use isolated YouTube instrument clips mixed pairwise. Jazz quartets feature comping piano voicings overlapping the trumpet's harmonic series, drum cymbal energy in the trumpet's brilliance band, and walking bass partials hitting trumpet sub-fundamentals. Expect 3-5 dB **lower** SDR than benchmark numbers in this scenario.
- **Generative separators (FlowSep, diffusion-based) are risky for transcription** — they can hallucinate musically plausible content that isn't there.
- **Query-by-example (CLAPSep, Zero-Shot QbE, hyperellipsoidal) is theoretically a better fit than text** when you have access to even a brief solo trumpet phrase from the same player/recording, but reported numbers don't dramatically beat text-only.

## Verdict

For the play-by-ear helper's "isolate the trumpet, then transcribe" path:

1. **Best ROI today:** Use **YourMT3+ or Jointist directly on the mix** — skip separation. These models are designed for exactly this scenario and avoid compounding errors from a noisy separation stage.
2. **If separation is required (e.g., for re-listening):** AudioShake's commercial API is the only production-grade option for trumpet. AudioSep is the best open-source baseline but expect 7-10 dB SI-SDR on real jazz with audible artifacts.
3. **Promising research worth piloting:** Hyperellipsoidal Queries (music-trained), CLAPSep (audio-query mode), Jointist (joint pipeline). All MIT/research-licensed.
4. **Do not rely on:** FlowSep or other generative separators if the downstream task is transcription — hallucination risk.
5. **End-to-end "prompt → MIDI for one instrument" does not exist** as a single model; build it as a 2-stage pipeline (multi-instrument AMT + filter) or (separation + monophonic AMT).

Realistically, query-conditioned separation is **research-grade with promising trajectory** but not yet the reliable foundation a user-facing "play-by-ear helper for jazz trumpet" would want. The pragmatic 2026 stack is YourMT3+/Jointist for transcription, optionally with AudioShake or a fine-tuned AudioSep as a separation auxiliary.

## Sources

- AudioSep paper: https://arxiv.org/abs/2308.05037
- AudioSep repo: https://github.com/Audio-AGI/AudioSep
- AudioSep demo: https://audio-agi.github.io/Separate-Anything-You-Describe/AudioSep_arXiv.pdf
- AudioSep on Replicate: https://replicate.com/cjwbw/audiosep/readme
- LASS / LASS-Net (Interspeech 2022): https://arxiv.org/abs/2203.15147 ; https://www.isca-archive.org/interspeech_2022/liu22w_interspeech.pdf
- LASS demo page: https://liuxubo717.github.io/LASS-demopage/
- DCASE 2024 Task 9 (LASS challenge): https://dcase.community/challenge2024/task-language-queried-audio-source-separation
- DCASE 2024 baseline: https://github.com/Audio-AGI/dcase2024_task9_baseline
- DCASE 2024 caption-augmented winning report: https://arxiv.org/abs/2406.11248
- CLIPSep (ICLR 2023): https://arxiv.org/abs/2212.07065 ; https://sony.github.io/CLIPSep/
- USS / Universal Source Separation with Weakly Labelled Data (Kong et al.): https://arxiv.org/abs/2305.07447
- CLAPSep: https://arxiv.org/html/2402.17455v3
- FlowSep (ICASSP 2025): https://arxiv.org/abs/2409.07614 ; https://github.com/Audio-AGI/FlowSep
- Hyperellipsoidal Queries: https://arxiv.org/abs/2501.16171
- TUSS (Task-Aware Unified Source Separation): https://arxiv.org/abs/2410.23987 ; https://github.com/merlresearch/unified-source-separation
- TUSS demo: https://www.jonathanleroux.org/research/ICASSP2025-tuss/
- Bandit-v2: https://github.com/kwatcharasupat/bandit-v2
- Zero-Shot QbE Source Separation (AAAI 2022): https://cdn.aaai.org/ojs/20366/20366-13-24379-1-2-20220628.pdf ; https://github.com/RetroCirce/Zero_Shot_Audio_Source_Separation
- iQuery (CVPR 2023): https://openaccess.thecvf.com/content/CVPR2023/papers/Chen_iQuery_Instruments_As_Queries_for_Audio-Visual_Sound_Separation_CVPR_2023_paper.pdf
- CodecSep / Neural Audio Codecs for Prompt-Driven Universal Source Separation: https://www.arxiv.org/pdf/2509.11717v2
- Hybrid-Sep: https://arxiv.org/pdf/2506.16833
- MARS-Sep: https://jzndd.github.io/assets/pdf/MARS_Sep.pdf
- ClearSep (improvement on AudioSep, real-world): https://arxiv.org/html/2504.17782v1
- MT3: https://arxiv.org/abs/2111.03017 ; https://github.com/magenta/mt3
- MR-MT3: https://arxiv.org/html/2403.10024v1 ; https://github.com/gudgud96/MR-MT3
- YourMT3+: https://arxiv.org/abs/2407.04822 ; https://github.com/mimbres/YourMT3
- Jointist (joint training): https://arxiv.org/abs/2302.00286 ; https://arxiv.org/abs/2206.10805
- Cerberus: https://interactiveaudiolab.github.io/project/cerberus.html ; https://github.com/sweetcocoa/cerberus-pytorch
- Source Separation & AMT survey 2024: https://arxiv.org/abs/2412.06703
- AudioShake wind instruments: https://www.audioshake.ai/post/stems-for-wind-instruments
- 2025 AMT Challenge: https://ai4musicians.org/transcription/2025transcription.html
