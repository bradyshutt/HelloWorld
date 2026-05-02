# Audio Quality Requirements & Robustness

## Summary
Modern AMT models (Onsets and Frames, MT3, Basic Pitch, ARIA-AMT) are trained almost entirely on clean, close-mic'd or DI sources (MAESTRO, MAPS, Slakh) and degrade meaningfully outside that distribution: published reports show note-level F1 drops of ~14 points from genre shift and ~20 points from sound/recording shift, and ambient or reverberant captures can lose another 10-20+ points without augmentation. Studio and lossless streaming audio (>=128 kbps MP3, FLAC, WAV) generally produce near-publication-quality transcriptions; 64-96 kbps lossy, phone-mic ambient recordings, and live/reverberant rooms are the most fragile. Preprocessing with Demucs (as a denoiser/source-separator) and matching the model's expected sample rate (16 kHz for Onsets and Frames, 22.05 kHz for Basic Pitch) recovers most of the gap; ARIA-AMT and Mobile-AMT show that training-time noise/RIR augmentation can add +14 F1 in-the-wild. For Play by Ear Helper, the highest-impact user guidance is: close mic placement (12-36 in / 30-90 cm, off-axis from hammers), quiet room (turn off HVAC, close windows), and at least 3-5 seconds of audio per phrase.

## Quality Tiers

### Tier 1 — Studio / lossless source files (best)
- WAV, FLAC, ALAC, or 256+ kbps lossy from a DAW/DI source.
- This matches the training distribution of MAESTRO/MAPS/Slakh exactly.
- Onsets and Frames hits ~95% note-with-offset F1 on MAESTRO test split; high-resolution piano transcription (Bytedance/Kong et al.) reaches ~96.7% note F1.
- Basic Pitch performs at near-published accuracy on monophonic and lightly polyphonic single-instrument tracks.
- **Recommendation:** pass through unchanged; only normalize loudness.

### Tier 2 — Streaming-quality audio (Spotify, MP3 128-320 kbps, AAC, Opus)
- Lossy compression mostly preserves pitch onset cues that AMT relies on, but introduces pre-echo and high-frequency smearing that hurts onset precision.
- Empirical AMT robustness studies (e.g., Edwards et al., "Improving Note Segmentation in AMT") explicitly evaluate at 64 kbps MP3 (LAME) as a worst case for "real-life" audio; degradation is small at >=128 kbps but visible at 64 kbps.
- Opus at 64-96 kbps preserves music quality better than AAC/MP3 at the same bitrate (Xiph multi-codec listening tests), so a phone using Opus for ambient capture loses less than one using a 64 kbps MP3.
- **Recommendation:** accept as-is at 128 kbps and above; warn users below ~96 kbps.

### Tier 3 — Phone recordings ("ambient capture")
- Built-in smartphone mics are tuned for voice (often AGC, noise gating, and HPF below ~80 Hz), which removes piano fundamentals in the bottom octave (A0=27.5 Hz, low-bass partials below 80 Hz).
- Mobile-AMT (EUSIPCO 2024, Yamaha) is the only published model purpose-built for this case: data augmentation specifically for "in-the-wild" recordings improves note F1 by **+14.3 points** vs the same architecture trained only on clean data.
- ARIA-AMT (Spangher et al.) augments training with 20 different room impulse responses, clapping/static noise, bandpass filtering, and ±15 cent detuning; it generalizes far better than Onsets and Frames to phone-quality input.
- Without those augmentations, expect note F1 to drop 20-40 points from a clean baseline on phone-mic captures of an acoustic piano in a typical room.
- **Recommendation:** preprocess with Demucs (denoising + source isolation) before transcription; prefer ARIA-AMT or a Mobile-AMT-class model.

### Tier 4 — Background noise (talking, HVAC, room reverb)
- "Towards Robust Transcription: Exploring Noise Injection Strategies" (arXiv 2410.14122) systematically tests white noise across SNRs and shows clean-trained models collapse at low SNR (<10 dB); noise-augmented training closes most of the gap.
- The "Data-Driven Analysis of Robust Automatic Piano Transcription" paper (arXiv 2402.01424) reports **+3.1 F1 OOD from pitch-shift augmentation and +2.8 F1 OOD from reverb augmentation** on top of standard pipelines.
- "Sound and Music Biases in Deep Music Transcription Models" (arXiv 2512.14602) reports **-20 note F1 from sound/recording shift, -14 from genre shift**.
- Talking voices in the same band as the instrument (~200-3000 Hz) cause spurious onset detections; constant fan noise mostly degrades frame-level pitch accuracy rather than onsets.
- **Recommendation:** RNNoise or Demucs as a denoiser pass; tell user to close the windows.

### Tier 5 — Live performance recordings
- Combination of room reverb, audience noise, and uneven mic placement.
- Long reverb tails (RT60 > 0.8 s) merge note offsets, hurting offset F1 most; onset F1 is more resilient.
- ARIA-AMT was specifically designed for diverse recording environments (cylinder recorder, vintage mic, concert hall RIRs) and is the most robust public model here.
- MT3 has been shown to do "zero-shot" transcription of YouTube audio, but with notable errors in instrument leakage and missed notes (MR-MT3 was proposed to mitigate this).
- **Recommendation:** Demucs to isolate the dominant instrument stem before transcription; warn the user about offset errors in long-reverb venues.

## Sample-rate sensitivity
| Model | Native input SR | Behavior on mismatch |
|---|---|---|
| Onsets and Frames | 16 kHz mel-spectrogram (229 bins, hop 512, FFT 2048) | Resamples internally; high frequencies above 8 kHz are lost (irrelevant for piano fundamentals up to ~4.2 kHz, but cymbals/sibilance gone). |
| Basic Pitch (Spotify) | 22.05 kHz mono | Auto-resamples any input; stereo down-mixed to mono at predict time. |
| MT3 / YourMT3+ | 16 kHz | Resamples internally. |
| ARIA-AMT | log-mel from variable input (commonly 16 kHz) | Robust to resampling artifacts due to augmentation. |
| Bytedance high-resolution piano | 16 kHz | Same. |

Practical implications:
- Feeding 44.1 kHz audio to a 16 kHz model is fine if you use a high-quality resampler (e.g., librosa's `kaiser_best`, `soxr_hq`); naive sample dropping introduces aliasing that degrades onset F1.
- Feeding 8 kHz telephone-quality audio is **bad**: the Nyquist limit of 4 kHz cuts piano harmonics from C5 (~523 Hz) upward partials, hurting timbre cues.
- Sample-rate mismatch is rarely the primary failure mode — bandwidth limiting and noise dominate in practice.

## Mono vs stereo
- All major AMT models down-mix to mono internally (Whisper does the same; Basic Pitch documents this explicitly).
- For Play by Ear Helper, accepting stereo input and down-mixing inside the pipeline is fine; there is no transcription benefit from preserving stereo for a single-instrument capture.
- Exception: stereo can help **upstream** of transcription if you use Demucs first — Demucs uses stereo cues for separation quality. So preserve stereo until after the source-separation step.

## Compression artifacts
| Codec / bitrate | Expected impact on AMT |
|---|---|
| WAV / FLAC / ALAC | None |
| MP3 320 kbps / 256 kbps | Negligible |
| MP3 128 kbps | Minor onset jitter, negligible note F1 hit |
| MP3 64 kbps (LAME) | Used as a worst-case real-world test; measurable F1 drop, especially on offsets and dense polyphony |
| AAC 128 kbps | Comparable to MP3 192-256 kbps |
| Opus 64-96 kbps | Best-in-class for music at low bitrate; minimal AMT impact |
| HE-AAC at 32 kbps | Significant transient smearing — avoid |

## Preprocessing Recommendations
Recommended pipeline for Play by Ear Helper inputs:
1. **Decode and resample** to the target model's native rate (22.05 kHz for Basic Pitch, 16 kHz for Onsets and Frames / MT3 / ARIA-AMT) using a high-quality polyphase resampler (`soxr_hq` or `kaiser_best`).
2. **Loudness normalization** (EBU R128 / ReplayGain target around -14 to -16 LUFS, or simple peak-normalize to -1 dBFS). Don't apply aggressive limiting — it changes onset transients.
3. **Optional denoising** for ambient/phone captures:
   - **Demucs** (`--two-stems=vocals` to discard speech, or run the 4-stem model and keep `other`/`piano`-like stem) — doubles as both denoiser and source separator, removing room reverb and background talking surprisingly well; SDR ~9.0 dB on MUSDB HQ.
   - **RNNoise** for stationary noise (HVAC, computer fans, hiss). It is per-band so it does not introduce musical-noise tones, but it is tuned for speech bands and may attenuate quiet musical content; safer for spoken-word or narration overlays.
   - **Facebook denoiser** (Demucs-architecture causal speech enhancer) — removes stationary and non-stationary noise plus room reverb, but again is speech-tuned.
4. **Skip aggressive AGC and EQ** — both reshape onset envelopes that AMT depends on. If you must AGC, use slow time constants (>500 ms attack/release).
5. **Match training distribution where possible**: if your model expects piano, use Demucs to extract the piano-like stem first instead of feeding a full mix.
6. **Minimum clip length:** the model itself has no hard minimum, but practically you want **>= 3-5 seconds** for the Play by Ear Helper output to be musically meaningful (one phrase). Whisper-style models have a 30-second window; Onsets and Frames operates frame-by-frame so any length works but very short clips give the rhythm-quantizer too little context to estimate tempo and meter.

## User Guidance

### Mic placement (smartphone capturing acoustic piano)
- **Distance:** 12-36 in (30-90 cm) from the instrument is the sweet spot. Closer than ~12 in over-emphasizes the nearest strings and picks up hammer/key noise; farther than ~6 ft adds room reverb that smears note offsets.
- **Position:** for a grand, half-stick or full-stick open, mic placed over the curve of the body (not directly over hammers). For an upright, point at the soundboard from a few feet back, or above the open top.
- **Off-axis:** point the mic slightly away from any hard, reflective wall to reduce comb filtering.
- **Phone in airplane mode** to suppress notification beeps and cell-radio interference.
- **External mic strongly recommended** if available (e.g., Shure MV88 lightning mic; any USB-C cardioid). Built-in phone mics have aggressive HPF that removes the piano's lowest octave.

### Room conditions
- Close windows; turn off HVAC, fans, dehumidifiers, fluorescent ballasts.
- Avoid bare echoey rooms (kitchens, bathrooms); a furnished living room is better.
- No talking during the recording — even quiet speech triggers spurious onsets.

### Format and length
- Prefer WAV or AAC/Opus at >=128 kbps over MP3. MP3 below 128 kbps is the weakest input.
- 16-bit / 44.1 or 48 kHz is more than sufficient; 24-bit gives no AMT benefit.
- Keep at least one full musical phrase (~5-10 seconds, 4-8 bars) in a single take so the downstream meter/tempo estimator has enough material.

### When to expect failure
- Multiple instruments playing simultaneously with overlapping pitch ranges (string quartet, ensemble).
- Extreme reverb (cathedrals, large halls) without a close mic.
- Very low-quality lossy audio (sub-64 kbps MP3, AMR-NB voice-call recordings).
- Heavy effects (delay, chorus) that fool models into doubled notes — Basic Pitch is documented to hallucinate extra notes per delay tap.

## Sources
- ["A Data-Driven Analysis of Robust Automatic Piano Transcription" — Maman et al., arXiv 2402.01424](https://arxiv.org/html/2402.01424v1) — F1 gains from pitch-shift (+3.1) and reverb (+2.8) augmentation.
- ["Towards Robust Transcription: Exploring Noise Injection Strategies for Training Data Augmentation" — arXiv 2410.14122](https://arxiv.org/html/2410.14122) — SNR sensitivity study; noise-augmented training closes the gap at low SNR.
- ["Sound and Music Biases in Deep Music Transcription Models" — arXiv 2512.14602](https://arxiv.org/pdf/2512.14602) — Quantifies -20 F1 sound shift, -14 F1 genre shift.
- ["Mobile-AMT: Real-time Polyphonic Piano Transcription for In-the-Wild Recordings" — Kusaka et al., EUSIPCO 2024](https://eurasip.org/Proceedings/Eusipco/Eusipco2024/pdfs/0000036.pdf) — +14.3 F1 in-the-wild via augmentation; 82.9% compute reduction.
- ["Musically Aware Automatic Piano Transcription Using Synthetic Pretraining" (ARIA-AMT) — Spangher et al.](https://www.alexander-spangher.com/papers/aria_amt.pdf) — 20 RIRs, clapping/static, bandpass, ±15 cent detuning augmentations.
- [ARIA-AMT GitHub — EleutherAI](https://github.com/EleutherAI/aria-amt) — Implementation details.
- ["Onsets and Frames: Dual-Objective Piano Transcription" — Hawthorne et al., ISMIR 2018](https://arxiv.org/pdf/1710.11153) — 16 kHz mel-spectrogram input; 229 bins, hop 512, FFT 2048.
- [Onsets and Frames overview, Magenta](https://magenta.tensorflow.org/onsets-frames)
- ["MT3: Multi-Task Multitrack Music Transcription" — Gardner et al., ICLR 2022](https://arxiv.org/pdf/2111.03017) — Out-of-domain leave-one-dataset-out evaluation; YouTube zero-shot transcription.
- ["YourMT3+: Multi-instrument Music Transcription" — arXiv 2407.04822](https://arxiv.org/html/2407.04822v1) — Cross-dataset stem augmentation.
- ["MR-MT3: Memory Retaining Multi-Track Music Transcription" — arXiv 2403.10024](https://arxiv.org/html/2403.10024v1) — Mitigates instrument leakage that MT3 suffers from on noisy/live audio.
- ["High-resolution Piano Transcription with Pedals" — Kong et al., arXiv 2010.01815](https://arxiv.org/abs/2010.01815) — 16 kHz; Bytedance reference implementation.
- ["Improving Note Segmentation in Automatic Piano Music Transcription" — ISMIR 2017](https://archives.ismir.net/ismir2017/paper/000100.pdf) — 64 kbps MP3 LAME used as a real-world worst case.
- ["Meet Basic Pitch: Spotify's Open Source Audio-to-MIDI Converter" — Spotify Engineering, 2022](https://engineering.atspotify.com/2022/6/meet-basic-pitch)
- [Basic Pitch GitHub README](https://github.com/spotify/basic-pitch/blob/main/README.md) — 22.05 kHz resample, mono down-mix, any input length.
- [Basic Pitch About page](https://basicpitch.spotify.com/about) — Built on CREPE; documented limitations on delay/dense mixes.
- ["CREPE: A Convolutional Representation for Pitch Estimation" — Kim et al., ICASSP 2018](https://arxiv.org/abs/1802.06182) — Noise robustness, brown-noise weakness.
- [Demucs GitHub — facebookresearch](https://github.com/facebookresearch/demucs) — 9.0 dB SDR on MUSDB HQ; 4-stem and `--two-stems=vocals` modes.
- [Facebook denoiser (Demucs-architecture, real-time speech enhancement)](https://github.com/facebookresearch/denoiser) — Removes stationary/non-stationary noise plus room reverb.
- [RNNoise — xiph](https://github.com/xiph/rnnoise) and [RNNoise: Learning Noise Suppression — Valin](https://jmvalin.ca/demo/rnnoise/) — 22-band per-band gains, no musical-noise artifacts.
- [Opus codec comparison — Xiph](https://www.opus-codec.org/comparison/) — Listening tests showing Opus > AAC > MP3 at 64-96 kbps.
- [Whisper long-form transcription chunking](https://medium.com/@yoad/whisper-long-form-transcription-1924c94a9b86) — 30-second windowing.
- [Tommy's Piano Corner — Recording piano with smartphone (Shure MV88)](https://tommyspianocorner.com/record-your-piano-great-smartphone-microphone/) — Mic placement at half-stick, airplane mode tip.
- [Joff Winks — How to Record a Grand Piano](https://www.joffwinks.com/blog/how-to-record-a-grand-piano) — Distance and curve-of-body placement guidance.
- [Practical Music Production — How to Mic a Piano](https://www.practical-music-production.com/how-to-mic-a-piano-for-recording/) — 3:1 spaced-pair rule, distance trade-offs.
- [Rev — 8 Killer Tips for Audio Recording for Transcription](https://www.rev.com/blog/8-killer-tips-for-audio-transcription-and-recording) — Quiet room, no reflective surfaces.
- [Videomaker — Recording professional audio with a smartphone](https://www.videomaker.com/how-to-record-professional-level-audio-with-your-smartphone/) — External-mic recommendation, WAV/AIFF preference.
