# Polyphonic Music Transcription — Feasibility & State of the Art

## Summary
Polyphonic Automatic Music Transcription (AMT) is **partially feasible** for the "Play by Ear Helper" use case in 2026, but with sharp caveats. Solo-instrument transcription (especially clean piano) is essentially solved — state-of-the-art models hit ~96.7% onset F1 on MAESTRO. However, the target use case ("song from the radio → sheet music") sits at the hardest end of the problem: dense, real-world, multi-instrument, vocal-bearing pop/rock recordings show **note-level F1 drops of 14-20 percentage points** due to genre and sound variation, and SOTA multi-instrument models (YourMT3+, MT3) score around 0.83 multi-F1 on synthesized Slakh2100 but degrade significantly on real recordings. Practical reviewer consensus (e.g., MusicRadar on Songscription, Mar 2025): "humans will be doing all the serious music transcription for the foreseeable future." A pragmatic MVP should target **single-instrument extraction from already-isolated stems** (use Demucs first, then transcribe), set user expectations for editing, and avoid claims of accuracy on full-band mixes.

## Key Findings

### 1. Solo Piano Transcription — Essentially Solved
- **hFT-Transformer (Toyama et al., ISMIR 2023)** achieves **96.72% onset F1 on MAESTRO v3.0.0**, beating the previous Onsets and Frames system (94.80%). It also reports the first pedal onset F1 benchmark on MAESTRO at **91.86%**.
  - Source: https://archives.ismir.net/ismir2023/paper/000024.pdf
- More recent non-hierarchical Transformers (arXiv 2404.09466, Nov 2024) and sparse-attention variants (arXiv 2509.09318, 2025) report further improvements at lower compute cost.
- **Caveat:** these numbers are on MAESTRO, which is studio-quality solo piano captured via Disklavier (audio + perfect MIDI ground truth). Real-world piano recordings with reverb, mic noise, or accompaniment perform substantially worse.

### 2. Multi-Instrument Transcription — Improving Fast on Synthetic Data
- **MT3 (Google Magenta, 2021)**: First general-purpose multi-instrument transformer. Treats AMT as a seq2seq problem outputting MIDI-like tokens. Onset-Offset F1 around 0.66 on Slakh2100 segments.
  - Source: https://arxiv.org/pdf/2111.03017 / https://github.com/magenta/mt3
- **YourMT3+ (Chang et al., MLSP 2024, arXiv 2407.04822)**: Hierarchical attention transformer + Mixture of Experts. Significantly outperforms MT3 and PerceiverTF on URMP, Slakh, and 10 public datasets. Best variant ("YPTF.MoE+ Multi") reports the strongest multi-AMT scores on Slakh2100 to date.
  - Source: https://arxiv.org/abs/2407.04822 / http://eecs.qmul.ac.uk/~simond/pub/2024/ChangEtAl-MLSP-2024.pdf
- **MR-MT3 (2024, arXiv 2403.10024)**: Adds memory retention to mitigate "instrument leakage" (notes assigned to the wrong instrument), a known MT3 failure mode.
- **2025 AMT Challenge (ai4musicians.org)**: New benchmark with 76 unseen pieces (~20s each, up to 3 of 8 instruments). Eight teams submitted; only **two beat the MT3 baseline**. The MusicFM + multi-decoder system reached **Slakh2100 multi-instrument F1 ≈ 0.83** and slightly outperformed YourMT3+ on the held-out competition data — suggesting YourMT3+ may be **overfit to Slakh**.
  - Source: https://ai4musicians.org/transcription/2025transcription.html / https://openreview.net/pdf?id=NG187AZ71W

### 3. Where AMT Breaks Down — The Real-World Gap
- **Genre/sound bias (Marták et al., J. Audio Speech Music Proc., 2025/2026)**: Systematic study found a **note-level F1 drop of ~20 percentage points due to sound (timbre/recording) variation and ~14 points due to genre**. Models trained on classical piano collapse on rock, jazz, world music.
  - Source: https://link.springer.com/article/10.1186/s13636-025-00428-z / https://arxiv.org/pdf/2512.14602
- **Spotify Basic Pitch (open-source baseline)**: Frame note accuracy ~63% and note-level F1 ~52% on vocals; ~79% F1 on GuitarSet (isolated guitar). Performance "drops significantly" on dense polyphonic full-band mixes per the README and reviews.
  - Source: https://github.com/spotify/basic-pitch / https://engineering.atspotify.com/2022/6/meet-basic-pitch
- **Commercial tool reality check (MusicRadar review of Songscription, 2025)**: "Works best on beginner-level classical repertoire and very simple pop songs recorded on solo piano. If you give it other instruments, more than one instrument at a time, or music with any kind of expressive timekeeping, it struggles." Headline quote: *"humans will be doing all the serious music transcription for the foreseeable future."*
  - Source: https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review
- **Specific failure modes:**
  - Vocals with vibrato/melisma → pitch tracking fails or produces noisy MIDI.
  - Distorted electric guitar → harmonics confuse pitch detection.
  - Drums in mix → bleed into pitched-instrument transcriptions.
  - Reverb / lossy MP3 / radio compression → degrade onset detection.
  - Dense chords (e.g., piano accompaniment under a band) → note dropping and instrument leakage.

### 4. Standard Benchmarks — Quick Reference
- **MAESTRO v3** — solo classical piano, ~200 hours, perfect MIDI alignment via Disklavier. SOTA onset F1 ~96.7%.
- **MAPS** — solo piano (synthesized + recorded). Mostly saturated.
- **Slakh2100** — 2,100 multi-track mixes synthesized from MIDI using sample-based synths. SOTA multi-F1 ~0.83. **Synthetic — does not reflect real recordings.**
- **MusicNet** — 34 hours of real classical chamber recordings with weak alignment. Recent gains: +6-7% onset F1 on guitar/strings/organ/reeds (PF2N, 2025).
- **GuitarSet, URMP, Cerberus4** — smaller specialized sets, used for cross-dataset evaluation.
  - Sources: https://paperswithcode.com/sota/music-transcription-on-slakh2100 / http://www.slakh.com/

### 5. Recent Research Frontiers (2023-2026)
- **Token-based seq2seq transformers** (MT3 family) dominate multi-instrument AMT.
- **Hierarchical / sparse / MoE encoders** (hFT, YourMT3+, sparse-attention 2025) cut compute and improve frequency-time modeling.
- **Foundation-model encoders** (MusicFM in the 2025 AMT Challenge) — using pretrained self-supervised audio backbones is a strong new direction.
- **Streaming/real-time AMT**: Mobile-AMT (EUSIPCO 2024) and consistent-onset streaming piano (arXiv 2503.01362, 2025) — relevant if the app needs to process audio incrementally.
- **Source-separation-then-transcribe pipelines**: Demucs v4 (Hybrid Transformer Demucs) produces high-quality stems; running AMT per stem dramatically improves accuracy on full mixes vs. transcribing the mix directly. This is the most practical path for the Play by Ear Helper.
  - Source: https://github.com/facebookresearch/demucs

### 6. Verdict for "Play by Ear Helper"
- **Solo piano from a clean recording → Sheet music**: Highly feasible. Use hFT-Transformer or a fine-tuned MT3 variant; expect >95% note accuracy and minor user editing.
- **Single-instrument extraction from a pop song (e.g., "show me the bassline")**: Marginally feasible. Pipeline: **Demucs → Basic Pitch / YourMT3+ on the isolated stem**. Expect 60-80% note F1 with errors clustered on fast passages, vibrato, and ornaments. Heavy user-editing UI required.
- **Full-band pop/rock/jazz transcription to multi-staff sheet music**: **Not feasible at production quality in 2026.** Best academic systems hit ~0.83 multi-F1 on synthetic Slakh, far below that on real radio recordings. Output will have wrong instruments, missing notes, and incorrect rhythms — usable only as a rough draft. Set user expectations accordingly or restrict scope.
- **Vocals to melody line**: Possible (Basic Pitch, Klangio's Sing2Notes), but vibrato/glissando still produce noisy MIDI that needs quantization heuristics.

## Sources
- https://archives.ismir.net/ismir2023/paper/000024.pdf (hFT-Transformer, ISMIR 2023)
- https://arxiv.org/pdf/2111.03017 (MT3, 2021)
- https://github.com/magenta/mt3
- https://arxiv.org/abs/2407.04822 (YourMT3+, 2024)
- http://eecs.qmul.ac.uk/~simond/pub/2024/ChangEtAl-MLSP-2024.pdf
- https://arxiv.org/html/2403.10024v1 (MR-MT3)
- https://ai4musicians.org/transcription/2025transcription.html (2025 AMT Challenge)
- https://openreview.net/pdf?id=NG187AZ71W (2025 AMT Challenge results paper)
- https://link.springer.com/article/10.1186/s13636-025-00428-z (Sound/music biases, 2025)
- https://arxiv.org/pdf/2512.14602 (Sound and Music Biases in Deep AMT Models)
- https://github.com/spotify/basic-pitch
- https://engineering.atspotify.com/2022/6/meet-basic-pitch
- https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review
- https://klang.io/
- https://www.songscription.ai/
- https://www.lunaverus.com/ (AnthemScore)
- https://melodyscanner.com/
- https://paperswithcode.com/sota/music-transcription-on-slakh2100
- http://www.slakh.com/
- https://arxiv.org/pdf/2404.09466 (Non-hierarchical Transformer for piano AMT, 2024)
- https://arxiv.org/html/2509.09318 (Sparse-attention piano transcription, 2025)
- https://arxiv.org/pdf/2503.01362 (Streaming piano transcription, 2025)
- https://eurasip.org/Proceedings/Eusipco/Eusipco2024/pdfs/0000036.pdf (Mobile-AMT)
- https://github.com/facebookresearch/demucs (Demucs v4)
- https://www.mdpi.com/2227-7390/13/11/1708 (PF2N multi-instrument, 2025)
- https://arxiv.org/html/2406.15249v1 (ML in AMT systematic survey, 2024)
