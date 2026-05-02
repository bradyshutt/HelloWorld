# Recent Research (2024-2026) Relevant to Jazz Transcription

Survey period: ISMIR 2024, ISMIR 2025, ICASSP 2025, WASPAA 2025, NeurIPS/ICLR 2025, plus arXiv submissions through early 2026. Use case: 15s jazz quartet (trumpet, piano, bass, drums) where the user wants either trumpet single-line notes or piano chord symbols.

## Summary

The state of the art is moving fast on the *adjacent* problems but slow on the **specific** thing we need (real-recording multi-instrument jazz transcription with brass + comping piano). Three pillars stand out:

1. **The OOD problem is now formally measured.** The 2025 "Sound and Music Biases" paper (and the 2025 AMT Challenge) quantify what jazz practitioners have long suspected: piano-trained AMT models lose ~20 F1 points on instrument shift and ~14 on genre shift, and ~52 points on far-OOD audio. This is the first time the gap has been benchmarked across orthogonal axes.
2. **Diffusion / generative-prior AMT is becoming a real category.** D3RM (piano), Noise-to-Notes (drums), and DiffRoll show generative refinement helps with noisy / messy inputs. None target jazz horns yet, but the architecture is portable.
3. **Music-specialised audio LLMs have arrived.** NVIDIA's Music Flamingo (Nov 2025, 7B) and the AMT Challenge 2025 are the first credible signs that LLM-style models can do chord-level harmonic analysis on real audio - the closest match to "give me piano chord symbols for this 15 s jazz clip." Open weights are on Hugging Face.

For our use case, the **single most relevant resource** is Drew Edwards / Simon Dixon / Emmanouil Benetos's work at QMUL: PiJAMA (jazz piano MIDI corpus) plus Huw Cheston's Jazz Trio Database (44.5 h of source-separated piano-trio jazz with onset/beat/MIDI annotation). These two datasets are the only large-scale jazz-specific transcription corpora that exist, and they enable any team that wants to fine-tune for jazz to actually do so. Trumpet/sax remains essentially untouched in the literature.

## Notable Papers

### Sound and Music Biases in Deep Music Transcription Models: A Systematic Analysis (arXiv 2512.14602 / J. Audio Speech Music Proc. 2026)
- https://arxiv.org/abs/2512.14602
- Approach: Decomposes OOD into orthogonal axes (sound, genre, dynamics, polyphony) and introduces the MDS corpus to test piano AMT models on each.
- Results: Note-F1 drops by 20 pp due to instrument/sound shift, 14 pp due to genre shift, ~52 pp on far-OOD ("non-musical") audio. Dynamics estimation is more brittle than onset prediction.
- Code/weights: MDS dataset on Zenodo (10.5281/zenodo.17467279).
- Relevance: HIGH. This is essentially the academic confirmation of why off-the-shelf piano transcribers will fail on jazz quartet audio. Worth citing as the framing for the feasibility study.

### Count The Notes: Histogram-Based Supervision for Automatic Music Transcription (ISMIR 2025, arXiv 2511.14250)
- https://arxiv.org/abs/2511.14250 / https://github.com/Yoni-Yaffe/count-the-notes
- Approach: CountEM - weakly-supervised AMT that only needs note-occurrence histograms (not aligned MIDI) using an EM loop.
- Results: Matches/beats prior weakly-supervised AMT on piano, guitar, multi-instrument; way cheaper labeling.
- Code: Yes (GitHub).
- Relevance: MEDIUM. Could let you bootstrap a jazz-specific model from rough chord-chart-style annotations rather than aligned MIDI - useful if we ever want to fine-tune.

### From Discord to Harmony: Decomposed Consonance-based Training for Improved Audio Chord Estimation (ISMIR 2025, arXiv 2509.01588)
- https://arxiv.org/abs/2509.01588 / https://github.com/andreamust/consonance-ACE
- Approach: Conformer-based ACE that estimates root, bass, and note-set separately; uses a consonance-based label-smoothing loss to handle the inherent subjectivity of chord annotation.
- Results: Improves chord recognition; explicitly evaluates on jazz-style ambiguous chord cases (relative major/minor, inversions).
- Code: Yes.
- Relevance: HIGH. This is the closest 2025 paper to the "give me piano chord symbols" half of our use case, and it directly addresses jazz-style chord ambiguity.

### Mel-RoFormer for Vocal Separation and Vocal Melody Transcription (ISMIR 2024, arXiv 2409.04702)
- https://arxiv.org/abs/2409.04702
- Approach: Spectrogram model with Mel-band Projection + interleaved RoPE Transformers; trained first on separation, then fine-tuned for melody transcription.
- Results: SOTA on vocal separation AND vocal melody transcription; same backbone for both tasks.
- Relevance: MEDIUM. Architecturally interesting - shows that a separation model fine-tuned for transcription is a strong recipe. Could in principle be retrained for trumpet stem - but no public trumpet weights.

### YourMT3+ (MLSP 2024, arXiv 2407.04822)
- https://arxiv.org/abs/2407.04822 / https://github.com/mimbres/YourMT3
- Approach: MT3-style multi-instrument transcription with hierarchical attention transformer + mixture-of-experts encoder + cross-stem augmentation.
- Results: Competitive or SOTA across 10 multi-instrument datasets; supports direct vocal transcription without a separator.
- Code/weights: Yes (GitHub, full reproducibility).
- Relevance: HIGH. This is currently the strongest open-source multi-instrument transcriber. Worth running on a jazz quartet sample as a baseline for the feasibility study - though MT3 itself was the AMT 2025 Challenge baseline that was beaten by only 2 of 8 submissions.

### Advancing Multi-Instrument Music Transcription: Results from the 2025 AMT Challenge (NeurIPS 2025)
- https://openreview.net/forum?id=NG187AZ71W / https://ai4musicians.org/transcription/2025transcription.html
- Approach: Challenge over 76 newly-composed pieces (~20 s each) covering 8 instruments with up to 3 simultaneous instruments, including modern atonal works and rare instruments.
- Results: Only 2 of 8 entries beat MT3 baseline. Best systems still struggle with polyphony + timbre variation.
- Relevance: MEDIUM-HIGH. Confirms that multi-instrument AMT in 2025 is an unsolved problem even on cleanly-synthesized data; jazz quartet is harder.

### D3RM: A Discrete Denoising Diffusion Refinement Model for Piano Transcription (ICASSP 2025, arXiv 2501.05068)
- https://arxiv.org/abs/2501.05068
- Approach: Discrete diffusion model with Neighborhood Attention; refines piano roll predictions from a pretrained acoustic backbone.
- Results: Improvements on MAESTRO; piano-only.
- Relevance: LOW-MEDIUM. Architecture is interesting for future jazz-specific models but the paper itself is piano classical.

### Noise-to-Notes: Diffusion-based Generation and Refinement for Automatic Drum Transcription (arXiv 2509.21739)
- https://arxiv.org/abs/2509.21739
- Approach: Conditional generative diffusion on drum events; uses music foundation model features to be more OOD-robust.
- Results: Better than discriminative baselines on out-of-domain drum audio.
- Relevance: MEDIUM. The drums in our jazz quartet would benefit; and the OOD-robustness story is exactly what we need.

### No Data Required: Zero-Shot Domain Adaptation for Automatic Music Transcription (ICASSP 2025, McLeod)
- https://ieeexplore.ieee.org/iel8/10887540/10887541/10890396.pdf
- Approach: At inference time, transcribe pitch-shifted versions of the input and aggregate where the model is unsure.
- Results: Improves OOD transcription with zero target-domain data.
- Relevance: HIGH (and pragmatic). Could be applied as a wrapper around any baseline transcriber for our use case without retraining.

### Unsupervised Domain Adaptation for Music Transcription: Exploiting Cross-Version Consistency (ICASSP 2025, Liu)
- https://www.researchgate.net/publication/390537940
- Approach: Trains a transcriber to be consistent across multiple versions/recordings of the same piece.
- Relevance: MEDIUM. Less directly applicable to jazz where every performance is different (improvised), but the consistency idea is portable.

### Score-informed Music Source Separation (arXiv 2503.07352)
- https://arxiv.org/abs/2503.07352
- Approach: Concatenate score with audio spectrogram for separation; trained on synthetic SynthSOD (covers brass: horn/trumpet/trombone/tuba), evaluated on URMP and Aalto orchestral data.
- Relevance: LOW for our case (we have no score), but the SynthSOD dataset is relevant if we want brass-aware separation.

### PiJAMA: Piano Jazz with Automatic MIDI Annotations (TISMIR 2024)
- https://transactions.ismir.net/articles/10.5334/tismir.162
- Approach: 200+ hours / 2,777 solo jazz piano performances from 120 pianists, MIDI auto-transcribed via SOTA piano transcribers.
- Code/data: https://almostimplemented.github.io/PiJAMA/ , Zenodo 8354955.
- Relevance: VERY HIGH for the piano-chords half of our use case. First large-scale jazz piano corpus.

### Jazz Trio Database (TISMIR 2024, Cheston)
- https://transactions.ismir.net/articles/10.5334/tismir.186 / https://github.com/HuwCheston/Jazz-Trio-Database
- Approach: 44.5 h of piano-trio jazz, processed through ZFTurbo source separation, then onset / beat / downbeat / MIDI annotation. Onset F-measure 0.94 vs ground truth.
- Relevance: VERY HIGH. The closest existing dataset to our quartet target (we have trumpet additionally; JTD has piano+bass+drums). Could be used directly to fine-tune a jazz-specific transcriber for piano comping.

### Aria-MIDI (ICLR 2025, arXiv 2504.15071)
- https://arxiv.org/abs/2504.15071 / https://huggingface.co/datasets/loubb/aria-midi
- Approach: 1.18M MIDI files (~100k h) auto-transcribed solo piano; for symbolic generative model pretraining.
- Relevance: MEDIUM. Not jazz-specific, but contains substantial jazz subset and is the largest piano MIDI corpus to date - useful as pretraining data for any chord/transcription model.

### VioPTT: Violin Technique-Aware Transcription from Synthetic Data Augmentation (arXiv 2509.23759)
- https://arxiv.org/abs/2509.23759
- Approach: Synthetic violin-with-techniques dataset (MOSA-VPT) plus model that generalizes to real recordings.
- Relevance: LOW directly, MEDIUM as proof-of-concept that an instrument-specific synthetic-to-real pipeline can work for non-piano monophonic instruments. The same recipe could in principle be applied to trumpet.

### MIDI-to-Tab (ISMIR 2024)
- ISMIR 2024 proceedings (Edwards, Riley, Sarmento, Dixon)
- Approach: Masked language modeling for guitar tablature inference from MIDI.
- Relevance: LOW. Tab-only, not relevant to jazz quartet audio in.

### Note-Level Transcription of Choral Music (ISMIR 2024, Yu & Duan)
- ISMIR 2024 proceedings
- Approach: Polyphonic vocal transcription per voice.
- Relevance: LOW directly, but tackles the same "multiple monophonic lines blending" problem we have with horns.

## Audio LLMs

### Music Flamingo (NVIDIA, Nov 2025, arXiv 2511.10289)
- https://arxiv.org/abs/2511.10289 / https://huggingface.co/nvidia/music-flamingo-hf
- 7B Audio-LLM built on Audio Flamingo 3 backbone, with Rotary Time Embeddings for absolute audio token timestamps. Trained with chain-of-thought (MF-Think) + GRPO RL with custom rewards including a "Theory Accuracy Reward" (keys/chords/scales) and "Harmonic Analysis Reward."
- Reviewers note "more faithful chord tracking" and improved localization of chord changes / solos / lyric entrances.
- Handles full-length songs up to 15 minutes.
- SOTA across 10+ music understanding/reasoning benchmarks.
- Relevance: VERY HIGH. This is, today, the closest off-the-shelf system to "feed me 15 s of jazz quartet, give me piano chord symbols." Open weights on HF means we could actually try it.

### Audio Flamingo 3 (NVIDIA, July 2025, arXiv 2507.08128)
- https://arxiv.org/abs/2507.08128 / https://huggingface.co/nvidia/audio-flamingo-3
- General audio LLM backbone for Music Flamingo. Less music-specialised but still competent at music QA.

### Qwen2-Audio-7B-Instruct (Alibaba, 2024)
- https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct
- Audio-LLM with voice chat + audio analysis modes. Music capability is generic - not specialised for transcription.
- Relevance: LOW-MEDIUM. Could be prompted for chord-level descriptions but unlikely to compete with Music Flamingo.

### Evaluating Multimodal LLMs on Core Music Perception Tasks (arXiv 2510.22455)
- https://arxiv.org/abs/2510.22455
- Finding: Models perform near-ceiling on MIDI input but accuracy drops sharply on raw audio - especially on syncopation scoring and chord-quality identification. Gemini Pro is best of the closed models. The bottleneck is specifically transcription/onset/pitch-salience.
- Relevance: HIGH. Confirms that even frontier audio LLMs in late 2025 fail at the specific task we want (chord quality from real audio).

### CMI-Bench (arXiv 2506.12285), MMAR (2505.13032), BASS AI (2602.04085), AHELM (2508.21376)
- Benchmarks showing audio LLMs lag task-specific supervised systems on standard MIR metrics, including chord recognition and lyric transcription. Gemini 2.0 Flash leads MMAR at 65.6%.

## New Startups / Tools

### Songscription (founded 2024, $5M raise Nov 2025)
- https://www.songscription.ai/
- "Shazam for sheet music." Upload audio or YouTube URL, get sheet music + MIDI + MusicXML + Guitar Pro.
- Officially supports trumpet, saxophone, trombone (plus piano, guitar, bass, violin, flute, clarinet, drums, vocals).
- 150k users, 150 countries, 5 months post-launch (per Music Business Worldwide).
- Freemium: 30-second clips free unlimited; $29.99/mo Pro tier.
- **Relevance: VERY HIGH.** This is the most direct commercial competitor for our use case. Worth running our jazz quartet test clip through it to see how it actually performs - claims trumpet support, but quality on real (non-isolated) jazz audio is unknown.

### Klangio Transcription Studio
- https://klang.io/transcription-studio/
- Multi-instrument ecosystem: Piano2Notes, Guitar2Tabs, Scan2Notes, Melody Scanner, Transcription Studio (flagship multi-instrument).
- Marketing claims jazz improvisation support.
- Relevance: HIGH. Established player; good UX baseline.

### Moises (Music AI)
- https://music.ai/
- Mainly stems / source separation, but expanding into MIR. WASPAA 2025 paper "Moises-Light" (resource-efficient band-split U-Net) plus broader 2025 research roadmap.
- Relevance: MEDIUM. The separation step is useful but transcription is not their main product yet.

### Note: no jazz-specific 2025-2026 startup found
- Despite searching, I did not find any 2025/2026 startup specifically targeting jazz or improvised music transcription. The closest is Songscription claiming trumpet/sax support; Klangio claiming jazz improvisation handling. Neither markets a "jazz transcription" product line.

## Verdict

For a 15 s jazz quartet -> trumpet notes OR piano chord symbols, the 2024-2026 research landscape is **mixed encouraging**:

- **For piano chord symbols**: Music Flamingo (Nov 2025) is genuinely the new state of the art for audio-conditioned chord/harmonic analysis on real recordings; combined with the consonance-aware ACE training from Poltronieri et al. (ISMIR 2025), there's a credible path to decent quartet-level chord recognition. PiJAMA + JTD give us jazz-piano-specific data to fine-tune on.
- **For trumpet single-line notes**: This remains the unsolved part. No 2025-2026 paper specifically attacks jazz horn transcription. The realistic recipe is (a) source-separate the trumpet via Mel-RoFormer / Demucs / Moises, (b) run a monophonic pitch tracker (Basic Pitch, CREPE) or a YourMT3+-style multi-instrument model, (c) optionally apply McLeod's zero-shot domain adaptation. None of this is ready off the shelf, but each piece exists.
- **OOD framing is now a recognized research problem**, which means there's tailwind: the 2025 AMT Challenge, the Sound-and-Music-Biases paper, and ICASSP 2025's two domain-adaptation papers all signal that the field knows it has a synthetic-to-real / classical-to-jazz gap and is starting to tackle it.

Bottom line: A bespoke jazz quartet transcriber is buildable in 2026 by combining (i) source separation (Mel-RoFormer or Moises), (ii) per-stem transcription with YourMT3+ for trumpet/bass/piano + a drum model, (iii) Music Flamingo as a high-level chord-symbol layer, (iv) PiJAMA + JTD for fine-tuning. No single off-the-shelf model does it well today, and the closest commercial offering (Songscription) advertises trumpet support but has no published jazz benchmark.

## Sources

- [Sound and Music Biases in Deep Music Transcription Models (arXiv 2512.14602)](https://arxiv.org/abs/2512.14602)
- [Sound and Music Biases - Springer journal version](https://link.springer.com/article/10.1186/s13636-025-00428-z)
- [Count The Notes: Histogram-Based Supervision (ISMIR 2025)](https://arxiv.org/abs/2511.14250)
- [Count The Notes - GitHub](https://github.com/Yoni-Yaffe/count-the-notes)
- [From Discord to Harmony: Consonance-based ACE (ISMIR 2025)](https://arxiv.org/abs/2509.01588)
- [Consonance-ACE GitHub](https://github.com/andreamust/consonance-ACE)
- [Mel-RoFormer (ISMIR 2024)](https://arxiv.org/abs/2409.04702)
- [YourMT3+ (MLSP 2024)](https://arxiv.org/abs/2407.04822)
- [YourMT3+ GitHub](https://github.com/mimbres/YourMT3)
- [Advancing Multi-Instrument Music Transcription: AMT Challenge 2025 results](https://openreview.net/forum?id=NG187AZ71W)
- [2025 AMT Challenge homepage](https://ai4musicians.org/transcription/2025transcription.html)
- [D3RM: Discrete Diffusion Refinement for Piano Transcription (ICASSP 2025)](https://arxiv.org/abs/2501.05068)
- [Noise-to-Notes diffusion drum transcription (arXiv 2509.21739)](https://arxiv.org/abs/2509.21739)
- [No Data Required: Zero-Shot Domain Adaptation for AMT (ICASSP 2025)](https://ieeexplore.ieee.org/iel8/10887540/10887541/10890396.pdf)
- [Unsupervised Domain Adaptation via Cross-Version Consistency (ICASSP 2025)](https://www.researchgate.net/publication/390537940_Unsupervised_Domain_Adaptation_for_Music_Transcription_Exploiting_Cross-Version_Consistency)
- [Score-informed Music Source Separation (arXiv 2503.07352)](https://arxiv.org/abs/2503.07352)
- [SynthSOD orchestral synthetic dataset (arXiv 2409.10995)](https://arxiv.org/pdf/2409.10995)
- [PiJAMA: Piano Jazz with Automatic MIDI Annotations (TISMIR 2024)](https://transactions.ismir.net/articles/10.5334/tismir.162)
- [Jazz Trio Database (TISMIR 2024)](https://transactions.ismir.net/articles/10.5334/tismir.186)
- [Jazz Trio Database GitHub](https://github.com/HuwCheston/Jazz-Trio-Database)
- [Aria-MIDI dataset (ICLR 2025)](https://arxiv.org/abs/2504.15071)
- [Aria-MIDI HuggingFace](https://huggingface.co/datasets/loubb/aria-midi)
- [VioPTT: Violin Technique-Aware Transcription (arXiv 2509.23759)](https://arxiv.org/abs/2509.23759)
- [Music Flamingo (NVIDIA, arXiv 2511.10289)](https://arxiv.org/abs/2511.10289)
- [Music Flamingo project page](https://research.nvidia.com/labs/adlr/MF/)
- [Music Flamingo HuggingFace weights](https://huggingface.co/nvidia/music-flamingo-hf)
- [Audio Flamingo 3 (arXiv 2507.08128)](https://arxiv.org/abs/2507.08128)
- [Qwen2-Audio-7B-Instruct](https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct)
- [Evaluating MLLMs on Core Music Perception (arXiv 2510.22455)](https://arxiv.org/pdf/2510.22455)
- [CMI-Bench (arXiv 2506.12285)](https://arxiv.org/html/2506.12285v1)
- [MMAR benchmark (arXiv 2505.13032)](https://arxiv.org/html/2505.13032v1)
- [MuQ self-supervised music representation (arXiv 2501.01108)](https://arxiv.org/pdf/2501.01108)
- [MuFun: Advancing the Foundation Model for Music Understanding (arXiv 2508.01178)](https://arxiv.org/abs/2508.01178)
- [Songscription homepage](https://www.songscription.ai/)
- [Songscription $5M raise (Music Business Worldwide)](https://www.musicbusinessworldwide.com/songscription-raises-5m-in-funding-as-shazam-for-sheet-music-platform-reaches-150k-users/)
- [Songscription Music Ally coverage](https://musically.com/2025/07/01/songscription-uses-ai-to-automate-sheet-music-transcription/)
- [Klangio Transcription Studio](https://klang.io/transcription-studio/)
- [Moises Research 2025](https://music.ai/blog/research/Moises-Research-Innovations-2025/)
- [WASPAA 2025 Technical Program](https://waspaa.com/technical-program-schedule/)
- [ISMIR 2025 program](https://ismir2025program.ismir.net/)
- [ISMIR 2024 conference](https://ismir2024.ismir.net/)
- [Spotify Basic Pitch GitHub](https://github.com/spotify/basic-pitch)
