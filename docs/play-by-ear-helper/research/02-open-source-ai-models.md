# Open-Source AI Models for Music Transcription

## Summary
For Play by Ear Helper, the strongest MVP candidate is **Spotify Basic Pitch** (Apache-2.0, instrument-agnostic, polyphonic, ~17K parameters, runs on CPU in real-time, has both Python and TypeScript/WASM ports). For piano-only material, **ByteDance Piano Transcription** (or its `piano_transcription_inference` pip package) and **Transkun V2** offer the highest reported note F1 scores (>0.95 on MAESTRO) but require more compute. For ambitious multi-instrument transcription with instrument labels, **Google MT3** and the newer **YourMT3+** (MLSP 2024) are the leading transformer options, though both are heavier and more research-grade. Output across all of these is MIDI; MusicXML/sheet-music rendering must be a downstream step (e.g., MuseScore, music21, VexFlow, or OpenSheetMusicDisplay). No first-party Meta/Facebook AMT model exists today (their open-source audio work centers on speech and on MusicGen for generation).

## Models

### Spotify Basic Pitch
- Repo: https://github.com/spotify/basic-pitch (Python) and https://github.com/spotify/basic-pitch-ts (TypeScript/JS)
- Stars: ~5,000 (Python repo)
- License: Apache-2.0
- Instruments: Instrument-agnostic, polyphonic; works on voice, guitar, piano, winds, strings (designed for one dominant instrument at a time, no instrument labels in output)
- Input/Output: Input WAV/MP3/OGG/FLAC/M4A (resampled to 22,050 Hz). Output MIDI (with pitch bends), CSV note events, NPZ raw model output, optional sonified WAV. No native MusicXML; pair with music21 or MuseScore.
- Accuracy: From the ICASSP 2022 paper "A Lightweight Instrument-Agnostic Model for Polyphonic Note Transcription and Multipitch Estimation" (Bittner et al.), Basic Pitch is competitive with much larger models on MAESTRO, GuitarSet, MAPS, MedleyDB and Slakh; not state-of-the-art on piano vs. ByteDance/Transkun, but excellent across mixed instruments.
- Hardware: Extremely lightweight: <17K parameters, <20 MB peak RAM. Runs comfortably on CPU; faster-than-realtime on a modern laptop. Models ship as TFLite, CoreML, ONNX and TensorFlow.js so it can run in browser, mobile, and edge. Optional TensorFlow install for Python.
- Inference time: Sub-second to a few seconds per song on CPU; real-time browser/mobile demos exist (e.g., basicpitch.spotify.com).
- Maintenance: Active. Last release v0.4.0 around Aug 2024; multiple community ports (basicpitch.cpp via ONNXRuntime, Vamp plugin for Sonic Visualiser).
- Notes: Best "just works" option. Python: `pip install basic-pitch`; CLI: `basic-pitch out_dir input.wav`. TS package can run client-side, eliminating server costs.

### Google Magenta - Onsets and Frames
- Repo: https://github.com/magenta/magenta/tree/main/magenta/models/onsets_frames_transcription (also @magenta/music JS port)
- Stars: parent magenta repo ~19k; submodel has no separate count
- License: Apache-2.0
- Instruments: Piano (primary), with a separate drum-transcription config; velocity estimation
- Input/Output: WAV input, MIDI output
- Accuracy: Strong baseline on MAPS and MAESTRO at release (Hawthorne et al., 2018); note-with-offset F1 ~0.79 on MAESTRO. Now superseded by ByteDance/Transkun/MT3.
- Hardware: GPU recommended for training; inference works on CPU. Browser version uses TensorFlow.js and runs in real time on a laptop.
- Maintenance: Inactive. The README points users to MT3 as the successor; the parent magenta repo was archived in early 2026. Use only if you specifically want the JS in-browser piano model.
- Notes: Best historical reference; primarily useful via @magenta/music for browser piano demos.

### Google MT3 (Multi-Task Multitrack Music Transcription)
- Repo: https://github.com/magenta/mt3
- Stars: ~1.7k
- License: Apache-2.0
- Instruments: Multi-instrument with General MIDI program labels (piano, guitar, bass, strings, brass, woodwinds, drums, etc.). Two checkpoints: piano-only and full multitrack.
- Input/Output: WAV input via Colab demo, outputs note sequences/MIDI with per-track instrument labels. No MusicXML.
- Accuracy: State-of-the-art at ICLR 2022 across six datasets (MAESTRO, Slakh2100, Cerberus4, GuitarSet, MusicNet, URMP). Reports Frame F1, Onset F1, Onset+Offset F1 at three granularities (Flat / MIDI Class / Full).
- Hardware: Built on T5X/JAX, so GPU/TPU strongly recommended. Inference is non-trivial on CPU. Training "not (easily) supported" per README.
- Maintenance: Lightly maintained; ~71 commits, 43 open issues. Several active forks: MR-MT3 (instrument-leakage mitigation), MT3-pytorch, YourMT3.
- Notes: Best open-source choice if you must label which instrument plays which note. Heavy ops dependency on T5X makes deployment harder than Basic Pitch.

### YourMT3 / YourMT3+ (Sungkyunkwan University, MLSP 2024)
- Repo: https://github.com/mimbres/YourMT3
- Stars: ~219
- License: GPL-3.0 (note: copyleft - constrains commercial use vs. Apache/MIT)
- Instruments: Multi-instrument including direct vocal transcription (no separation pre-processor needed)
- Input/Output: Audio in, MIDI/note sequence out
- Accuracy: Paper (arXiv:2407.04822) reports competitive or superior results vs. MT3 on ten public datasets; uses hierarchical time-frequency attention transformer with mixture-of-experts and cross-stem augmentation.
- Hardware: GPU required for practical inference (transformer encoder-decoder).
- Maintenance: Active in 2024 with a HuggingFace Spaces demo; YouTube ingestion is flaky per repo notes.
- Notes: Most modern transformer multi-instrument option. GPL-3.0 may be a blocker if your app is closed-source.

### ByteDance High-Resolution Piano Transcription
- Repo: https://github.com/bytedance/piano_transcription (training/research) and https://github.com/qiuqiangkong/piano_transcription_inference (pip package for inference)
- Stars: ~2,000 (training repo); ~462 (inference package)
- License: Apache-2.0
- Instruments: Piano only, including sustain/sostenuto/soft pedal events
- Input/Output: MP3/WAV input (needs ffmpeg), MIDI output with pedal CCs
- Accuracy: Best-in-class on MAESTRO at release. Reported metrics: frame AP 0.9285, regression onset MAE 0.097, offset MAE 0.135, velocity MAE 0.027 (from paper "High-resolution Piano Transcription with Pedals by Regressing Onset and Offset Times", Kong et al., 2020).
- Hardware: Trained on Tesla V100 32GB; inference works on CPU but GPU is much faster. Inference package supports `device='cuda'` or `'cpu'`.
- Maintenance: Main training repo archived Dec 2025 (read-only). The inference pip package (`pip install piano_transcription_inference`) is the practical entry point and remains usable.
- Notes: If users will mostly upload piano recordings, this is arguably the highest-quality option. Easy GUI wrapper at https://github.com/azuwis/pianotrans.

### Transkun (Yan et al., ISMIR 2024)
- Repo: https://github.com/Yujia-Yan/Transkun
- Stars: ~334
- License: MIT
- Instruments: Piano only
- Input/Output: MP3/WAV in, MIDI out
- Accuracy: Note Onset+Offset F1 - MAESTRO V3: 0.9505, MAPS: 0.8843, SMD: 0.9448 (V2 transformer model). Among the highest reported piano scores in any open repo.
- Hardware: Runs on CPU by default; `--device cuda` for GPU acceleration. Multiple model sizes available (V2, V2 Aug, V2 No Ext).
- Inference: Trivial CLI - `pip3 install transkun` then `transkun input.mp3 output.mid`.
- Maintenance: Active; transformer architecture introduced in V2 (2024).
- Notes: Strongest CLI-friendly piano transcriber with a permissive MIT license. Excellent fallback or default for piano.

### Omnizart (Music and Culture Technology Lab)
- Repo: https://github.com/Music-and-Culture-Technology-Lab/omnizart
- Stars: ~1.9k
- License: MIT
- Instruments: Pitched instruments, vocal melody (note + frame F0), drums, chord progressions, beat tracking - the broadest task coverage of any toolkit listed
- Input/Output: Audio in; per-task outputs (MIDI for notes, label files for chords/beats). No native MusicXML.
- Accuracy: Combines multiple per-task models (vocal contour from Bittner et al., chord from Chen & Su, drum from Wei et al.); typically not SOTA on any single task but solid across all.
- Hardware: TensorFlow-based; GPU recommended for batch jobs but CPU works. ARM macOS unsupported.
- Maintenance: Mostly stagnant. Last release v0.5.0 in Dec 2021; some open issues unresolved. Replicate hosting available.
- Notes: Useful if you want chord/beat alongside notes in one library, but you'll likely outgrow it for note-level accuracy.

### Meta / Facebook
- No first-party automatic music transcription model has been released. Meta's open-source audio releases are speech (Omnilingual ASR, MMS, wav2vec 2.0) or music *generation* (MusicGen, AudioCraft). Skip for AMT.

### Honorable Mentions / Newer 2024-2026
- **MR-MT3** (https://github.com/gudgud96/MR-MT3) - Memory-retaining multi-track variant of MT3, ICASSP 2024, mitigates instrument leakage. Apache-2.0.
- **MT3-PyTorch** (https://github.com/rlax59us/MT3-pytorch) - Unofficial PyTorch port of MT3 for those who don't want JAX/T5X.
- **Music-Transcription-with-Semantic-Segmentation** (https://github.com/BreezeWhite/Music-Transcription-with-Semantic-Segmentation) - SOTA-class on MAPS and MusicNet; MIT license.
- **Streaming Piano Transcription based on Consistent Onset** (arXiv:2503.01362) - 2025 paper on real-time/streaming piano AMT; check for code drop.
- **2025 AI4Musicians AMT Challenge** (https://ai4musicians.org/transcription/2025transcription.html) - Winners must open-source; worth tracking for fresh top models late-2025.

## Recommendation
For an MVP that targets a *single* user-specified instrument at a time (the apparent product spec for Play by Ear Helper):

1. **Default engine: Spotify Basic Pitch.** Apache-2.0, instrument-agnostic, polyphonic, runs in browser via `basic-pitch-ts` (no server cost) or as a thin Python service. Smallest integration risk.
2. **Optional "high-quality piano mode": Transkun V2** (MIT) or **piano_transcription_inference** (Apache-2.0). Switch to these when the user tags the input as solo piano - both are pip-installable one-liners and beat Basic Pitch on piano F1.
3. **Future: MT3 or YourMT3+** if you later want automatic instrument labeling for a multi-instrument mix. Be prepared for GPU hosting and (for YourMT3) GPL-3.0 implications.

Pipeline shape: audio file -> Basic Pitch (or Transkun for piano) -> MIDI -> music21 / Verovio / OpenSheetMusicDisplay to render playable sheet music in the browser. MusicXML can be produced from the MIDI via music21 (`stream.write('musicxml')`).

## Sources
- https://github.com/spotify/basic-pitch
- https://github.com/spotify/basic-pitch-ts
- https://basicpitch.spotify.com/about
- https://engineering.atspotify.com/2022/06/meet-basic-pitch
- https://github.com/magenta/magenta/tree/main/magenta/models/onsets_frames_transcription
- https://magenta.withgoogle.com/onsets-frames
- https://github.com/magenta/mt3
- https://arxiv.org/abs/2111.03017
- https://magenta.withgoogle.com/transcription-with-transformers
- https://github.com/mimbres/YourMT3
- https://arxiv.org/abs/2407.04822
- https://github.com/bytedance/piano_transcription
- https://github.com/qiuqiangkong/piano_transcription_inference
- https://github.com/bytedance/GiantMIDI-Piano
- https://ieeexplore.ieee.org/document/9585550/
- https://github.com/Yujia-Yan/Transkun
- https://github.com/Music-and-Culture-Technology-Lab/omnizart
- https://arxiv.org/abs/2106.00497
- https://github.com/gudgud96/MR-MT3
- https://arxiv.org/html/2403.10024v1
- https://github.com/rlax59us/MT3-pytorch
- https://github.com/BreezeWhite/Music-Transcription-with-Semantic-Segmentation
- https://arxiv.org/pdf/2503.01362
- https://ai4musicians.org/transcription/2025transcription.html
- https://github.com/azuwis/pianotrans
- https://github.com/sevagh/basicpitch.cpp
- https://venturebeat.com/ai/meta-returns-to-open-source-ai-with-omnilingual-asr-models-that-can
- https://huggingface.co/spotify/basic-pitch
