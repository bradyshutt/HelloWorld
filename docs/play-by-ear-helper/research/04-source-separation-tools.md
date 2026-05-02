# Audio Source Separation Tools

## Summary
Source separation in 2025-2026 is excellent for the canonical 4 stems (vocals/drums/bass/other) — leading models reach ~9.0-9.9 dB SDR on MUSDB18-HQ, which is perceptually clean enough for downstream transcription. Separation quality drops significantly for "other" instruments such as piano, guitar, strings, and synths; only a few systems (Demucs `htdemucs_6s`, LALAL.AI Perseus, Moises Hi-Fi, AudioShake, MVSep specialized models) attempt these, and quality is uneven (Demucs explicitly notes piano "is not working great"). Recommended pipeline for Play by Ear Helper: use **Demucs v4 (htdemucs_ft)** as the open-source default for vocals/drums/bass/other, fall back to **htdemucs_6s** when the user picks guitar/piano, and consider a commercial fallback (**LALAL.AI** or **AudioShake**) or specialized **MVSep / BS-RoFormer / Mel-RoFormer** checkpoints when accuracy matters. Plan for ~1-2x realtime CPU inference and ~30x realtime on a modern GPU/Apple Silicon.

## Tools

### Demucs v4 (Hybrid Transformer Demucs)
- Type: open-source (Meta / facebookresearch, now mirrored at github.com/adefossez/demucs after Jan 2025 archive)
- Stems: 4-stem (`htdemucs`, `htdemucs_ft`) = vocals/drums/bass/other; 6-stem (`htdemucs_6s`) adds guitar + piano
- Accuracy (SDR on MUSDB18-HQ): 9.00 dB (htdemucs), 9.20 dB (fine-tuned `htdemucs_ft` with sparse attention) — state-of-the-art among generally available open models
- Hardware: ~3 GB VRAM minimum, 7 GB recommended; CPU runs at ~1.5x realtime; RTX 3090 with TensorRT FP16 ~30x faster; Apple Silicon (M4 Max) ~34x realtime (7-min track in ~12 sec)
- License: MIT (commercial use OK)
- Notes: Default and most-cited choice. `htdemucs_ft` is 4x slower at inference but slightly higher quality. `htdemucs_6s` works well for guitar but the docs explicitly warn piano quality is poor. Easy Python integration: `import demucs.separate; demucs.separate.main([...])`.

### Spleeter (Deezer)
- Type: open-source
- Stems: 2-stem (vocals/accompaniment), 4-stem (v/d/b/other), 5-stem (adds piano)
- Accuracy (SDR): roughly 3 dB below Demucs v4; "almost on par with Demucs" in 2019 but no longer competitive
- Hardware: Very fast — 100x realtime on GPU; ~2 sec for a 3-min track; runs comfortably on CPU
- License: MIT
- Notes: TensorFlow-based. Last meaningful update 2019; Deezer has declared it "feature complete." Spectrogram-only (no waveform reconstruction), which causes ringing/metallic artifacts and bleed (notably kick drum into bass stem). Still useful when speed matters more than fidelity.

### Open-Unmix (UMX / UMXHQ / UMXL)
- Type: open-source (sigsep)
- Stems: 4-stem (vocals/drums/bass/other)
- Accuracy (SDR): around SiSEC 2018 SOTA; reference-grade but eclipsed by Demucs/RoFormer family
- Hardware: Lightweight LSTM; runs on CPU; C++ port (umx.cpp) and NNabla port available
- License: MIT for code; **UMXL pretrained weights are CC BY-NC-SA 4.0 (non-commercial only)** — important constraint
- Notes: Designed as a clean reference implementation for research. Good for embedded/edge or as a baseline; not the right choice for production quality.

### MDX-Net (KUIELab-MDX-Net) and the MDX family
- Type: open-source
- Stems: 4-stem (vocals/drums/bass/other); various community fine-tunes for instruments
- Accuracy (SDR): 2nd/3rd in MDX 2021 challenge; SDX'23 winners exceeded prior winners by >1.6 dB SDR. Demucs ships `mdx` and `mdx_extra` checkpoints derived from this lineage.
- Hardware: Comparable to Demucs; two-stream (time-frequency + time-domain) architecture
- License: typically MIT
- Notes: Strong on vocal isolation specifically. The MDX23/MDX23C architecture underpins many community-trained instrument-specific models on MVSep.

### BS-RoFormer / Mel-RoFormer (ByteDance)
- Type: open-source implementations (lucidrains/BS-RoFormer, ZFTurbo training repo)
- Stems: trainable per-stem; community 6-stem checkpoints exist
- Accuracy (SDR on MUSDB18-HQ): **9.92 dB BS-RoFormer**, **9.64 dB Mel-RoFormer** (vocals 11.21 dB) — currently SOTA. Mel-RoFormer vocal-extraction checkpoint hits 11.28 dB vocals / 17.59 dB instrumental; "Best SDR" version reaches 11.93 dB on Multisong.
- Hardware: Heavier than Demucs; benefits significantly from GPU
- License: implementation MIT; pretrained weights vary
- Notes: Currently the leading architecture per MVSep leaderboards as of early 2026. Pre-trained weights are scattered across community releases — less plug-and-play than Demucs.

### SCNet (Sparse Compression Network)
- Type: open-source (starrytong/SCNet, amanteur PyTorch port)
- Stems: 4-stem (extendable)
- Accuracy (SDR on MUSDB18-HQ): 9.0 dB **without extra training data** — strong drums/bass numbers
- Hardware: 10.08M params (~25% of HT Demucs); CPU inference ~48% of HT Demucs time
- License: open (check repo)
- Notes: Best efficiency-vs-quality trade-off in 2024. Good candidate when running on commodity hardware. Used as a backbone in some MVSep specialized piano/guitar models.

### Bandit / Bandit-v2
- Type: open-source (kwatcharasupat)
- Stems: 3-stem dialogue/music/effects (cinematic), with v2 adding a 4-stem variant that splits singing voice from instrumental
- Accuracy: SOTA on Divide-and-Remaster (DnR v3); above the ideal-ratio-mask oracle for dialogue
- Hardware: GPU-friendly band-split RNN
- License: open
- Notes: **Wrong target domain for Play by Ear Helper.** Bandit is built for film/TV (dialogue vs music vs SFX), not instrument-level music separation. Skip unless transcribing audio from video.

### LALAL.AI (commercial)
- Type: commercial SaaS / API
- Stems: up to 10 — Vocal, Instrumental, Drums, Bass, Electric Guitar, Acoustic Guitar, Piano, Synthesizer, Strings, Wind Instruments, Voice/Noise
- Accuracy: among the best for piano and guitar specifically; one of only two tools that successfully extracted piano in MusicTech's 2025 review. Uses in-house "Perseus" transformer (Feb 2025 expanded to acoustic guitar, electric guitar, piano)
- Hardware: cloud
- License: pay-per-minute or subscription; free tier 10 min preview; ~€6.75/mo (annual) or €8.99/mo (monthly)
- Notes: Strongest commercial option for the multi-instrument case. API available. Per-minute cost scales with stem count and quality.

### AudioShake (commercial)
- Type: commercial SaaS / API / SDK (real-time SDK available)
- Stems: vocals, drums, bass, guitar, piano, strings, "and more"
- Accuracy: production-grade; used by Disney, Netflix, Oreo, Taco Bell for licensing/dubbing pipelines
- Hardware: cloud; SDK supports on-device/self-hosted enterprise deploys
- License: enterprise commercial; pricing on request
- Notes: Strongest enterprise option, especially for real-time or licensing-sensitive use cases. Likely overkill (and over-priced) for a hobbyist Play by Ear Helper but worth evaluating if commercializing.

### Moises.ai (commercial)
- Type: commercial app + API
- Stems: vocals, drums (kick/snare separable), bass, guitar (lead/rhythm, electric/acoustic separable in Premium), piano, strings, others
- Accuracy: Hi-Fi separation model (2025) is their best generation; major guitar-separation upgrade in 2025
- Hardware: cloud + mobile/VST plugin
- License: subscription (free tier with limits, Premium ~$3.99-7.99/mo equivalents)
- Notes: Aimed at musicians/practice; strong UX. Closer to LALAL.AI in accuracy, with better mobile/DAW integration.

### MVSep (platform + leaderboards)
- Type: hybrid — runs many open-source models in cloud; publishes public leaderboards
- Stems: piano, guitar, vocals, drums, bass, plus combos via specialized models (e.g., MVSep Piano = MDX23C + Mel-RoFormer + SCNet Large ensemble; MVSep Guitar = MDX23C + Mel-RoFormer + BS-RoFormer)
- Accuracy: leaderboards at mvsep.com/quality_checker for piano, multisong, etc. Ensemble models for piano/guitar typically beat any single model
- Hardware: cloud (paid credits) or self-host the underlying open models
- License: per-use credits; underlying models' licenses vary
- Notes: Best place to track which checkpoint is currently SOTA per stem. Useful as a benchmark reference even if you self-host.

## Recommendation

**Pipeline for Play by Ear Helper:**

1. **Default open-source path:** Run Demucs v4 (`htdemucs_ft` for the 4 canonical stems; `htdemucs_6s` if user requests guitar). MIT license, easy Python API, runs on CPU at ~1.5x realtime, much faster on GPU/Apple Silicon. Covers ~80% of pop/rock requests.
2. **For piano:** `htdemucs_6s` is the easy default but quality is mediocre. For better results, route piano requests through an MVSep-style ensemble (MDX23C + Mel-RoFormer + SCNet Large) or call **LALAL.AI** API as a paid fallback.
3. **For "other" instruments (synth, strings, wind):** Demucs lumps these into the "other" stem. The user will need to accept that polyphonic transcription happens on a still-mixed-down "other" stem, OR use **LALAL.AI** (10 stems) / **AudioShake** for proper isolation.
4. **Optional SOTA upgrade:** Swap Demucs for **BS-RoFormer / Mel-RoFormer** community checkpoints (ZFTurbo's training repo) for ~0.5-0.7 dB SDR improvement, at the cost of more setup friction.
5. **Avoid for this use case:** Spleeter (outdated, artifacts), Open-Unmix UMXL (non-commercial license), Bandit (cinematic domain, not instruments).

**Realistic expectations:** Separating a clean vocal or bass line from a typical pop mix is now near-perfect. Separating a piano line is feasible but artifact-prone — expect bleed from other harmonic instruments. Separating a single guitar from a dense rock mix with multiple guitars is the hardest case and may require LALAL.AI's lead/rhythm or electric/acoustic split.

## Sources

- [Demucs GitHub (facebookresearch)](https://github.com/facebookresearch/demucs)
- [Hybrid Transformers for Music Source Separation (arXiv 2211.08553)](https://arxiv.org/pdf/2211.08553v1)
- [Spleeter GitHub (deezer)](https://github.com/deezer/spleeter/)
- [Spleeter JOSS paper](https://www.theoj.org/joss-papers/joss.02154/10.21105.joss.02154.pdf)
- [Spleeter vs Demucs comparison (StemSplit, 2026)](https://stemsplit.io/blog/spleeter-vs-demucs)
- [Open-Unmix GitHub](https://github.com/sigsep/open-unmix-pytorch)
- [Open-Unmix project page](https://sigsep.github.io/open-unmix/)
- [KUIELab-MDX-Net paper (arXiv 2111.12203)](https://arxiv.org/abs/2111.12203)
- [Sound Demixing Challenge 2023 (TISMIR)](https://transactions.ismir.net/articles/10.5334/tismir.171)
- [SCNet paper (arXiv 2401.13276)](https://arxiv.org/abs/2401.13276)
- [SCNet GitHub](https://github.com/starrytong/SCNet)
- [BS-RoFormer GitHub (lucidrains)](https://github.com/lucidrains/BS-RoFormer)
- [Mel-Band RoFormer paper (arXiv 2310.01809)](https://arxiv.org/abs/2310.01809)
- [Music Source Separation Training repo (ZFTurbo)](https://github.com/ZFTurbo/Music-Source-Separation-Training)
- [Bandit-v2 GitHub](https://github.com/kwatcharasupat/bandit-v2)
- [Facing the Music: Singing Voice in Cinematic Audio Separation (arXiv 2408.03588)](https://arxiv.org/abs/2408.03588)
- [LALAL.AI pricing](https://www.lalal.ai/pricing/)
- [LALAL.AI 2025 Wrapped (Perseus stems)](https://www.lalal.ai/blog/lalalai-wrapped-2025/)
- [AudioShake developer docs](https://developer.audioshake.ai/)
- [AudioShake instrument stem separation](https://www.audioshake.ai/instrument-stem-separation)
- [Moises.ai homepage](https://moises.ai/)
- [Moises October 2025 releases](https://moises.ai/blog/moises-news/improvements-latest-releases/)
- [MVSep algorithms catalog](https://mvsep.com/en/algorithms)
- [MVSep Piano leaderboard](https://mvsep.com/quality_checker/leaderboard/piano)
- [MVSep Multisong leaderboard](https://mvsep.com/quality_checker/multisong_leaderboard)
- [MusicTech: 9 best stem separation tools (2025)](https://musictech.com/guides/buyers-guide/best-stem-separation-tools/)
- [Source Separation & Automatic Transcription (arXiv 2412.06703)](https://arxiv.org/abs/2412.06703)
- [Music Source Separation (Wikipedia)](https://en.wikipedia.org/wiki/Music_Source_Separation)
- [Demucs Apple Silicon port (Medium)](https://medium.com/@andradeolivier/i-ported-demucs-to-apple-silicon-it-separates-a-7-minute-song-in-12-seconds-6c4e5cffb5c3)
- [Benchmarks and leaderboards for sound demixing tasks (arXiv 2305.07489)](https://arxiv.org/pdf/2305.07489)
