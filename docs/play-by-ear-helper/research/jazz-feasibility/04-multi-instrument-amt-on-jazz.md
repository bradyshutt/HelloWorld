# Multi-Instrument AMT on Jazz

## Summary

The state of the art for general multi-instrument AMT in 2024-2025 is **YourMT3+** (MLSP 2024), a successor to Google's **MT3** (ICLR 2022) and a contemporary of **PerceiverTF** and **MR-MT3**. These models are trained on a near-identical mix of datasets dominated by Slakh2100 (synthesized General-MIDI renders of pop/rock MIDI files), MAESTRO (classical piano), URMP (small classical chamber, 13 orchestral instruments including trumpet), GuitarSet, and MusicNet. None were trained on real jazz recordings as a major source. They emit MT3's instrument vocabulary (MIDI program / 16 MIDI classes), so a "trumpet" track is in principle outputtable -- but reported per-instrument F1 on real-world brass is essentially absent from the literature, and the sole real-recording brass benchmark (URMP trumpet) is single-line classical, not jazz-mix conditions. **Realistic expectation: 50-65% multi-F1 (MIDI-class) on a clean jazz quartet, with severe instrument leakage between trumpet/sax/trombone, and no published jazz number to anchor the estimate.** Biggest gap: there is no real-jazz multi-instrument benchmark, no jazz training corpus mixing brass/piano/bass/drums with note-level labels, and the timbral confusion problem is openly acknowledged as unsolved (it's the entire motivation for MR-MT3).

## Models

### MT3 (Google Magenta, ICLR 2022)
- **Architecture**: T5-style encoder-decoder transformer, mel-spectrogram in, MIDI-event token sequence out.
- **Training datasets**: MAESTRO, MusicNet, Slakh2100, Cerberus4, GuitarSet, URMP -- jointly mixed.
- **Instruments labeled**: Full MIDI program (128) or 16-class MIDI grouping. Trumpet exists as a distinct label (program 56 / "Brass" class). Slakh2100's 14 instrument categories include trumpet, horn, trombone individually; URMP also includes trumpet in real recordings.
- **Headline F1**: On Slakh2100, multi-instrument Onset+Offset+Program F1 ~0.48 (Flat), ~0.62 (MIDI-class), ~0.55 (Full). On URMP (real chamber recordings) it's lower but the paper does not break out "trumpet F1" vs "violin F1".
- **Jazz**: No jazz dataset in training, no jazz benchmark in evaluation. Magenta's own demo page warns "multi-instrument transcription is still not a completely-solved problem" and "neither model is trained on singing." Anecdotal community use on jazz mixes is reported as poor.
- **Real recordings**: URMP gives a real-acoustic generalisation read but it's classical chamber music with clean isolated stems; Slakh is fully synthesized.

### YourMT3+ (Chang et al., QMUL, MLSP 2024)
- **Architecture**: MT3-style encoder-decoder with two upgrades: a hierarchical time-frequency Perceiver-TF encoder and a Mixture-of-Experts (MoE) decoder. Adds multi-channel decoding so it can train on stems with incomplete annotations, plus intra/cross-stem augmentation.
- **Training datasets**: Same backbone mix as MT3 (Slakh2100, MAESTRO, MusicNet, URMP, GuitarSet, ENST-Drums) plus vocal data (MIR-1K, CMedia, JVS). Benchmarked on 10 public datasets.
- **Instruments labeled**: Same MT3 vocabulary -- piano, guitar, bass, brass-trumpet, brass-trombone, brass-horn, sax, strings, drums, plus direct vocals (no separator needed).
- **Headline F1**: "YPTF.MoE+ Multi" reportedly outperforms MT3 and PerceiverTF on Slakh and URMP multi-instrument F1; numerical headline gains of a few F1 points over MT3. No public per-instrument F1 for trumpet specifically.
- **Jazz**: No jazz dataset in training or evaluation.
- **Real recordings**: Best public option for "real audio" generalization given URMP/MusicNet/MedleyDB-style augmentation, but again, no jazz numbers reported.

### MR-MT3 (Tan et al., 2024)
- **Architecture**: MT3 plus a memory-retention mechanism, prior-token sampling and token shuffling -- specifically designed to fix MT3's **instrument leakage** (a single melody line getting split across trumpet/sax/clarinet labels segment by segment).
- **Training datasets**: Slakh2100, ComMU, NSynth (eval).
- **Headline F1**: Multi-instrument MIDI-class F1 on Slakh: 61.6% (MT3 baseline) -> 66.4% (MR-MT3). Instrument-leakage ratio: 1.65 -> 1.05 (lower is better).
- **Trumpet/jazz**: Same MT3 vocabulary, no jazz eval. MR-MT3's existence is itself the strongest evidence that timbral confusion is real and unfixed by MT3 -- exactly the trumpet-vs-trombone-vs-sax problem the user is asking about.

### PerceiverTF (Lu et al., ICASSP 2023)
- **Architecture**: Hierarchical Perceiver over time-frequency input plus a temporal Transformer.
- **Instruments**: 12 instrument classes + vocal, multi-task. Trumpet/brass is one of the classes.
- **Training**: Slakh2100 + augmentation. No jazz.
- **Headline**: Beat MT3 and SpecTNT on Slakh2100 multi-instrument Onset F1; specifically improved on **less-common instruments** via random-mixing augmentation -- which is a tacit admission that brass etc. are tail classes.

### Pop2Piano (Choi et al., ICASSP 2023)
- **Goal**: Pop audio -> piano-cover MIDI. NOT multi-instrument transcription -- it collapses everything to piano. Quantizes to 8th notes (no triplets/swing). Useless for "give me the trumpet line" but possibly useful for "give me the piano voicing" if the input is solo-piano-friendly. Trained on paired pop-audio + human piano-cover MIDI, not jazz.

### Sheet Sage (Donahue et al., 2022)
- **Goal**: Pop audio -> lead sheet (single melody + chord symbols). Uses Jukebox features + transformers, trained on HookTheory's TheoryTab DB (~50 h pop annotations).
- **Output**: Melody + chord names, not per-instrument tracks. Could plausibly extract a "lead line" + "chords" view of a jazz quartet but its training distribution is pop, and the chord vocabulary is pop-style (won't render altered/extension jazz voicings well).

### MIDI-DDSP (Wu et al., ICLR 2022)
- **Direction**: This is the **inverse** -- MIDI -> realistic audio synthesis, not audio -> MIDI. Trained on URMP, can synthesize 13 orchestral instruments including trumpet. Some inverse-rendering AMT work uses it as a differentiable forward model, but no production-grade AMT system uses MIDI-DDSP as the transcriber. Listed because it's referenced in the question but it doesn't transcribe.

### Onsets and Frames (and "Plus" variants) / Basic Pitch
- **Onsets and Frames**: Piano-only, MAESTRO-trained. Out of scope.
- **Spotify Basic Pitch**: Lightweight, polyphonic, generalizes to several instruments and vocals -- but the docs explicitly say "best on one instrument at a time." Not a multi-instrument labeller; you'd run it post-source-separation.

### Jointist (Cheuk et al., 2022)
- Multi-instrument transcription with explicit instrument-detection head feeding a transcriber. Slakh-trained. Same data-distribution problem.

### NoteDance / 2024-2026 transformers
- No widely benchmarked "NoteDance" AMT model surfaced in the search. The 2025 AMT Challenge (OpenReview NG187AZ71W) is the most recent organized benchmark: 8 teams, 2 beat MT3 baseline. Submissions extend MT3 with self-supervised random-projection quantizers, MoE routing, hierarchical TF attention, cross-dataset augmentation, auxiliary onset/offset losses. No jazz track in the challenge.

## Benchmarks for Jazz

| Benchmark | Domain | Has trumpet? | Multi-instrument? | Used for AMT eval? |
|---|---|---|---|---|
| **MAPS** | Solo piano, real | No | No | Piano AMT only |
| **MAESTRO** | Solo piano, real (Disklavier) | No | No | Piano AMT only |
| **Slakh2100** | Pop/rock MIDI, **synthesized** | Yes (as GM trumpet patch) | Yes (up to ~10 instruments) | Default multi-AMT bench, but synth-only |
| **URMP** | Classical chamber, **real** | Yes (single-line) | Yes (2-5 players) | Real-audio sanity check; small (~1.3 h) |
| **MusicNet** | Classical orchestra, real | Yes (orchestral) | Yes | Real but noisy alignment |
| **MedleyDB** | Mixed genres incl. some jazz, real multitrack | Sometimes | Yes | Used for source-sep / melody, not directly for multi-AMT |
| **Cerberus** | Slakh-derived 4-instrument mixes | Yes (synth) | Yes (2-5) | Joint sep+transcription |
| **JSB Chorales** | Bach 4-part, MIDI | No | Voices only | Symbolic gen, not AMT |
| **Filosax** | Jazz **alto sax** + backing tracks, real | No (sax, not trumpet) | Solo + backing | Jazz-specific, but sax only |
| **FiloBass** | Jazz **double bass**, real | No | Bass only | Jazz-specific, bass only |
| **PiJAMA** | Solo jazz **piano**, real | No | No | 200 h of jazz piano with auto-MIDI labels |
| **Weimar Jazz Database (WJazzD)** | ~456 monophonic jazz solos, real | Yes (mixed instruments incl. trumpet) | No (solo lines only) | Jazz solo MIR research, monophonic |

Bottom line: **there is no real-jazz multi-instrument note-level benchmark.** The closest you get is stitching together Filosax (sax) + FiloBass (bass) + PiJAMA (piano) + WJazzD (solos) -- all monophonic-or-solo jazz datasets, none of which exercise a four-piece mix with brass, piano, bass, and drums together with stem-level note labels.

## Out-of-Distribution Behavior on Real Recordings

- **Synth-to-real gap is acknowledged.** Multiple papers note that MT3 trained on Slakh "overfits to audio mixtures" and degrades on real audio. A 2024 paper ("Analyzing and reducing the synthetic-to-real transfer gap... drum transcription," arXiv 2407.19823) quantifies this for drums and proposes mitigations. The synthetic distribution has artificial tempo peaks (90, 120 bpm) and lacks room/microphone variability -- both features of real jazz recordings.
- **Instrument leakage is the core failure mode.** MR-MT3's authors describe MT3 transcribing the same melody line as trumpet in segment 1, sax in segment 2, clarinet in segment 3 -- because each ~5-second window is decoded independently with no memory. On a real jazz quartet, a trumpet solo over 16 bars would routinely fragment across multiple brass/wind labels.
- **No formal "real jazz mix" evaluation.** Searched ICASSP/ISMIR 2022-2025 multi-instrument AMT papers; none report jazz-mix results. The 2025 AMT Challenge does not include jazz audio either.
- **Anecdotal user reports**: GitHub/forum discussions of MT3 on jazz indicate reasonable piano transcription, frequent confusion of trumpet with sax/trombone, dropped notes on fast bebop runs, drums often labeled adequately as a kit (since drum tokens are a separate class) but cymbal/tom-specific accuracy is poor. No quantitative published numbers.
- **Timbral confusion problem**: Confirmed real for trumpet vs. trombone vs. sax vs. clarinet because (a) Slakh's GM brass patches are smoother than real brass timbre, (b) the orchestral overlap of brass/wind formants is exactly where Slakh's training distribution is thinnest, and (c) the per-instrument F1 columns published by these papers are typically dominated by piano/guitar/bass/drums (the well-resourced classes); brass/wind/strings often appear under "other" or "Strings" aggregates.

## Verdict

**Is there a single model today that can take a 15 s jazz-quartet mix and reliably output `{trumpet: notes, piano: chords, bass: notes, drums: rhythms}`?**

- **In principle**: YourMT3+ or MT3 will produce something in that shape (it has trumpet, piano, bass, drums tokens in its vocabulary).
- **In practice**: Drums will be the most reliable (~70-80% drum F1 on Slakh transfers reasonably well). Bass single-line will be ok (~60% MIDI-class F1 plausible, monophonic helps). Piano chord/voicing will be partially right but with missing inner voices and timing errors -- treat as "chord symbol guess" not "exact voicing." **Trumpet will be the weakest link**: expect leakage to sax/trombone, missed bends/scoops, and dropped notes on fast lines. There is no published jazz-mix F1, but a 40-55% trumpet-line note-onset F1 on a real bebop clip is a reasonable rough expectation given Slakh-trained brass behavior.
- **Recommended stack if building today**: YourMT3+ as the primary multi-instrument transcriber (best published OOD generalization, vocal head as a bonus); fall back to source-separation (Demucs/HT-Demucs / MDX-Net) followed by per-instrument transcribers (Basic Pitch for monophonic stems, dedicated piano AMT like high-resolution Onsets-and-Frames or hFT-Transformer for the piano stem). For chord output specifically, run a chord-recognition model (BTC, ChordNet, or a recent transformer chord-recognizer trained on jazz) on the piano stem rather than rely on note-level voicing transcription.
- **Active work but no shipped solution**: PerceiverTF's random-mix augmentation, MR-MT3's memory mechanism, and YourMT3+'s MoE+stem-augmentation all attack pieces of the timbral-confusion / leakage problem. None are evaluated on jazz mixes. There is no public ongoing project that targets jazz-quartet multi-AMT specifically.

## Sources

- MT3 paper (ICLR 2022): https://openreview.net/pdf?id=iMSjopcOn0p and https://arxiv.org/abs/2111.03017
- MT3 GitHub: https://github.com/magenta/mt3
- MT3 blog/demo: https://magenta.tensorflow.org/transcription-with-transformers
- YourMT3+ (MLSP 2024): https://arxiv.org/abs/2407.04822 and http://eecs.qmul.ac.uk/~simond/pub/2024/ChangEtAl-MLSP-2024.pdf
- YourMT3+ GitHub: https://github.com/mimbres/YourMT3
- YourMT3+ literature review: https://www.themoonlight.io/en/review/yourmt3-multi-instrument-music-transcription-with-enhanced-transformer-architectures-and-cross-dataset-stem-augmentation
- MR-MT3: https://arxiv.org/abs/2403.10024 and https://arxiv.org/html/2403.10024v1
- MR-MT3 GitHub: https://github.com/gudgud96/MR-MT3
- PerceiverTF: https://arxiv.org/html/2306.10785 and https://ieeexplore.ieee.org/document/10096688/
- 2025 AMT Challenge: https://openreview.net/forum?id=NG187AZ71W
- Pop2Piano: https://www.semanticscholar.org/paper/Pop2Piano-:-Pop-Audio-Based-Piano-Cover-Generation-Choi-Lee/b2cd9cbc9521c398d003e367c2311ad5796e232f
- Sheet Sage: https://github.com/chrisdonahue/sheetsage and https://arxiv.org/pdf/2212.01884
- MIDI-DDSP: https://midi-ddsp.github.io/ and https://magenta.tensorflow.org/midi-ddsp
- Cerberus: https://interactiveaudiolab.github.io/project/cerberus.html and https://arxiv.org/abs/1910.12621
- Slakh2100 dataset: http://www.slakh.com/
- MedleyDB: https://medleydb.weebly.com/
- Filosax (jazz sax): https://www.eecs.qmul.ac.uk/~simond/pub/2021/FosterDixon-Filosax-ISMIR2021.pdf and https://zenodo.org/records/5625643
- FiloBass (jazz bass): https://arxiv.org/abs/2311.02023
- PiJAMA (jazz piano): https://transactions.ismir.net/articles/10.5334/tismir.162
- Weimar Jazz Database: https://jazzomat.hfm-weimar.de/dbformat/dboverview.html
- Synth-to-real gap (drums case study): https://arxiv.org/html/2407.19823v1
- Jointist: https://arxiv.org/pdf/2302.00286
- Slakh2100 SOTA leaderboard: https://paperswithcode.com/sota/music-transcription-on-slakh2100
- AMT survey 2024: https://arxiv.org/html/2406.15249
- Community AMT notes (devalias gist): https://gist.github.com/0xdevalias/f2c6e52824b3bbd4fb4c84c603a3f4bd
