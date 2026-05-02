# Direct Mix vs. Separated Piano Stem for Chord Recognition

## Summary

**Verdict:** For a 15-second jazz quartet clip where the user wants the *piano comping chords*, the recommended pipeline is a **hybrid**: do **light source separation as preprocessing** (drums removed, optionally vocals removed), but **keep the bass+piano+other together** as a single "harmonic mix" for the chord recognizer. Do **not** isolate `htdemucs_6s`'s piano stem alone — its quality is documented as poor (heavy artifacts and bleed), and it strips away the bass which carries the chord root that comping voicings deliberately omit (rootless voicings, common in modern jazz piano).

**Reasoning, in order of evidence weight:**

1. **The piano stem of `htdemucs_6s` is officially flagged as low-quality.** The Demucs README states the piano source "doesn't work so well at the moment" with "a lot of bleeding and artifacts." Multiple secondary sources echo this. Running a chord recognizer on a noisy/artifacted piano-only stem trades one source of error (mix interference) for another (separation artifacts).
2. **Existing research consistently shows that *full* separation as preprocessing hurts chord recognition more than it helps**, because (a) the separation introduces artifacts that degrade chroma features, and (b) chord recognizers are *trained* on full mixes and have learned to be robust to drums and vocals in context. Daniel Ko's UW-Madison study trained a chord recognizer on Demucs-separated audio vs. original mix and found the separated version performed *worse on average across all MIREX metrics*.
3. **However, *partial* separation — specifically removing drums and (sometimes) vocals while keeping bass and harmonic instruments — does help.** HPSS-style preprocessing has a long history (Ono et al. 2008/2010) of reducing "relative chord-recognition error rate by 28%" by suppressing percussive interference. The recent APSIPA 2025 paper "Accuracy Improvement of Automatic Chord Recognition with Source Separation Preprocessing" confirms this with modern Demucs: separate into 4 stems, **boost** the pitched stems (vocals, bass, other) and **suppress** drums, then recombine. They report 1–2.7% MIREX accuracy gains.
4. **Bass should be kept, not removed.** The bass is the harmonic foundation of jazz; jazz pianists routinely play *rootless* voicings (Bill Evans-style, A and B voicings — 3-5-7-9 or 7-9-3-5) precisely *because* the bassist supplies the root. Removing the bass stem before chord recognition would strip away the root information for a large fraction of jazz piano comping. The recent (Sept 2025) "Enhancing ACR through LLM Chain-of-Thought Reasoning" paper uses the bass stem as an explicit auxiliary input rather than discarding it, confirming this.
5. **Jazz is a hard regime regardless.** State-of-the-art chord recognizers achieve ~80% MIREX accuracy on Beatles/Isophonics-style pop, but only ~32–40% on jazz datasets like JAAH (CREMA: 40.26% on Jazz5; Chordino: 32.68%). Whatever pipeline you choose, expect a roughly 2x error rate vs. pop benchmarks.

## Studies and Evidence

### 1. Direct comparison: separated input hurts when used naively

**Daniel Ko, UW-Madison — "Automatic Chord Recognition by Music Source Separation"** ([ko28.github.io](https://ko28.github.io/chord-transcription/))
- Trained two CNN chord recognizers on identical architecture: one on raw Beatles audio, one on Demucs "Other" stem (which holds piano/guitar/keys).
- Result: **the Demucs-trained model performed worse across all MIREX metrics** (Root, Thirds, Maj-Min, Triads, Sevenths, Tetrads, MIREX).
- Cause: Demucs separation artifacts vary widely per song, "ranging from unnoticeable to barely listenable," and the chroma features computed on artifacted audio are degraded.
- Implication for our use case: don't *replace* the full mix with an isolated harmonic stem.

### 2. Partial separation as preprocessing — works, modestly

**APSIPA 2025 — "Accuracy Improvement of Automatic Chord Recognition with Source Separation Preprocessing"** ([IEEE Xplore 11249321](https://ieeexplore.ieee.org/document/11249321/), [APSIPA paper P307](http://www.apsipa.org/proceedings/2025/papers/APSIPA2025_P307.pdf))
- Pipeline: Demucs 4-stem split → **boost** the pitched stems (vocals, bass, "other") → **attenuate** drums → recombine into a single track → run chord recognizer.
- Reported gain: **1–2.7% MIREX accuracy improvement** across multiple datasets vs. baseline.
- Key insight: don't discard stems, *re-balance* them. This is essentially "loudness-aware HPSS via deep separation."

**arXiv 2509.18700 — "Enhancing Automatic Chord Recognition through LLM Chain-of-Thought Reasoning" (Sept 2025)** ([arxiv.org/abs/2509.18700](https://arxiv.org/abs/2509.18700))
- Uses HT Demucs to generate three auxiliary tracks: (a) **drum-removed**, (b) **drums-and-vocals-removed**, (c) **isolated bass stem**.
- Rationale stated: "Removing drums eliminates non-tonal percussive interference that may confuse the chord recognition model... drums-and-vocals-removed track focuses purely on instrumental harmony from bass and accompanying instruments."
- The isolated bass stem is *not* discarded — it is fed in as a separate input so the LLM-augmented system can reason about root motion.
- Confirms the pattern: drums hurt, vocals can hurt, bass helps (don't remove it).

**Ono et al., "Harmonic and Percussive Sound Separation and Its Application to MIR-Related Tasks"** ([Springer chapter](https://link.springer.com/chapter/10.1007/978-3-642-11674-2_10), [ResearchGate 225805350](https://www.researchgate.net/publication/225805350))
- Classical HPSS (anisotropic spectrogram diffusion) applied as preprocessing.
- "Weakly guided separation of percussive and harmonic content... helped... reducing the relative error rate for chord recognition by 28%."
- This is the canonical citation that "remove the drums" helps chord recognition. It pre-dates Demucs and has been replicated many times.

### 3. End-to-end models implicitly do their own separation

**BTC (Bi-directional Transformer for Chord Recognition, Park & Choi, ISMIR 2019)** ([arxiv.org/abs/1907.02698](https://arxiv.org/abs/1907.02698), [github BTC-ISMIR19](https://github.com/jayg996/BTC-ISMIR19))
- Self-attention maps show the model learns to attend selectively to harmonically-relevant frames and frequencies. Early layers use local context; middle layers broaden the receptive field; final layer attends "only on essential information for chord recognition."
- Implication: a strong end-to-end model already learns to suppress drums/vocals internally. Pre-separating may be redundant or harmful.

**ChordFormer (2025)** ([arxiv.org/html/2502.11840v1](https://arxiv.org/html/2502.11840v1))
- Conformer-based architecture, 2% frame-wise accuracy and 6% class-wise accuracy improvement on large-vocabulary chord datasets. Same theme: a stronger end-to-end model needs less preprocessing.

### 4. Demucs `htdemucs_6s` piano stem quality

- **Official Demucs README** ([github.com/facebookresearch/demucs](https://github.com/facebookresearch/demucs)): the piano source "doesn't work so well at the moment" with "a lot of bleeding and artifacts."
- **MVSep, Stem Splitter reviews** ([stemsplitter.github.io/demucs-vs-spleeter/](https://stemsplitter.github.io/demucs-vs-spleeter/), [mvsep.com/algorithms/3](https://mvsep.com/algorithms/3)): "experimental 6-source models... currently exhibit artifacts and stem bleeding, particularly in the piano source." Guitar stem is "okay"; piano is "so-so."
- For jazz quartets specifically (piano + bass + drums + horn), the standard 4-stem `htdemucs` is reported as "noticeably cleaner on jazz and acoustic recordings where instruments share a lot of frequency content," but the 6-stem extension is still experimental.
- **Practical implication:** if you isolate the piano stem and feed it to the chord recognizer, you're feeding it audio with bleed from the trumpet/sax + bass + drum cymbals + spectral artifacts. That is *worse* than the original mix on chroma features.

### 5. Bass removal — counter-productive for jazz

- Walking bass lines in jazz emphasize the **root and 5th** of each chord ([thejazzpianosite.com walking bass](https://www.thejazzpianosite.com/jazz-piano-lessons/jazz-chord-voicings/walking-bass-lines/), [studybass.com](https://www.studybass.com/lessons/harmony/chord-tones-in-basslines/)).
- "The most direct way to figure out a chord progression is to focus on the roots of all the chords, which means listening to what the bass player is doing" ([Jazzadvice](https://www.jazzadvice.com/lessons/how-to-hear-chord-changes/)).
- Chordino/NNLS-Chroma uses a **24-dim chromagram with a separate bass-chroma channel** because the bass register is so informative for the root ([isophonics.net/nnls-chroma](http://www.isophonics.net/nnls-chroma)). Mauch & Dixon (ISMIR 2010) showed this lifts accuracy on difficult chords.
- Conclusion: removing bass is counterproductive in general and *especially* counterproductive when the harmonic instrument is playing rootless voicings.

### 6. Chroma feature behavior under separation

- Chroma features are sensitive to "transients and noise" (kick drums, cymbals, sibilants in vocals) — HPSS-style preprocessing addresses this and reliably improves chroma quality.
- However, chroma is *also* sensitive to spectral artifacts introduced by neural separators. The separator introduces broadband phase artifacts and missing partials; Ko's experiment shows this can wipe out the gain from removing drums.
- The classical "deep chroma extractor" (Korzeniowski & Widmer, ISMIR 2016, [arxiv 1612.05065](https://arxiv.org/abs/1612.05065)) was trained directly on full mixes and learned its own noise robustness; it *does not* benefit from being given separated audio.

## Comping vs. Solo Piano

This is critical for the use case (a piano *comping in a quartet*, not solo piano).

**Comping voicings are typically rootless.**
- Bill Evans/Wynton Kelly/Ahmad Jamal-style "A and B voicings" leave out the root (and often the 5th), playing only the 3-5-7-9 or 7-9-3-5 ([pianogroove.com rootless](https://www.pianogroove.com/jazz-piano-lessons/rootless-chord-voicings/), [thejazzpianosite.com rootless](https://www.thejazzpianosite.com/jazz-piano-lessons/jazz-chord-voicings/rootless-voicings/), [pianowithjonny.com rootless](https://pianowithjonny.com/piano-lessons/rootless-voicings-for-piano-the-complete-guide/)).
- "Chord shells" provide root + 3 + 7 or 3 + 7 only — the 3rd and 7th carry the chord *quality* and the bass supplies the root.
- Quartal voicings (So-What chord, McCoy Tyner) stack 4ths and are deliberately ambiguous between several chord interpretations.
- **Direct consequence:** the piano stem *alone* often does not contain enough information to identify the chord. The recognizer needs the bass.

**Comping is also sparse and syncopated.**
- Comping rhythms are deliberately off-beat, with rests. A chord recognizer trained on continuous pop guitar strumming will see long silences in a piano-only stem.
- The full mix preserves continuous harmonic energy (bass sustains, cymbals wash) that helps the recognizer's frame-level classifier produce stable estimates and helps the smoother (HMM/CRF) avoid spurious chord changes.

**Solo piano is the easier case.** Modern solo-piano transcribers (Klangio Piano2Notes, Onsets-and-Frames-style models) handle dense block-chord/stride playing well because all the chord tones (including root) are present and there is no other instrument competing for the chroma. But that's the *opposite* of comping in a quartet.

**Modern jazz piano techniques the recognizer will struggle with regardless:**
- Quartal/quintal voicings (e.g., So-What chord = perfect 4ths stacked) — read as sus chords or ambiguous.
- Upper-structure triads (a major triad over a different bass) — recognizer often picks the upper triad and misses the slash.
- Slash chords / pedal points — recognizer struggles when bass and harmony disagree.
- Tritone substitutions — sometimes resolved correctly because the tritone (3-7) is preserved.
- Modal vamps with no root motion — recognizers tuned for ii-V-I cadences may oscillate.
- Reharmonizations changing every 1–2 beats — most ACR models smooth too aggressively.

## Recommended Pipeline

For the 15s jazz quartet → piano chords use case:

```
[1] Input: 15s mix (44.1 kHz stereo)
       │
       ▼
[2] htdemucs (4-stem) → drums, bass, vocals, other
       │
       ▼
[3] Re-mix WITHOUT solo instrument and WITHOUT drums:
       harmonic_mix = 1.0 * other + 1.0 * bass + 0.0 * drums + 0.3 * vocals
       (vocals stem in instrumental jazz often catches the trumpet/sax —
        attenuate but don't fully remove, or remove if the soloist bleeds
        clearly into "other")
       │
       ▼
[4] Run chord recognizer on harmonic_mix
       Recommended: BTC (ISMIR 2019) or ChordFormer (2025) for the
       chroma + transformer approach; or Chordino/NNLS-Chroma for
       a classic chroma + HMM baseline that uses bass-chroma explicitly
       │
       ▼
[5] Optional auxiliary signal: pitch-track the bass stem
       (e.g., basic-pitch or CREPE on the bass) → root sequence
       Use root sequence to disambiguate rootless voicings
       (i.e., if recognizer says Cm7 but bass plays F, output is F7sus
        or Cm7/F).
```

**Do NOT:**
- Use `htdemucs_6s` and run chord recognition on its piano stem alone. Documented to be artifact-heavy and strips away the root information rootless voicings depend on.
- Remove the bass stem. The bass carries the root, especially in jazz.
- Apply heavy de-noising or de-reverberation as preprocessing — chord recognizers are trained on real recordings and like the natural reverb tail.

**DO:**
- Remove drums (well-established 28% relative error reduction from HPSS-era work).
- Run an end-to-end transformer-based ACR model (BTC, ChordFormer) on the drum-suppressed mix — these models already learn implicit "separation" via attention.
- If serious about jazz accuracy, consider fine-tuning on JAAH (113 jazz tracks with chord labels). Off-the-shelf ACR scores ~32–40% on jazz vs. ~80% on pop — there is a known domain gap.
- Cross-reference with bass-line root tracking to handle rootless voicings.
- Set user expectations: even with the optimal pipeline, jazz comping chord recognition will be substantially less reliable than pop-song chord recognition. A 15s clip with a clear ii-V-I and a steady comp is the best case; modal vamps, fast reharms, and quartal voicings will produce errors.

## Sources

### Primary research
- [APSIPA 2025 — Accuracy Improvement of Automatic Chord Recognition with Source Separation Preprocessing (IEEE Xplore)](https://ieeexplore.ieee.org/document/11249321/)
- [APSIPA 2025 paper P307 PDF](http://www.apsipa.org/proceedings/2025/papers/APSIPA2025_P307.pdf)
- [Enhancing Automatic Chord Recognition through LLM Chain-of-Thought Reasoning (arXiv 2509.18700, Sept 2025)](https://arxiv.org/abs/2509.18700)
- [Daniel Ko — Automatic Chord Recognition by Music Source Separation (UW-Madison)](https://ko28.github.io/chord-transcription/)
- [BTC — Bi-Directional Transformer for Musical Chord Recognition (Park & Choi, ISMIR 2019)](https://arxiv.org/abs/1907.02698)
- [BTC GitHub](https://github.com/jayg996/BTC-ISMIR19)
- [ChordFormer — Conformer-Based Architecture for Large-Vocabulary ACR (2025)](https://arxiv.org/html/2502.11840v1)
- [Deep Chroma Extractor (Korzeniowski & Widmer, ISMIR 2016)](https://arxiv.org/abs/1612.05065)
- [Ono et al. — Harmonic and Percussive Sound Separation and Its Application to MIR-Related Tasks (Springer)](https://link.springer.com/chapter/10.1007/978-3-642-11674-2_10)
- [HPSS LabCourse (Müller, Audiolabs Erlangen)](https://www.audiolabs-erlangen.de/content/05_fau/professor/00_mueller/02_teaching/2017w_mpa/LabCourse_HPSS.pdf)
- [Mauch & Dixon — Approximate Note Transcription for the Improved Identification of Difficult Chords (ISMIR 2010)](https://www.researchgate.net/publication/220723830_Approximate_Note_Transcription_for_the_Improved_Identification_of_Difficult_Chords)
- [NNLS Chroma / Chordino (Isophonics)](http://www.isophonics.net/nnls-chroma)
- [JAAH — Audio-Aligned Jazz Harmony Dataset (Eremenko et al., ISMIR 2018)](https://archives.ismir.net/ismir2018/paper/000206.pdf)
- [JAAH dataset site](https://mtg.github.io/JAAH/)
- [JAAH GitHub](https://github.com/MTG/JAAH)
- [State of the Art in Audio Chord Estimation (MIR blog, comparing Chordino vs CREMA on jazz)](https://musicinformationretrieval.wordpress.com/2017/02/06/state-of-the-art-in-audio-chord-estimation/)
- [State of the Art Audio Chord Estimation algorithms evaluation (jazz numbers)](https://musicinformationretrieval.wordpress.com/2017/03/06/state-of-the-art-audio-chord-estimation-algorithms-evaluation/)
- [Training chord recognition models on artificially generated audio (arXiv 2508.05878)](https://arxiv.org/pdf/2508.05878)
- [20 Years of Automatic Chord Recognition (ISMIR 2019 retrospective)](https://archives.ismir.net/ismir2019/paper/000004.pdf)

### Demucs / source separation
- [Demucs GitHub (facebookresearch)](https://github.com/facebookresearch/demucs)
- [Demucs vs Spleeter — Stem Splitter review](https://stemsplitter.github.io/demucs-vs-spleeter/)
- [MVSep — htdemucs algorithm description](https://mvsep.com/algorithms/3)
- [Demucs Open Laboratory model card](https://openlaboratory.ai/models/demucs)

### Jazz piano voicing references
- [Rootless Voicings — PianoGroove](https://www.pianogroove.com/jazz-piano-lessons/rootless-chord-voicings/)
- [Rootless Voicings — The Jazz Piano Site](https://www.thejazzpianosite.com/jazz-piano-lessons/jazz-chord-voicings/rootless-voicings/)
- [Rootless Voicings Complete Guide — Piano with Jonny](https://pianowithjonny.com/piano-lessons/rootless-voicings-for-piano-the-complete-guide/)
- [Walking Bass-lines — TJPS](https://www.thejazzpianosite.com/jazz-piano-lessons/jazz-chord-voicings/walking-bass-lines/)
- [How to Hear Chord Changes — Jazzadvice](https://www.jazzadvice.com/lessons/how-to-hear-chord-changes/)
- [Chord Tones in Basslines — StudyBass](https://www.studybass.com/lessons/harmony/chord-tones-in-basslines/)
