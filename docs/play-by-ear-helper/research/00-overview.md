# Play by Ear Helper — Research Overview

**Date:** 2026-05-02
**Source files:** [`research/01`–`research/18`](./research/)

---

## TL;DR

A **per-instrument transcription tool** (user picks the instrument, we isolate it from the mix and transcribe just that stem) is **technically feasible and cheap to build in 2026** — but full "song-from-the-radio → multi-staff orchestral score" is not. The economics are remarkably good (≈ **$0.007/song** on serverless GPU with an open-source pipeline, vs. $2–4/song on commercial APIs), and there is a **clear unmet market gap**: no competitor lets the user choose which instruments to transcribe from a mix. The honest constraints are accuracy on real-world recordings (14–20 F1 point drop vs. clean benchmarks), an unavoidable 5–60 second offline wait (no real-time polyphonic AMT exists outside piano-only research), and a fuzzy copyright posture that requires a personal-use framing and a lawyer review before launch.

---

## Headline Findings (most significant first)

### 1. Feasibility — yes, with realistic scope

- **Solo piano is essentially solved**: hFT-Transformer hits **96.72% onset F1** on MAESTRO; ByteDance and Transkun V2 are pip-install-and-go.
- **Multi-instrument transcription** sits around **0.83 multi-F1 on Slakh2100** (synthesized) but **drops 14–20 F1 points on real-world recordings** — sound/timbre and genre shift are the dominant failure modes.
- **The viable architecture is a separation-first pipeline**: Demucs isolates a stem → per-stem AMT (Basic Pitch / Transkun / MT3) → quantize + spell → MusicXML. Trying to transcribe a full mix into a multi-staff score is **not production-quality**.
- The MusicRadar review of Songscription (2025) bluntly concludes humans will keep doing serious transcription "for the foreseeable future." Treat AI output as a **starting draft + editor**, not a final product.

→ See `01-polyphonic-transcription-feasibility.md`, `15-limitations-and-hard-cases.md`.

### 2. Cost is extremely favorable

- **Open-source MVP pipeline** (Demucs v4 + Basic Pitch) on Modal L4 / RunPod Serverless 4090: **~$0.007–0.010 per 3-minute song**.
- **Commercial APIs** (Klangio, AudioShake): **~$0.50–4.00 per song** — 50–500× more expensive.
- 100/mo: free-tier credits cover everything. 10,000/mo: ~$75–100/mo on serverless, ~$300/mo on reserved RunPod.
- **Cloudflare R2** for audio storage (zero egress) beats S3 by orders of magnitude at any meaningful scale.
- Self-hosted 4090 hardware payback vs. cloud rental: **3–9 months**, but only operationally sane above **~10,000 songs/day**.
- Replicate is the most expensive serverless option (~$0.056/song). Banana is dead (sunset 2024).

→ See `08-cost-analysis.md`, `18-gpu-economics.md`.

### 3. Latency — offline only; near-real-time is research, not product

- 3-minute song end-to-end: **~5–60 seconds** depending on hardware.
  - Basic Pitch: faster than real-time on modern CPU (10–30 s).
  - Demucs htdemucs: ~12 s on M4 Max, ~5 s on RTX 3090 with TensorRT.
- **Streaming polyphonic AMT does not exist** at general-purpose quality. Mobile-AMT and the 2025 streaming piano papers reach 128–320 ms latency but are **piano-only**.
- Recommended UX: **progress bar with partial-result reveal**, not a real-time scrolling staff. A monophonic "live pitch" preview (CREPE-tiny / pYIN via Web Audio) is feasible if the user wants to hum/sing.

→ See `09-latency-realtime-feasibility.md`.

### 4. The clearest market gap

No existing competitor (AnthemScore, Klangio/Melody Scanner, ScoreCloud, Songscription, Chordify) lets the user pick **which instruments** to transcribe from a mixed recording. They either dump everything onto one staff or transcribe a single solo instrument poorly. User reviews consistently complain about "mishmash of notes," paywalled basic playback, and read-only output.

**Recommended positioning:** "choose your instruments" + per-stem separation + clean engraving + inline editor with audio scrubbing, in a **$5–10/mo band** (below ScoreCloud, above Chordify).

→ See `07-competitor-products.md`, `17-ux-patterns.md`.

---

## Recommended MVP Pipeline & Stack

```
┌─ Web (Next.js 15 PWA) ──────────────────────────────────────────┐
│   MediaRecorder (Opus/WebM, AAC for iOS)                        │
│   OpenSheetMusicDisplay (MusicXML rendering on top of VexFlow)  │
└────────────────────────────┬────────────────────────────────────┘
                             │  upload + SSE for progress
┌────────────────────────────▼────────────────────────────────────┐
│   FastAPI (Python) on Fly.io / Render                           │
│   • Auth: Clerk                                                 │
│   • Audio in Cloudflare R2  • Metadata in Neon Postgres         │
└────────────────────────────┬────────────────────────────────────┘
                             │  job dispatch
┌────────────────────────────▼────────────────────────────────────┐
│   Modal serverless (L4 GPU, scale-to-zero)                      │
│   ffmpeg → Demucs (v4 htdemucs) → Basic Pitch / Transkun        │
│         → madmom/Beat This! (beat+downbeat)                     │
│         → key + chord (Essentia / BTC) → music21 → MusicXML     │
└─────────────────────────────────────────────────────────────────┘
```

**Why these choices:**
- **Web-first, not native** — sheet-music rendering ecosystem is JS-native; one codebase ships faster.
- **Modal over Celery/Redis** — keep MVP free of worker fleet ops.
- **Cloudflare R2 over S3** — zero egress, huge cost difference for media payloads.
- **Basic Pitch as default** — Apache-2.0, ~17K params, runs in-browser as a fallback, instrument-agnostic. **Transkun V2** for piano-tagged input (~0.95 note-with-offset F1).
- **Demucs htdemucs** for separation (MIT, 9.2 dB SDR). Fall back to LALAL.AI commercial API for hard guitar/piano cases (acoustic vs electric guitar split is a known weakness).

→ See `02-open-source-ai-models.md`, `04-source-separation-tools.md`, `05-sheet-music-rendering.md`, `14-tech-stack-architecture.md`.

---

## Critical "Don't Step Here" List

| Trap | Source |
|---|---|
| **YourMT3+** is GPL-3.0 — copyleft will infect a closed-source product. | `02-open-source-ai-models.md` |
| **madmom** is BSD code but its CRNN model weights are CC-BY-NC-SA — non-commercial only. Use Beat This! (ISMIR 2024) instead. | `10-tempo-beat-key-chord-detection.md` |
| **Open-Unmix UMXL weights** are non-commercial. | `04-source-separation-tools.md` |
| **MuseScore / webmscore** is GPLv3 — would force the whole client GPL. Use OSMD or Verovio. | `05-sheet-music-rendering.md` |
| Browser `getUserMedia({audio:true})` defaults enable `echoCancellation`/`noiseSuppression`/`autoGainControl` — these **degrade music transcription**. Pass explicit `false`. | `11-audio-recording-stack.md` |
| iOS Safari emits MP4/AAC, not WebM, has 44.1 vs 48 kHz mismatch, and 2.5 ms forced frame duration — handle server-side via ffmpeg. | `11-audio-recording-stack.md` |
| Spleeter is ~3 dB behind SOTA and unmaintained — don't use for new work. | `04-source-separation-tools.md` |
| Magenta Onsets and Frames is archived — points to MT3. | `02-open-source-ai-models.md` |
| Transcription output is almost certainly a derivative work; fair use is weak for the operator. **Lawyer review before launch.** | `12-legal-copyright.md` |
| Replicate's per-second pricing makes it 5–8× more expensive than Modal/RunPod for this workload. | `18-gpu-economics.md` |

---

## Topic-by-Topic Summary

### 01 · Polyphonic Transcription Feasibility
Solo-piano AMT is solved (~96.7% F1). Slakh2100 multi-instrument synthesized: ~0.83 F1. Real-world drop: 14–20 F1 from sound/timbre, 14 F1 from genre. **Verdict:** per-stem pipeline yes, full-mix-to-orchestral-score no.

### 02 · Open-Source AI Models
- **Spotify Basic Pitch** — Apache-2.0, ~5k stars, instrument-agnostic, runs in-browser. Best general MVP choice.
- **Transkun V2** — MIT, MAESTRO note-with-offset F1 ~0.95. Best piano choice.
- **ByteDance piano_transcription_inference** — Apache-2.0, frame AP ~0.93, includes pedal events.
- **Google MT3** — Apache-2.0, multi-instrument, transformer.
- **YourMT3+** — GPL-3.0 (license trap), MLSP 2024, beats MT3 but partially overfit to Slakh.
- Magenta Onsets and Frames archived; Omnizart stale; Meta has no AMT model.

### 03 · Commercial Transcription APIs
- **Klangio** — only commercial API returning MIDI/MusicXML/PDF/GP5 directly from audio across multiple instruments. Consumer $8.49–19.99/mo; API tier sales-contact.
- **Music.ai (Moises B2B)** — granular per-minute pricing ($0.10/min stems, $0.04/min chords).
- **LALAL.AI Pro** — $15/mo, 250 min, includes API.
- **AudioShake** — ~$1/min retail, strong stem separation.
- AnthemScore CLI is a viable self-hosted fallback.

### 04 · Source Separation
- **Demucs v4 htdemucs_ft** — MIT, 9.2 dB SDR, ~30× realtime on consumer hardware. Default.
- **BS-RoFormer / Mel-RoFormer** — community SOTA at ~9.9 dB.
- Piano/guitar separation is the hardest case — Demucs's `htdemucs_6s` warns piano quality is poor.
- **LALAL.AI Perseus** does 10 stems (acoustic vs electric guitar split) — paid fallback for hard cases.
- Spleeter and Open-Unmix are obsolete or license-trapped.

### 05 · Sheet Music Rendering
- **OpenSheetMusicDisplay (OSMD)** — MIT, MusicXML on top of VexFlow. Default.
- **Verovio** — LGPL, MEI-native with on-the-fly conversion, runs as WASM, includes MIDI export.
- **VexFlow** — solid low-level engine but no MusicXML import.
- MIDI → MusicXML: use a Python microservice with `music21` + `pretty_midi`.
- Avoid MuseScore/webmscore (GPL v3).

### 06 · Music Format Standards
- **MusicXML 4.0** — only realistic interchange target (270+ programs).
- **MNX** (W3C JSON successor) — still draft as of March 2026, watch but don't adopt.
- Hard work is **inferring** key/time sig/beaming/voicing/staff splits during MIDI→MusicXML conversion. SOTA transformer models hit ~83% note-value accuracy on ASAP-class material.
- Internal representation: custom JSON NoteSequence; export via music21.

### 07 · Competitor Products
- **AnthemScore** — desktop, $32–107 one-time, only one to expose AI uncertainty as a UI control.
- **Klangio family** (Melody Scanner, Piano2Notes, Transcription Studio).
- **ScoreCloud** — freemium up to $19.99/mo.
- **Songscription** (2025, Reach Capital backed) — $9.99–29.99/mo.
- **Chordify** — chord-only ($3.49/mo, has actual publisher licenses).
- Riffstation defunct. Logic Pro Flex Pitch is monophonic only.
- **Gap:** nobody lets the user pick instruments from a mix.

### 08 · Cost Analysis
- Open-source pipeline: **$0.007–0.010 per 3-min song**.
- Commercial APIs: **$0.50–4.00 per song**.
- 10,000 songs/mo: ~$75–100/mo on Modal serverless, vs. $1,000–20,000/mo on commercial APIs.
- Cloudflare R2 for storage (zero egress).

### 09 · Latency / Real-Time Feasibility
- Offline 3-min song: **5–60 seconds**.
- True real-time polyphonic AMT does not exist outside piano-only research (Mobile-AMT, 2025 streaming piano papers, 128–320 ms).
- Recommended UX: progress bar + partial reveal. Optional monophonic live-pitch preview via CREPE-tiny / pYIN.

### 10 · Tempo / Beat / Key / Chord
- **Beat / downbeat:** Beat This! (ISMIR 2024) > madmom (license issue).
- **Chords:** BTC (Bi-directional Transformer, ISMIR 2019) or Chordino (GPL).
- **Key:** Essentia `KeyExtractor` or Krumhansl-Schmuckler over librosa CQT chroma.
- **All-In-One Music Structure Analyzer** returns tempo + beats + downbeats + verse/chorus from one model.

### 11 · Audio Recording Stack
- Web: MediaRecorder (Opus/WebM; AAC/MP4 for iOS Safari).
- Mobile: **expo-audio** (expo-av deprecated in SDK 53+).
- Server-side ffmpeg: mono, 16 kHz, two-pass loudnorm, silence trim.
- **Disable** echoCancellation/noiseSuppression/autoGainControl for music.
- iOS Safari has multiple gotchas — handle them server-side.

### 12 · Legal / Copyright
- Output is almost certainly a derivative work; fair use weak for the operator.
- No mechanical-license analogue for sheet music.
- Existing competitors survive with personal-use ToS framing and DMCA takedown response. Only Chordify has actual publisher licenses.
- **Required posture:** personal-use-only product, no sharing/distribution feature, ephemeral audio (delete after transcription), DMCA agent registration, GDPR-compliant privacy notice, no training on user audio, lawyer review before launch.

### 13 · Audio Quality Requirements
- Studio + ≥128 kbps streaming = published-benchmark accuracy.
- Phone-mic ambient capture: 14–20+ F1 point drop. Mobile-AMT (EUSIPCO 2024) and ARIA-AMT show in-the-wild augmentation can recover ~14 F1.
- Resampler quality matters more than the exact rate.
- **Demucs is the single most useful preprocessing step** (acts as separator + denoiser).
- User guidance: 12–36 in mic distance, off-axis, quiet room, airplane mode, ≥128 kbps.

### 14 · Tech Stack & Architecture
- Next.js 15 PWA → FastAPI → Modal serverless GPU.
- Cloudflare R2 (storage) + Neon Postgres (metadata) + Clerk (auth) + Sentry (errors).
- SSE for progress streaming.
- Web-first beats React Native / Flutter for a small team because the sheet-music rendering ecosystem is JS-native.

### 15 · Limitations & Hard Cases
Hard genres: heavily distorted electric guitar, dense classical/orchestral, jazz with extended chords, electronic/synth, dense vocal harmonies. Hard musical content: trills/ornaments, fast passages, octave errors, microtonal/non-Western tunings. User complaints across AnthemScore, Klangio, Songscription consistently call out polyphonic accuracy and the manual cleanup burden.

### 16 · Instrument Classification
- Open-source SOTA (clip-level multi-label): PANNs / AST / PaSST on AudioSet (~0.44–0.50 mAP); music-specific OpenMIC-2018 fine-tunes (~0.70–0.80 mAP).
- **Most HF "instrument classification" checkpoints are misleading** — trained on isolated mono samples, fail on polyphonic music.
- **Don't put detection on the critical path.** Have the user pick the instrument; use detection only as a suggestion / mismatch warning.

### 17 · UX Patterns
- Single capture entry point with deferred config.
- AnthemScore's confidence visualization (white candidate notes + likelihood slider + spectrogram) is the strongest competitor differentiator — clone it.
- Soundslice's synced playhead + click-to-seek + smooth scroll is the playback model to copy.
- Free end-to-end demo on short clips, ungated PDF preview, inline editor with audio scrubbing.
- Avoid: paywalled basic playback (Chordify's mistake), microphone-input gated to paid tier (ScoreCloud's mistake).

### 18 · GPU Economics
- **RunPod Serverless 4090: ~$0.007/song** (cheapest).
- **Modal L4: ~$0.010/song** (best DX).
- Replicate ~$0.056/song — avoid for production.
- Serverless → reserved breakeven: ~2,800 transcriptions/day.
- Self-hosted hardware payback: 3–9 months, only sane above 10k/day.
- On-device (Core ML / TFLite) is realistic for Basic Pitch today; full Demucs on mid-tier mobile is ~12–18 months out.

---

## Open Questions for the Build Phase

1. **Instrument scope for v1**: piano-only is the safest accuracy story. Guitar/bass/vocals add user value but each adds a separation+transcription failure mode.
2. **Editing UX depth**: how much of AnthemScore's confidence-driven editor do we ship in v1?
3. **Free-tier limits**: clip length cap (30 s? 60 s?) and watermarked PDF, or full clips with rate limits?
4. **YouTube/Spotify URL ingest**: legally risky vs. explicit user upload — defer?
5. **Mobile**: PWA-only at launch, or React Native shell wrapping the web app for App Store presence?
6. **Lawyer review**: when, and what jurisdictions matter most (US? EU? both)?
