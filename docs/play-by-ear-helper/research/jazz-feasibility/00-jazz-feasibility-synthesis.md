# Jazz Quartet Feasibility — Synthesis

**Use case under evaluation:**
> Record a 15-second clip of a jazz quartet (piano + drums + acoustic bass + trumpet). Ask the app: (a) what notes is the **trumpet** playing, or (b) what chords is the **piano** playing.

**Verdict in one sentence:** This is **technically possible but materially below what a jazz musician would consider "working well"** as of 2026 — expect ~50–70% note-F1 on the trumpet line and a chord output that mostly degrades to triads + dom7s. A useful product *is* buildable, but only if it is positioned as a **first-draft assistant with strong editing**, not "the app that gives you the notes."

---

## Headline Numbers

| Sub-task | Realistic accuracy on a real jazz-quartet recording (2026) | Confidence |
|---|---|---|
| Trumpet **separation** (SDR) from full mix | 5–7 dB commercial · 3–5 dB open-source | High |
| Frame-level **F0 pitch** on a separated trumpet stem | 90%+ at 10 cents | High |
| **Note-level F1** for trumpet (after separation + segmentation) | 50–75% (depends on style; bebop lines worst) | Medium |
| Direct multi-instrument AMT (YourMT3+, MT3) on trumpet | ~40–55% F1, with brass↔sax↔trombone label confusion | Medium |
| Chord recognition root accuracy on jazz mix | ~75–85% | Medium |
| Chord recognition full extended-chord-symbol accuracy | < 40–50% | High |
| End-to-end ACR (BTC/ChordFormer) on JAAH jazz | ~32–40% | High |
| Same models on Beatles/Isophonics pop | ~80% | High |

The 30–40 percentage-point gap between pop and jazz is not a margin-of-error problem — it is the entire ballgame for our use case.

---

## Why It's This Hard

1. **No jazz training data at scale.** The only large jazz-transcription corpora are PiJAMA (jazz piano, TISMIR 2024) and Jazz Trio Database / Charlie Parker Omnibook (sax/piano/drums). Slakh2100 — the dominant multi-instrument benchmark — is **synthesized General-MIDI renderings**, not real recordings, and contains no real jazz. The Dec 2025 "Sound and Music Biases" paper formally measures the gap: ~20 pp F1 drop on instrument shift, ~52 pp on far-OOD audio.
2. **Standard separators don't have a brass stem.** Demucs's 4-stem (vocals/drums/bass/other) lumps trumpet with piano in "other." The 6-stem `htdemucs_6s` adds piano + guitar but no horns, and even its piano stem has documented heavy artifacts. Open-source horn-targeted separators barely exist; the strongest are ZFTurbo's 53-stem BS-Roformer (community) and AudioShake's commercial wind-instrument stem.
3. **Jazz chord vocabulary exceeds every public model's output space.** Chord recognizers ship with vocabularies of ~24 (Maj/Min) to ~60 chords (Maj/Min/Maj7/Min7/Dom7). Jazz uses extended/altered/slash/quartal/rootless voicings (b9, #11, alt, m7b5, sus4(add9), etc.). The output literally cannot represent what the pianist played.
4. **Rootless voicings break root detection.** Bill Evans-style A/B voicings deliberately omit the root — the bass plays it. This means **separating piano alone** for chord recognition makes things *worse*, not better. NNLS-Chroma already uses a dedicated bass-chroma channel for this reason.
5. **Jazz performance idioms break note segmentation.** Trumpet vibrato (4–7 Hz, ~50 cent depth), scoops, smears, and half-valve effects cause CREPE-style frame-level pitch trackers to produce spurious note splits. Bebop 16ths at 240 BPM = 16 notes/sec, right at the temporal resolution limit of mainstream onset detectors.
6. **Every consumer tool fails on jazz today.** Empirical reports across AnthemScore, Klangio, ScoreCloud, Songscription, Chordify, and Moises converge on the same failure modes. AnthemScore's own vendor **recommends Transcribe! (a manual aid)** for jazz. The 2025 MusicRadar review of Songscription, after testing on Monk, was titled "Humans will be doing all the serious music transcription for the foreseeable future."
7. **Audio LLMs hallucinate notes.** Gemini 2.5/3 Pro and Music Flamingo (NVIDIA, Nov 2025) score well on music *understanding* (e.g., 92% key detection, 76.83 MMAU-Music) but are unreliable for per-note transcription — they confidently emit wrong pitches. GPT-4o frequently refuses pitch tasks. Claude has no native audio input.

---

## What's Actually Promising

- **Music Flamingo (NVIDIA, Nov 2025)** — open-weights 7B audio-LLM with chord-tracking RL rewards. Strongest single 2025 leap for the *chord half* of the problem.
- **PiJAMA + Jazz Trio Database** (both TISMIR 2024) — first large real-jazz corpora that make jazz-specific fine-tuning realistic.
- **Jointist** — joint separation + transcription, emits per-instrument MIDI from the mix without an explicit separation step.
- **YourMT3+** (MLSP 2024) — multi-track MIDI with task queries, the strongest off-the-shelf multi-instrument AMT for our scenario despite the 40–55% trumpet F1 ceiling.
- **CREPE / PESTO + CREPE Notes** — solid monophonic stack once a stem is separated. Filosax (jazz sax) reports note-F1 ~82–90% on this stack.
- **Hybrid pipeline (the recommended approach)** — drum-removed remix → YourMT3+/MT3 for multi-instrument scaffolding, AudioShake (or `htdemucs` other-stem + custom post-processing) for trumpet isolation, CREPE/PESTO + CREPE Notes for monophonic horn note segmentation, BTC/ChordFormer for chord *backbone*, then **rule-based (or LLM-based) chord-symbol inference** from the detected pitch set + bass (modeled on the MuseScore "Chord Identifier (Pop & Jazz)" plugin).
- **License opportunity:** PiJAMA + JAAH + URMP-brass + a frozen MERT/MusicFM encoder + small head ≈ a single-A100, days-of-work fine-tune that could meaningfully outperform off-the-shelf models on jazz piano/horns.

---

## Recommended Pipeline (Jazz-Aware v1)

```
Input: 15s jazz quartet clip
  │
  ├── Demucs htdemucs (4-stem)
  │     ├── drums      ──▶ (discarded for ACR; kept for tempo/beat via Beat This!)
  │     ├── bass       ──▶ pitch track (CREPE) → root candidates
  │     ├── vocals     ──▶ (usually empty in instrumental jazz)
  │     └── other      ──▶ remixed (drum-removed) audio
  │
  ├── Drum-removed audio
  │     │
  │     ├──▶ BTC / ChordFormer (end-to-end ACR)  ──▶ triad/dom7 backbone
  │     │
  │     └──▶ YourMT3+ / MT3                       ──▶ multi-instrument MIDI scaffold
  │             ├── trumpet track (raw, leaky, ~40-55% F1)
  │             └── piano track (notes + voicing)
  │
  ├── For "trumpet notes" task:
  │     other-stem ──▶ AudioShake wind stem
  │                ──▶ CREPE / PESTO (frame F0)
  │                ──▶ CREPE Notes (segment to notes)
  │                ──▶ ornament collapse: vibrato + scoops → pitch-bend marks
  │                ──▶ swing-aware quantization (madmom downbeats)
  │
  └── For "piano chords" task:
        piano voicing pitches (from YourMT3+) + bass root (from CREPE on bass stem)
                ──▶ chord-symbol inference (rule-based / LLM)
                ──▶ output extended chord symbol (e.g., Cm9, F7alt, Bb13#11)
                ──▶ confidence-tagged (so user knows where it guessed)
```

Key nuances:
- **Don't isolate the piano stem** for chord recognition — Demucs piano stems have artifacts and the bass is needed for rootless voicings. Drum-removed remix beats piano-stem-only.
- **Confidence visualization is required.** AnthemScore-style "candidate notes" + likelihood slider is the only honest UX given these accuracy numbers. Otherwise the user feels lied to.
- **15-second clips are fine** for tempo/beat/chord/separation/note-level pitch with dedicated MIR tools. Borderline for Krumhansl key estimation (use signature-of-fifths instead). Weak for unusual time signatures.

---

## Honest Product Verdict

For the literal stated use case — *"hear a jazz quartet, get the trumpet's notes or the piano's chords"* — the answer is:

- **Trumpet notes**: a v1 product can plausibly hit 50–70% note-F1 on swung melodic phrasing in a clean recording. **Bebop, fast 16ths, dense comping leakage, and ornaments will be visibly wrong** to a jazz musician. Not "wow," but useful as a starting draft.
- **Piano chords**: extended-chord output is **not solved** in 2026. Best realistic output is "triad + bass note + maybe 7th quality" for ~70% of bars, with the user expected to fill in tensions/alterations. This will feel like a downgrade to anyone who knows jazz harmony.

### What we should consider doing instead or alongside

1. **Re-scope to where the technology actually works.** Solo piano (~96% F1 on MAESTRO) is a real product. Vocal-and-guitar songs are tractable. Jazz quartet is an aspirational stretch goal, not a v1.
2. **Lean into the editing surface.** The honest jazz-product story is *"AI-assisted manual transcription"* — Transcribe!-style slow-down + loop + pitch-display, with the AI providing a draft to correct. Soundslice has shown this is a viable business; nobody else has put modern AMT into that workflow.
3. **Train one jazz-specific model.** PiJAMA + Jazz Trio Database + JAAH is enough fine-tuning data to meaningfully beat off-the-shelf models on jazz piano voicings. This is a genuine moat — no consumer tool has done it.
4. **Set explicit expectations in onboarding.** "Best on solo or duo, recorded close-miked, in a quiet room. Quartet and ensemble recordings are an experimental beta with significant errors." Honesty here will reduce support burden and improve retention.
5. **Make the chord output a chord *backbone*, not a chord *answer*.** Show the triad + dom7 backbone with high confidence; show possible extensions (b9, #11, alt) as low-confidence overlays the user toggles on/off. This converts a weakness into an interactive feature.

---

## What Would Need to Happen for This to "Just Work"

For full-band-jazz-mix → multi-staff lead-sheet to genuinely "just work," at least three of these would need to land:

1. A real-jazz-trained multi-instrument AMT (a jazz-tuned YourMT3+ on PiJAMA + Jazz Trio Database). **Plausible by 2027.**
2. A horn-aware separator with > 8 dB SDR on small jazz combos. **Currently research-grade; commercially via AudioShake-class tools.**
3. An extended-chord-symbol output layer trained on jazz harmonic vocabulary. **No public model; would need custom dataset construction.**
4. An audio LLM strong enough to do per-note transcription without hallucination. **Music Flamingo + Gemini 3 are improving fast but not there for pitch-precise output as of early 2026.**

None of these are blocked by physics — they're blocked by data and engineering work nobody has prioritized. A small focused team could plausibly build (1) and (3) in 6–9 months and have a real moat.

---

## Files in This Folder

| # | Topic | File |
|---|---|---|
| 01 | Trumpet/horn separation from jazz combo | `01-trumpet-separation.md` |
| 02 | Brass / trumpet monophonic pitch tracking | `02-trumpet-pitch-tracking.md` |
| 03 | Jazz chord recognition (extended chords) | `03-jazz-chord-recognition.md` |
| 04 | Multi-instrument AMT performance on jazz | `04-multi-instrument-amt-on-jazz.md` |
| 05 | Query-based / conditional separation | `05-query-conditional-separation.md` |
| 06 | Foundation audio models for transcription | `06-foundation-models-for-transcription.md` |
| 07 | Empirical product tests on jazz | `07-empirical-product-tests-on-jazz.md` |
| 08 | Recent 2024–2026 research | `08-recent-research-2024-2026.md` |
| 09 | Direct mix vs. separated piano stem ACR | `09-direct-vs-separated-chord-recognition.md` |
| 10 | Short clips and audio LLMs | `10-short-clip-and-audio-llms.md` |
