# Brass / Trumpet Monophonic Pitch Tracking

## Summary

Frame-level pitch (F0) on a clean monophonic trumpet stem is largely a solved
problem: CREPE and successors hit ~95-99% raw pitch accuracy at the 50-cent
threshold on synthetic and clean instrumental sets, and >90% at 10 cents.
The hard part is going from a continuous F0 contour to discrete, musically
meaningful **notes** (onset/offset/pitch/dynamics) on jazz playing - and on
that task realistic note-F1 expectations on a clean separated trumpet jazz
solo are roughly **70-85% (no-offset note F1)**, extrapolating from the closest
published result we have: CREPE Notes hits ~82% no-offset F1 on Filosax
(jazz saxophone), ~74% on Irish flute, but the bebop Omnibook benchmark is
"considerably lower" out-of-distribution. No public model is trained or
benchmarked specifically on jazz **trumpet** notes.

The dominant failure mode is **note segmentation, not pitch identification**:
vibrato, scoops/smears, and legato slurs cause spurious note splits or
merges, and fast bebop lines (16th notes at 240 BPM = ~16 notes/sec) push
onset detection past its temporal resolution. Octave errors from missing
fundamental in brass spectra are a secondary risk. Top recommendation for
this use case: **CREPE (or PESTO) for F0 + CREPE Notes for segmentation**, with
custom post-processing that explicitly suppresses vibrato and merges
gliss/scoop trajectories.

## Tools

### Frame-level F0 trackers

- **CREPE** (Kim et al., ICASSP 2018). 6-layer CNN on raw waveform.
  - On MDB-stem-synth: 0.967 / 0.953 / 0.909 RPA at 50/25/10-cent thresholds;
    0.970 RCA at 50 cents. Outperforms pYIN and SWIPE by ~8% at 10 cents.
  - MedleyDB: >90% at 10 cents. RWC-synth: near 100%.
  - Per-instrument breakdown exists in the original paper (230 tracks across
    instruments including trumpet) but no isolated published trumpet-only
    F1; brass typically lands close to the dataset mean because trumpet has a
    strong, mostly stable F0.
  - Standard reference baseline; PyPI package `crepe`.
  - https://arxiv.org/abs/1802.06182 / https://github.com/marl/crepe

- **PESTO** (Riou et al., ISMIR 2023, Sony CSL Paris). Self-supervised,
  CQT-based, transposition-equivariant.
  - ~30k params (~800x smaller than CREPE), 12x faster than real-time on CPU,
    trained without annotated data.
  - Comparable accuracy to CREPE on standard mono benchmarks.
  - Best choice when latency or model size matters; good fit for brass since
    the CQT representation handles vibrato modulation well.
  - https://arxiv.org/abs/2309.02265 / https://github.com/SonyCSLParis/pesto

- **SPICE** (Google, 2019). Self-supervised pitch via relative-pitch loss.
  Competitive with CREPE despite no labels, but typically a hair behind on
  intricate sources. https://arxiv.org/pdf/1910.11664

- **pYIN / pYAAPT / SWIPE** (classical). pYIN ~91% RPA on iKala (vs CREPE
  90.5%); robust on vocals but more octave-error-prone on instruments with
  weak fundamental like brass. Lower noise robustness than CREPE.

- **SwiftF0** (Nieradzik, Aug 2025). New SOTA on noise-robust mono pitch.
  91.80% harmonic mean at 10 dB SNR, ~12 pp better than CREPE under noise,
  95k params, 42x faster than CREPE on CPU. Useful if separation leakage is
  bad. https://arxiv.org/abs/2508.18440

- **PENN, DA-TC** (recent). DA-TC reports +1.15% RPA over pYIN, +0.65% over
  SWIPE, +4.05% over SPICE; incremental improvements, no jazz-specific eval.

### Note-level (pitch + onset + offset, sometimes dynamics)

- **CREPE Notes** (Riley & Dixon, 2023). Post-processes CREPE F0 into MIDI
  notes using contour gradient + inverse confidence; falls back to madmom
  onset detector for repeated-pitch notes.
  - **90% no-offset F-measure on Filosax (jazz saxophone)** per the GitHub
    README; the paper reports the SOTA result on that benchmark.
  - 74% on Irish flute (ITM GT Flute 99), 72% on FiloBass double bass; both
    +7-10 pp over Basic Pitch.
  - **No trumpet evaluation published.** Saxophone is the closest analog
    (similar register, similar vibrato/articulation idioms, also monophonic).
  - https://arxiv.org/abs/2311.08884 / https://github.com/xavriley/crepe_notes

- **Basic Pitch** (Spotify, ICASSP 2022). Lightweight CNN, polyphonic-capable
  with pitch-bend detection. Free, fast, widely deployed.
  - Vocal note F1 (no offset) ~52% on Molina; GuitarSet F1 ~79%.
  - Not trained or benchmarked on brass; user reports show it works on solo
    trumpet but with frequent false notes from vibrato and missed transients
    in fast passages. Pitch-bend detection helps with scoops but only
    coarsely.
  - https://huggingface.co/spotify/basic-pitch

- **MT3 / YourMT3+** (Google / mimbres, 2022 / 2024). Multi-instrument
  transformer transcribers; URMP includes trumpet. Onset-Offset F1 gain of
  263% on URMP for one mixture-formulation variant. Optimised for ensemble
  rather than expressive solo lines.
  - https://arxiv.org/abs/2407.04822

- **Saxophone Transcription Pipeline** (Riley/Foster/Dixon, arXiv 2405.16687,
  2024). Audio-to-score for the Charlie Parker Omnibook: source separation
  -> solo-sax MIDI transcription -> MIDI-to-score quantization.
  - Strong on Filosax-style data; "considerably lower" results on the
    out-of-distribution Omnibook (real Bird recordings). This is the most
    directly relevant published pipeline - it explicitly admits jazz mono
    transcription is not solved.
  - https://arxiv.org/abs/2405.16687 / https://aim-qmul.github.io/SaxTranscriptionPipeline/

### Commercial

- **Klangio Wind2Notes**. Marketed for trumpet, sax, clarinet, trombone,
  flute, tuba. Trustpilot reviews are mixed: users report inaccurate notes,
  wrong chords, and clef issues; accuracy drops on poor audio or complex
  parts. Free 20-second demo lets you sanity-check on a real solo. No
  published F1 numbers. https://klang.io/wind2notes/

- **Melodyne / NeuralNote**. Melodyne handles solo monophonic conversion
  well for clean inputs; vibrato and scoops are usually preserved as pitch
  curves but require manual cleanup to discretize. NeuralNote is essentially
  Basic Pitch in a plugin shell.

## Brass / Jazz Challenges

### Vibrato (4-7 Hz, ~50 cent depth typical)

Real trumpet vibrato modulates pitch, intensity, and timbre simultaneously,
and vibrato rate/depth drift over the duration of a note. At ~50 cents
depth, vibrato can cross the boundary between two semitones; naive note
segmenters that quantize each frame will emit chains of alternating
neighbour-tone notes. CREPE Notes uses pitch-contour gradient + confidence,
which damps slow modulation, but fast/wide vibrato can still trigger false
onsets. PESTO's CQT input is more vibrato-tolerant than waveform models.
Mitigation: smoothing over a 100-200 ms window, then median-pitch-per-note.

### Scoops, smears, doits, falls

These are intentional pitch trajectories from below/above the target.
Pitch-bend-aware models (Basic Pitch, Melodyne) capture them as continuous
bends, but discretizing them into a single note vs a pair of notes is a
musical judgement call no current model handles reliably. Expect a scoop to
be transcribed as either: (a) one note at the target pitch with a pitch
bend prefix (good), (b) two notes - the start pitch and the target (bad,
introduces a phantom passing tone), or (c) a glissando spanning every
chromatic step in between (worst, only with naive frame-quantization).
CREPE Notes' gradient-based onset detection should generally pick (a) or
(b); Basic Pitch + pitch-bend often picks (a); Klangio anecdotally produces
(b) on smears.

### Half-valve effects

Pitch is microtonal and unstable - the timbre also shifts. Pitch trackers
will return some F0 in the gap between semitones, and segmenters will
either drop the frames (low confidence) or quantize to whatever bin is
closest. In practice these become "missing" notes or blurred pitches; this
is a small fraction of jazz solos but a real issue in players like Miles or
Lee Morgan.

### Mute changes

A mute (Harmon, cup, plunger) is a timbral change that doesn't directly
affect F0 but does shift harmonic energy distribution. Models trained
without mutes can lose tracking confidence transiently. Most pitch trackers
recover quickly; note-level models may insert spurious onsets at the mute
transition.

### Articulation: legato/staccato/accents/ghosting

Ghost notes (very low velocity, partially articulated) are routinely
missed - they're below the onset detector's energy threshold. Legato slurs
between two pitches with no re-articulation can be merged into one note (a
miss), especially if the second note is close in pitch. Tongued repeated
notes at the same pitch require an onset detector (not pitch change) to
separate them - CREPE Notes explicitly falls back to madmom for this case,
which is a known weak spot.

### Bebop fast lines

16th notes at 240 BPM = ~16 notes/sec, ~62 ms per note. CREPE's standard
hop is 10 ms, so frame-rate is fine, but onset detectors typically require
30-50 ms to localize and ~50 ms minimum-note-length thresholds are common
defaults - meaning notes can be merged or temporally offset. The Omnibook
result ("considerably lower" than Filosax) is the canonical evidence that
fast bebop lines break current tools. Expect note F1 to drop 10-20 pp on
sustained fast 16th-note passages.

### Wide range / register

Trumpet sounds from ~F#3 (concert) up to C6 and beyond. Low-register
trumpet is at risk of octave errors because the brass spectrum is missing
or weak at the fundamental and pitch trackers can lock onto the second
harmonic. CREPE is generally robust here but pYIN is not. High-register
playing (above the staff) has fewer harmonics in band and stronger
inharmonicity, but still tracks reliably.

### Swing eighth-note timing

Onset detectors don't care about swing - they detect transients. The
challenge is downstream **quantization** to a metric grid, not the onset
detection itself. Outputs from these pipelines are timestamped onsets;
swung eighths just give triplet-feel timestamps. Quantization to score
notation is a separate (hard) problem handled in the Omnibook pipeline's
MIDI-to-score stage, not the pitch-tracking stage.

### Chromatic passing tones, blue notes

These are real notes that should be transcribed. Risk is that very fast
chromatic passing tones (1/16 or 1/32 at fast tempo) are below the note
duration threshold and get merged. Blue notes that are intentionally bent
~25-50 cents flat are at risk of being transcribed as the unbent neighbour
or as a pitch-bend on the unbent note - depends on the player and the
model's quantization strategy.

## Datasets

- **URMP** (Univ. of Rochester). Multi-instrument classical, includes
  trumpet, horn, trombone with separate stems and aligned MIDI ground truth.
  Used as the standard multi-instrument transcription benchmark (MT3,
  YourMT3+). Trumpet parts are classical, not jazz - useful for pitch
  accuracy but not for jazz idiom validation.
  https://labsites.rochester.edu/air/projects/URMP.html

- **MedleyDB / MDB-stem-synth**. Has horn / brass examples; CREPE's
  per-instrument numbers come from here. Mostly pop/rock context.

- **Filosax** (Foster & Dixon, ISMIR 2021). 48 multitrack jazz recordings
  with 5 sax players, ~24 hours total, with note-event and score-level
  annotations. Saxophone, not trumpet, but the closest jazz proxy and the
  standard jazz-mono benchmark.
  https://dave-foster.github.io/filosax/

- **Charlie Parker Omnibook + audio pairs** (Riley et al. 2024). Score-audio
  pairs with MIDI alignments for 60 Bird solos. Out-of-distribution
  benchmark for real bebop. Sax, not trumpet.
  https://aim-qmul.github.io/SaxTranscriptionPipeline/

- **MAESTRO** is piano-only (irrelevant). **GuitarSet** is guitar-only
  (irrelevant).

- **No public jazz-trumpet-specific dataset** with note-level annotations
  exists at the time of writing. This is the single biggest gap.

## Hugging Face / GitHub landscape

- Searches for "trumpet transcription" and "brass transcription" return no
  trumpet-specific models on Hugging Face. The closest available models are
  instrument-agnostic (Basic Pitch, YourMT3+) or sax-trained (CREPE Notes
  weights, sax pipeline checkpoints).
- `xavriley/crepe_notes` - CREPE Notes implementation, monophonic, the most
  directly applicable open repo.
- `SonyCSLParis/pesto` - PESTO pitch tracker.
- `marl/crepe` - CREPE.
- `spotify/basic-pitch` - Basic Pitch.
- `mimbres/YourMT3` - multi-instrument transcriber (URMP-trained, includes
  trumpet class).
- `lars76/swift-f0` and `lars76/pitch-benchmark` - SwiftF0 and a benchmark
  suite worth running on jazz trumpet stems before committing to a tracker.

## Verdict

**Realistic note-F1 on a clean separated trumpet jazz solo: 70-85% no-offset.**
Best estimate is ~80% if the recording is clean and the player's vibrato is
moderate; mid-70s on faster bebop or heavy expressive ornaments; potentially
90%+ on simple ballad lines. With offsets included, drop another 10-15 pp.
This range is extrapolated from CREPE Notes on Filosax (~82-90%), Basic
Pitch on monophonic vocals (~52%) and GuitarSet (~79%), and the Omnibook
"considerably lower" finding for OOD bebop. Trumpet vs sax is roughly a
wash for pitch-tracking purposes - similar registers, similar vibrato
characteristics, sax slightly easier because of stronger fundamentals.

**Recommended stack**: CREPE (or PESTO if latency matters) for F0, then
CREPE Notes for segmentation. Add custom post-processing:

1. Smooth pitch contour over 100-150 ms before segmentation to suppress
   vibrato.
2. Detect rising/falling pitch ramps faster than ~200 ms and merge them as
   scoop/fall ornaments on the next/previous note rather than emitting them
   as discrete notes.
3. Suppress notes shorter than ~50 ms unless preceded by a strong onset
   (defends against vibrato splits and chromatic confusion).
4. For separation leakage robustness, evaluate SwiftF0 in parallel - it is
   substantially more noise-robust than CREPE.

**Dominant failure modes, ranked**:

1. **Spurious note splits from vibrato and scoops** (most common; 50%+ of
   errors on expressive playing).
2. **Missed/merged notes in fast bebop lines** (note duration shorter than
   onset detector resolution; 16th notes at 240 BPM are right at the limit).
3. **Ghost notes and ultra-soft articulations missed** (fall below onset
   threshold).
4. **Octave errors in low register** (less common with CREPE/PESTO than
   pYIN, but real for low-register trumpet where the fundamental is weak).
5. **Half-valve passages dropped or pitch-blurred** (rare overall but
   uncorrectable).

**Will scoops/smears get transcribed as multiple notes?** Often yes with
naive setups, especially Klangio and naive Basic Pitch usage. CREPE Notes
and pitch-bend-aware Basic Pitch handle the common case (single scoop to
target) reasonably; long smears across multiple semitones will produce
spurious passing tones unless explicitly post-processed. This is the
single most important problem to solve in a custom post-processing step
for a jazz transcription product.

## Sources

- CREPE paper: https://arxiv.org/abs/1802.06182
- CREPE GitHub: https://github.com/marl/crepe
- CREPE Notes paper: https://arxiv.org/abs/2311.08884
- CREPE Notes GitHub: https://github.com/xavriley/crepe_notes
- PESTO paper: https://arxiv.org/abs/2309.02265
- PESTO GitHub: https://github.com/SonyCSLParis/pesto
- SPICE paper: https://arxiv.org/pdf/1910.11664
- SwiftF0 paper: https://arxiv.org/abs/2508.18440
- SwiftF0 GitHub: https://github.com/lars76/swift-f0
- Pitch benchmark suite: https://github.com/lars76/pitch-benchmark
- Basic Pitch (Spotify): https://huggingface.co/spotify/basic-pitch
- Basic Pitch GitHub: https://github.com/spotify/basic-pitch
- YourMT3+ paper: https://arxiv.org/abs/2407.04822
- YourMT3 GitHub: https://github.com/mimbres/YourMT3
- Filosax dataset: https://dave-foster.github.io/filosax/
- Filosax paper: https://www.eecs.qmul.ac.uk/~simond/pub/2021/FosterDixon-Filosax-ISMIR2021.pdf
- Charlie Parker Omnibook pipeline: https://arxiv.org/abs/2405.16687
- Sax Transcription Pipeline site: https://aim-qmul.github.io/SaxTranscriptionPipeline/
- URMP dataset: https://labsites.rochester.edu/air/projects/URMP.html
- Klangio Wind2Notes: https://klang.io/wind2notes/
- Klangio reviews (Trustpilot): https://www.trustpilot.com/review/klang.io
- ITG vibrato presentation: https://www.trumpetguild.org/images/pdf/ITG-NonProCommittee/VibratoPresentation.pdf
- Trumpet effects (half-valve, scoops, doits): https://themoderntrumpet.com/2020/12/21/half-valve/
- AMT survey 2024: https://arxiv.org/html/2406.15249v1
