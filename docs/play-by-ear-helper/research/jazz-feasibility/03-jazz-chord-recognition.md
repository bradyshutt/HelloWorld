# Jazz Chord Recognition

## Summary

For a 15-second clip of a real jazz quartet, **2026-era ACR is fundamentally a triad / basic-7th technology, not a jazz-voicing technology**. State of the art on pop benchmarks (Isophonics, McGill Billboard) reaches **~83-87% MajMin accuracy** and ~75-85% on a "sevenths" vocabulary (maj/min/maj7/min7/dom7). On the only purpose-built jazz benchmark (JAAH, 113 tracks with maj/min/dom7/hdim7/dim labels), reported baseline accuracy is **~42%** and even modern systems materially underperform their pop numbers. ChordFormer (ISMIR-adjacent, 2025) is the current open SotA at **84.7% Root / 84.1% MajMin / 83.6% MIREX** on large-vocab pop, but no public model is trained or evaluated specifically on extended/altered jazz piano voicings (b9, #9, #11, 13, alt, sus4(b7), polychords, quartal). The biggest gap: **upper-structure tones (9/11/13) and altered tensions (b9/#9/#11/b13) are systematically discarded or collapsed to the underlying 7th**, which is exactly the information a jazz user cares about. Realistic expectation for a jazz quartet recording in 2026: **~75-85% on root, ~60-75% on triad+dom7 quality, <40% on the full extended/altered chord symbol** the pianist actually played.

The most promising direction for a jazz-grade product is **hybrid**: piano-stem source separation (Demucs htdemucs_6s) + high-resolution piano transcription (Bytedance/MAESTRO models, >90% F1 on note onsets) + rule-based / LLM-based chord-symbol inference from the detected pitch set. No off-the-shelf product does this end-to-end well today; commercial leader Chord AI lists 13b9 / 9sus4 / min7add13 in its Pro vocabulary but reviewers explicitly note "accuracy drops on complex jazz." Chordify drops to <40% on unaccompanied jazz guitar and misses Cmaj9#11 entirely.

## Models and Their Vocabularies

### Chordino / NNLS Chroma (Mauch & Dixon, Vamp plugin)
- Wraps NNLS chroma + HMM. The python wrapper (`ohollo/chord-extractor`) exposes labels like `Emaj7`, `Am7b5`, `Cb7`, `N` (no chord). Vocabulary is roughly **maj, min, maj7, min7, dom7, dim, m7b5, sus** — i.e. a "sevenths" vocab, no 9/11/13/alt.
- Known failure mode: "assumes any tones higher than the 7th are melody"; rootless voicings collapse to a triad/7th of a different root.
- Source: https://github.com/ohollo/chord-extractor ; http://www.isophonics.net/nnls-chroma

### BTC — Bi-directional Transformer (Park et al., ISMIR 2019)
- CQT input over a 10s window, multi-head self-attention. Two modes via the `voca` flag:
  - `voca=False` → 25 classes (12 maj + 12 min + N)
  - `voca=True`  → "large vocabulary" (~170 classes, maj/min/maj7/min7/dom7/dim/aug/hdim7/sus + inversions; **no extended 9/11/13/alt**).
- Trained on Isophonics + Robbie Williams + UsPop2002 (pop, not jazz).
- Source: https://github.com/jayg996/BTC-ISMIR19 ; https://arxiv.org/abs/1907.02698

### Large-Vocabulary Chord Transcription via Chord Structure Decomposition (Jiang & Chen, ISMIR 2019)
- **301 classes**: triads (maj/min/aug/dim), inverted triads, sevenths (maj7/7/min7/dim7/hdim7), **extended (maj9, 9, min9, 11, 13)**, suspended (sus4, sus2, sus4(b7)), slash chords, plus N.
- This is the most ambitious public vocabulary, but training data is still pop-skewed; rare-class accuracy is poor due to long-tail imbalance.
- Source: https://github.com/music-x-lab/ISMIR2019-Large-Vocabulary-Chord-Recognition

### ChordFormer (Wang et al., arXiv Feb 2025) — current open SotA
- Conformer backbone (CNN + Transformer). Targets **structural** decomposition: root, bass, triad, sevenths.
- Reported on large-vocab pop datasets: **Root 84.69%, MajMin 84.09%, MIREX 83.62%**; ~+2% frame-wise / +6% class-wise vs prior SotA (BTC, CRNN baselines).
- Addresses class imbalance with reweighted loss — but vocabulary is still triads/bass/sevenths, **not** 9/11/13/alt.
- Source: https://arxiv.org/abs/2502.11840

### Harmony Transformer (Chen & Su) / BTC-FDAA-FGF
- Multitask: chord segmentation + classification. Marginal gains over BTC, same vocabulary tier.
- Source: https://transactions.ismir.net/articles/10.5334/tismir.65

### Madmom DeepChroma + DeepChromaChordRecognitionProcessor
- Deep-chroma front-end + CRF decoder. **Major + minor only** — 25 classes. Useful as a fast baseline; useless for jazz semantics.
- Source: https://madmom.readthedocs.io/en/v0.16/modules/features/chords.html

### Essentia ChordsDetection
- HPCP + HMM. **Major/minor triads only** (e.g., "C", "C#m"). No sevenths.
- Source: https://essentia.upf.edu/reference/std_ChordsDetection.html

### ChordSync (Pasini et al., SMC 2024)
- Conformer + CTC forced **alignment** of pre-existing chord labels to audio — not recognition from scratch. Useful when chord chart already exists (e.g., from a fake book) and you want timing.
- Source: https://arxiv.org/abs/2408.00674

### LLM Chain-of-Thought ACR (arXiv 2509.18700, 2025)
- Uses GPT-4o on top of a baseline recognizer (the 301-class large-vocab model) with a 5-stage CoT pipeline. Suggests genre-specific prompts could specialize for jazz harmonies. Modest gains, but vocabulary is bounded by what the underlying model can emit.
- Source: https://arxiv.org/html/2509.18700v1

### DECIBEL (Odekerken et al., TISMIR 2021)
- Fuses audio ACR with crowd-sourced MIDI/tab via DTW alignment. Adds **+0.5 to +13.6 percentage points** over base ACR — but only when matching tab/MIDI exists for the song. Not applicable to a freshly-recorded improvisation.
- Source: https://transactions.ismir.net/articles/10.5334/tismir.81

### Foundation models (MERT, JukeMIR, OMAR-RQ)
- MERT (arXiv 2306.00107) and JukeMIR are evaluated on chord recognition as a **probing** downstream task, typically on the small (Maj/Min) vocabulary. MERT performs well thanks to a CQT-reconstruction auxiliary loss. None target extended/altered jazz vocab; their chord head is a small MLP probe with the standard limited vocab.

### Commercial systems
| Tool | Claimed jazz vocab | Real jazz behavior |
|---|---|---|
| **Chordify** | maj/min + sevenths | Drops from ~85% (pop) to **<40%** on unaccompanied jazz guitar; misses Cmaj9#11; users report ~70% wrong on funk/jazz uploads. Does **not** detect upper extensions. |
| **Chord AI / chordai.net** | Pro tier lists 6/9/11/13, add9/11/13, 69, 11b5, 13b9, 9sus4, min7add13, alt | Reviews say "accuracy drops on complex jazz or avant-garde music." Best of the consumer apps for jazz. |
| **Moises (Advanced Chord Detection, 2024-25)** | Easy/Medium/Advanced tiers; "extended chords for Jazz and Bossa" | No published accuracy numbers; user-facing only. |
| **Klangio Transcription Studio** | Lead-sheet style chord symbols + melody | Marketed for jazz; "noisy or overlapping instruments can trip it up." |
| **AnthemScore** | Chord mode mostly triads/7ths | Single-instrument focus; weak on dense quartet mixes. |
| **Samplab Chord Finder** | Triads + 7ths | Pop-oriented. |
| **MuseScore "Chord Identifier (Pop & Jazz)" plugin** | Symbolic only — names notes already on staff. Will write `C11(b9/b13)`, `Bb7#5`, `F7(b9)`. Does **not** do audio. Useful as the *back end* of a hybrid pipeline. |

## Datasets

| Dataset | Genre | Size | Vocab annotated | Notes |
|---|---|---|---|---|
| **Isophonics** | Pop (Beatles, Queen, Carole King, Zweieck) | 225 songs | Maj/min + extended (Harte syntax) | Standard pop benchmark since MIREX 2009 |
| **McGill Billboard** | Pop | 740 songs (~1000 collected, MIREX test set kept private) | Full Harte syntax incl. extensions | Largest pop benchmark |
| **RWC, USPOP, Robbie Williams** | Pop | misc | Triads + 7ths | Common training augmenters |
| **JAAH (Jazz Audio-Aligned Harmony)** | Jazz | **113 tracks** (Smithsonian Anthology) | **5 classes only: maj, min, dom7, hdim7, dim** + meter/structure | The **only** dedicated jazz benchmark; deliberately collapses 9/11/13/alt into their underlying 7th. https://mtg.github.io/JAAH/ |
| **iRb (iReal Pro corpus)** | Jazz | ~1300 standards | Symbolic chord changes only — no audio | Useful for chord-language priors / LM pretraining |
| **Schubert Winterreise (SWD)** | Classical | 9 performances of 24 songs | Chords + keys + structure | Maj/min triads = 66% of duration; dom7 + dim ~25%. Hard for ACR; not jazz. |
| **jazznet (Akinwale et al., ICASSP 2023)** | Synthesized solo piano | **162,520 patterns**, ~26k h | chords / arpeggios / scales / progressions in all keys, all inversions | Pattern-level, not real performance audio. Good for pretraining a jazz-piano-aware classifier. https://arxiv.org/abs/2302.08632 |
| **PiJAMA** | Jazz piano (solo) | 200 h, 2,777 perfs, 120 pianists | Auto-MIDI annotations (no chord symbols) | https://transactions.ismir.net/articles/10.5334/tismir.162 |
| **Jazz Trio Database** | Jazz piano trio | 44.5 h | Auto-annotated via Demucs + onset detection | Useful for stem-based pipelines. https://transactions.ismir.net/articles/10.5334/tismir.186 |
| **Chordonomicon** | Multi-genre | 666,000 songs' chord progressions (symbolic) | Full Harte | Symbolic LM pretraining only. https://arxiv.org/html/2410.22046v1 |
| **AAM (Artificial Audio Multitracks)** | Synthetic | algorithmic | Full Harte | Used in 2025 paper "Training chord recognition models on artificially generated audio" (https://arxiv.org/abs/2508.05878) |

Reported baseline jazz numbers: **JAAH yields ~42% chord-estimation accuracy** with off-the-shelf algorithms vs ~75-87% on Isophonics/Billboard — even though JAAH only labels 5 chord classes. That gap is the headline finding for jazz feasibility.

## Direct vs. Separated Stem

### Findings from the literature
- Daniel Ko (Wisconsin, 2023, https://ko28.github.io/chord-transcription/): pre-process with HT-Demucs, **boost the "other" stem volume** (which contains piano + guitar harmony), then run ACR. Achieves consistent improvements on pop datasets.
- "Accuracy Improvement of Automatic Chord Recognition…" (APSIPA 2025, http://www.apsipa.org/proceedings/2025/papers/APSIPA2025_P307.pdf): combining drum-removed, drum+vocal-removed, and isolated-bass stems to vote on root vs. quality gives **+1 to +2.7 pp on MIREX** across three datasets. The bass stem is used only for **root** (it has the strongest root anchor); the harmony stem decides quality.
- Demucs `htdemucs_6s` adds explicit **piano** and **guitar** stems, but the Demucs authors themselves note "the piano source is not working great at the moment" — separation artifacts hurt downstream ACR more than the cleaner spectrum helps, especially on a piano voicing where exact upper-register partials matter.

### Practical implication for a jazz quartet
- A jazz quartet (piano, bass, drums, sax/trumpet) is **the worst case for vocal/drum-removal pipelines**: there is no vocal to remove, drums bleed harmonically little, and the "other" stem already is mostly the piano + horn. So the upside of separation is small.
- A horn solo on top of piano comping will **confuse** chroma-based ACR because the horn often plays tensions (9/11/13) that the chord-tone classifier interprets as new chords. Removing the horn (treating it as "vocal") would actually help — but no off-the-shelf model isolates a saxophone stem cleanly.
- Best-case stem strategy for this app: **Demucs htdemucs_6s → keep "piano" stem only → run high-resolution piano transcription (Bytedance/MAESTRO, F1>90% on solo piano)** → infer chord symbol from pitch set (next section). Expect 5-15 pp degradation vs. a clean solo-piano recording due to bleed.

## Hybrid Approaches (Pitch -> Chord Symbol)

This is the most promising path for a jazz product, and it's underexplored in the academic ACR literature (which mostly treats chord symbol as the direct classification target).

### Pipeline
1. **Source separation**: Demucs htdemucs_6s → piano stem.
2. **Multi-pitch / piano transcription**: Bytedance piano_transcription (https://github.com/bytedance/piano_transcription, MAESTRO-trained, >90% note F1) or Onset-and-Frames. Output: time-aligned MIDI of the piano.
3. **Beat / harmonic-rhythm segmentation**: madmom or beat-tracker; group MIDI notes into harmonic windows (typically 1 chord per bar or per 2 beats in jazz).
4. **Chord-symbol inference from pitch set**:
   - **Rule-based**: enumerate candidate roots; score each using jazz voicing priors (rootless A/B voicings, shell voicings, quartal stacks). The MuseScore "Chord Identifier (Pop & Jazz)" plugin is an open-source reference implementation that already labels things like `C11(b9/b13)`, `Bb7#5`, `F7(b9)` from a pitch set — porting that logic is feasible.
   - **LLM-based**: feed the pitch set + recent chord history to a small LLM with a jazz-theory system prompt; output the chord symbol. This is essentially what arXiv 2509.18700 does at a higher level, but using GPT for the *symbolization* step (not the recognition step) is cheaper and aligns better with how jazz musicians read voicings.
   - **Hybrid**: rule-based symbol + LLM tiebreaker for rootless / ambiguous voicings.

### Why this beats end-to-end ACR for jazz
- Decouples the hard acoustic problem (pitch extraction) from the hard music-theory problem (voicing → symbol). Pitch extraction has a strong supervised signal (MAESTRO); voicing → symbol can be done with rules + an LLM-as-judge with no additional training data.
- The hard-to-train tail (b9, #11, alt) becomes easy: if you know the pitch set is {C, E, G, Bb, Db, F#}, naming it `C7b9#11` is deterministic.
- Jazz pianists usually omit the root (rootless) and the 5th (shell). Rule sets that explicitly enumerate "A-voicing", "B-voicing", "So What", "Kenny Barron" templates can label these correctly where chroma-based ACR cannot.

### Caveats
- Piano transcription degrades on multi-instrument mixes even after separation; expect note F1 closer to 70-80% on a real quartet, not 90%. Missed pitches → wrong chord symbol.
- Sustain pedal blurs harmonic rhythm; segmentation is non-trivial.
- Tensions in the horn line (sax playing the 9 or 13) will sneak into the symbol unless stem isolation is clean.
- No published end-to-end accuracy numbers exist for this hybrid path on JAAH or any jazz benchmark — would have to be measured.

## Verdict

**For the use case (15s jazz quartet, "what is the piano playing?")**:

- **Realistic accuracy with off-the-shelf ACR (BTC large-vocab, ChordFormer, Chord AI Pro)**: ~75-85% on **root**, ~60-75% on **triad + dom7 quality**, **<40-50%** on the full extended/altered chord symbol the pianist played. Output will *feel* like dumbed-down triads to a jazz user — Cmaj9#11 will read as "C" or "Cmaj7"; G7alt will read as "G7"; quartal voicings will read as random sus chords or be labeled `N`.
- **With a hybrid pitch→symbol pipeline (Demucs piano stem → Bytedance transcription → rule/LLM symbol)**: theoretically much better on tensions when the pitch set is captured, but bottlenecked by transcription quality on a non-solo-piano mix. Best path forward, but requires custom engineering and there is no public benchmark proving it works on quartet audio.
- **Best commercial option today**: **Chord AI Pro** — has the richest published vocab (13b9, 9sus4, min7add13, alt) and decent jazz reviews. **Moises Advanced** is a credible second. Both will still feel coarse on Bill Evans / Herbie Hancock-grade voicings.
- **What to ship for v1**: be honest about the chord-vocab tier. Expose a tier toggle (Triads / Sevenths / Extended) and degrade gracefully. Surface confidence per chord. Show the **transcribed pitch set** alongside the chord symbol so a jazz user can self-correct ("oh, that's actually a 13b9, not a 7"). Plan a hybrid pitch→symbol pipeline for v2 if jazz users are the main audience.
- **Biggest gap in the field**: no public model is **trained** on extended/altered jazz piano voicings with audio supervision. JAAH only annotates 5 chord classes. iRb is symbolic. jazznet is synthesized patterns. PiJAMA has no chord labels. Until somebody publishes a "Real Book Audio" dataset with full Harte-syntax annotations, jazz ACR will lag pop ACR by a generation.

## Sources

- ChordFormer (2025, current open SotA, 84.7%/84.1%/83.6% on Root/MajMin/MIREX): https://arxiv.org/abs/2502.11840 ; https://arxiv.org/html/2502.11840v1
- BTC (ISMIR 2019, transformer ACR, 25 / ~170 class modes): https://arxiv.org/abs/1907.02698 ; https://github.com/jayg996/BTC-ISMIR19
- Large-Vocabulary Chord Transcription via Structure Decomposition (ISMIR 2019, **301 classes** including 9/11/13/sus): https://github.com/music-x-lab/ISMIR2019-Large-Vocabulary-Chord-Recognition
- LLM Chain-of-Thought ACR (2025): https://arxiv.org/html/2509.18700v1 ; https://www.arxiv.org/pdf/2509.18700
- ChordSync (SMC 2024, conformer alignment): https://arxiv.org/abs/2408.00674 ; https://github.com/andreamust/ChordSync
- Madmom DeepChromaChordRecognitionProcessor (Maj/Min only): https://madmom.readthedocs.io/en/v0.16/modules/features/chords.html
- Essentia ChordsDetection (Maj/Min only): https://essentia.upf.edu/reference/std_ChordsDetection.html
- Chordino / NNLS Chroma + chord-extractor wrapper: https://github.com/ohollo/chord-extractor ; http://www.isophonics.net/nnls-chroma
- DECIBEL (TISMIR, +0.5 to +13.6 pp via tab/MIDI fusion): https://arxiv.org/abs/2002.09748 ; https://transactions.ismir.net/articles/10.5334/tismir.81
- Harmony Transformer / Attend-to-Chords: https://transactions.ismir.net/articles/10.5334/tismir.65
- BACHI symbolic ACR (2025): https://arxiv.org/html/2510.06528
- Training chord recognition on artificial audio (2025): https://arxiv.org/abs/2508.05878
- Foundation model survey / MERT chord probe: https://arxiv.org/html/2306.00107v3 ; https://arxiv.org/html/2409.09601
- JAAH dataset (113 jazz tracks, 5-class vocab, ~42% baseline): https://mtg.github.io/JAAH/ ; https://archives.ismir.net/ismir2018/paper/000206.pdf
- jazznet dataset (162k piano patterns, ICASSP 2023): https://arxiv.org/abs/2302.08632 ; https://github.com/tosiron/jazznet
- PiJAMA (jazz piano MIDI, no chord labels): https://transactions.ismir.net/articles/10.5334/tismir.162
- Jazz Trio Database (Demucs-annotated): https://transactions.ismir.net/articles/10.5334/tismir.186
- Chordonomicon (666k symbolic): https://arxiv.org/html/2410.22046v1
- Schubert Winterreise (classical baseline; maj/min triads only 66% of duration): https://www.audiolabs-erlangen.de/fau/assistant/weiss/data
- ACR by music source separation (Daniel Ko, Wisconsin): https://ko28.github.io/chord-transcription/
- APSIPA 2025 multi-stem ACR (+1 to +2.7 pp): http://www.apsipa.org/proceedings/2025/papers/APSIPA2025_P307.pdf
- Demucs (htdemucs_6s with piano+guitar stems): https://github.com/facebookresearch/demucs
- Bytedance high-resolution piano transcription (>90% note F1): https://github.com/bytedance/piano_transcription
- MIREX 2024 Audio Chord Estimation: https://music-ir.org/mirex/wiki/2024:Audio_Chord_Estimation
- 20 Years of Automatic Chord Recognition (ISMIR 2019 review): https://archives.ismir.net/ismir2019/paper/000004.pdf
- MIR blog state-of-the-art ACR review (~83% MajMin SotA): https://musicinformationretrieval.wordpress.com/2017/03/06/state-of-the-art-audio-chord-estimation-algorithms-evaluation/
- Chordify reviews (drops to <40% on jazz, misses Cmaj9#11): https://www.guitarchalk.com/chordify-review/ ; https://www.alibaba.com/product-insights/ai-powered-guitar-chord-recognizer-vs-chordify-for-complex-jazz-progressions-which-reads-voicings-correctly.html
- Chord AI (Pro vocab includes 13b9, 9sus4, min7add13, alt): https://chordai.net/
- Moises Advanced Chord Detection (Easy/Medium/Advanced jazz tier): https://moises.ai/blog/moises-news/advanced-chord-detection/
- Klangio (lead-sheet jazz transcription): https://klang.io/transcription-studio/
- MuseScore "Chord Identifier (Pop & Jazz)" plugin (symbolic, full jazz vocab incl. C11(b9/b13)): https://musescore.org/en/project/chord-identifier-pop-jazz
- Samplab Chord Finder: https://samplab.com/chord-detection
