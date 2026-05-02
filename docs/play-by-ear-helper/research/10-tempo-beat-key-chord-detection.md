# Tempo, Beat, Key, and Chord Detection

## Summary
Raw pitch transcription alone produces unreadable sheet music: notes drift across barlines, accidentals proliferate, and rhythmic values are nonsensical. A "musical context layer" that estimates tempo/beats, downbeats and time signature, key, and chord progression is what turns a stream of (pitch, time, duration) events into properly notated music. Modern open-source MIR offers strong tools at every stage: librosa and Essentia for classical baselines, madmom for high-accuracy beat/downbeat/onset CRNN models, BeatNet and Beat This! for state-of-the-art neural beat/downbeat tracking, BTC and Chordino for chord recognition, and the All-In-One Music Structure Analyzer for joint beat/downbeat/segment analysis. The recommended pipeline is: (optional) source separation, then onset+beat+downbeat tracking, then key and chord estimation in parallel with pitch transcription, then quantize the pitch events onto the beat grid and use key/chord context to choose enharmonic spellings before engraving.

## Tools

### Beat / Tempo
- **librosa.beat (`beat_track`, `plp`)** - The Python MIR baseline. Dynamic-programming beat tracker on an onset-strength envelope, plus a Predominant Local Pulse method for time-varying tempo. ISC license, trivially integrable with the rest of librosa. Accuracy is acceptable for steady pop/rock but lags ML methods on classical or expressive music; community reports note that beats can land slightly late vs. madmom because they are anchored to onset peaks. (https://librosa.org/doc/main/generated/librosa.beat.beat_track.html, https://deepwiki.com/librosa/librosa/5.2-beat-tracking-and-tempo-estimation)
- **madmom** - CRNN-based beat, downbeat, tempo, and meter tracking with a Dynamic Bayesian Network postprocessor. State-of-the-art for many years; ranked among the best on standard benchmarks. Source code is BSD; pretrained models are CC-BY-NC-SA 4.0, so commercial use of the shipped weights requires contacting the authors. Pure-Python, easy `pip install madmom`. (https://github.com/CPJKU/madmom, https://pypi.org/project/madmom/, https://arxiv.org/abs/1605.07008)
- **BeatNet / BeatNet+** - ISMIR 2021 (BeatNet) and 2024 (BeatNet+) joint beat/downbeat/tempo/meter tracker using a causal CRNN plus two-stage particle filtering. Supports streaming-from-mic, real-time-from-file, and offline modes. BeatNet+ adds robustness to non-percussive and vocal-only audio. Apache 2.0-style permissive license on the repo. (https://github.com/mjhydri/BeatNet, https://transactions.ismir.net/articles/10.5334/tismir.198)
- **Beat This! (CPJKU, ISMIR 2024)** - Transformer-based beat/downbeat tracker that drops the DBN postprocessor entirely; trained on diverse data (classical, solo instruments, time-signature changes). Reports the best published F1 on beat and downbeat as of ISMIR 2024. Has community ports to C++ and Rust, a REAPER plugin (ReaBeat), and is a strong drop-in replacement for madmom when license/commercial-use is a concern. (https://github.com/CPJKU/beat_this, https://github.com/b451c/ReaBeat)
- **All-In-One Music Structure Analyzer (mir-aidj)** - Joint model that emits tempo, beats, downbeats, and functional segments (intro/verse/chorus/bridge/outro) from source-separated spectrograms. State-of-the-art on the Harmonix Set. Useful when you also want structure for arrangement decisions. Available as a Python package, CLI, Hugging Face Space, and Replicate API. (https://github.com/mir-aidj/all-in-one, https://arxiv.org/abs/2307.16425)

### Key
- **Krumhansl-Schmuckler algorithm** - Classical baseline. Computes a 12-bin pitch-class profile (chromagram histogram) and correlates it against 24 hand-tuned major/minor key profiles derived from Krumhansl-Kessler probe-tone experiments. The key with the highest Pearson correlation wins. Trivial to implement on top of `librosa.feature.chroma_cqt`; works very well on diatonic pop/Bach, less well on chromatic or modal music (Chopin etc.). Variants: Temperley's reweighted profiles, Aarden-Essen, Bellman-Budge. (http://rnhart.net/articles/key-finding/, https://davidtemperley.com/wp-content/uploads/2015/11/temperley-mp99.pdf)
- **librosa** - Doesn't ship a dedicated key estimator, but `librosa.feature.chroma_cqt` / `chroma_cens` plus a hand-rolled K-S correlation is the standard approach. Reported practical accuracy in the 85-95% range on pop/rock. (https://stemsplit.io/blog/bpm-key-detection-feature)
- **Essentia (`KeyExtractor`, `Key`)** - C++ library with Python bindings (Affero GPL or commercial license from MTG-UPF). Multiple profile choices (Temperley, Krumhansl, Edma) and a deep-learning option via TensorFlow inference of in-house models from MTG. Generally outperforms naive librosa+K-S on EDM/pop. Has a JS port (Essentia.js) for browser use. (https://essentia.upf.edu/, https://transactions.ismir.net/articles/10.5334/tismir.111)
- **Deep-learning key detection** - Various ResNet/CRNN approaches; integrated into Essentia model zoo and into commercial tools (e.g. Mixed in Key, StemSplit). Helpful when modulations or chromatic harmony confuse profile-correlation methods.

### Chord
- **Chordino / NNLS Chroma (c4dm, QMUL)** - Open-source Vamp plugin for chord transcription. Computes NNLS-chroma features and matches against a user-editable chord dictionary, optionally with Viterbi decoding. The de-facto baseline for amateur-grade chord transcription; widely used (e.g. Sonic Visualiser). GPL-licensed. (https://code.soundsoftware.ac.uk/projects/nnls-chroma/, https://github.com/c4dm/nnls-chroma)
- **BTC - Bi-directional Transformer for Chord Recognition (Park & Choi, ISMIR 2019)** - PyTorch implementation; single-phase training, competitive accuracy on Isophonics/Billboard. Repo includes pretrained weights and CQT preprocessing. MIT-style. (https://github.com/jayg996/BTC-ISMIR19, https://arxiv.org/abs/1907.02698)
- **Harmony-Transformer-v2 (Tsung-Ping Chen)** - Improved transformer for joint chord segmentation+labeling; repo also re-implements BTC and several CRNN baselines for direct comparison. (https://github.com/Tsung-Ping/Harmony-Transformer-v2)
- **Chord-CNN / CRNN baselines** - McFee, Korzeniowski, Humphrey models; mostly available as research code. Madmom does not ship chord recognition out of the box, but its features module is often used as a feature frontend.
- **Commercial-style services** - Chordify, Klangio, AnthemScore are closed-source but use similar feature-frontends (CQT/chroma) plus learned classifiers; useful as reference targets.

### Time Signature / Downbeats
- **madmom `DBNDownBeatTrackingProcessor` / `RNNDownBeatProcessor`** - Joint downbeat detection over a configurable set of time signatures (typically 3/4 and 4/4). Mature and well-benchmarked. (https://github.com/CPJKU/madmom/blob/main/madmom/features/downbeats.py)
- **BeatNet / BeatNet+** - Outputs downbeats and meter alongside beats; supports real-time. (https://github.com/mjhydri/BeatNet)
- **Beat This! (CPJKU)** - Joint beat+downbeat without DBN; training data covers time-signature changes, so it tolerates 3/4, 6/8, etc. (https://github.com/CPJKU/beat_this)
- **All-In-One** - Returns downbeats jointly with segments; useful when 4/4 cannot be assumed. (https://github.com/mir-aidj/all-in-one)
- **Time-signature classifiers** - ResNet18 and CRNN-based meter classifiers reported in recent literature; often run on top of detected downbeat spacing (e.g. ReaBeat infers 2/4-7/4 from inter-downbeat intervals). (https://link.springer.com/article/10.1186/s13636-024-00346-6, https://pmc.ncbi.nlm.nih.gov/articles/PMC8512143/)
- **Pop2Piano** - Not a meter detector per se; it's a T5-style encoder-decoder that maps pop audio directly to a tokenized piano MIDI cover, internally relying on a beat-conditioned tokenizer. Mentioned because it side-steps explicit beat/key/chord inference, but it produces piano covers, not faithful transcriptions, so it is more of a baseline / inspiration than a component. (https://github.com/sweetcocoa/pop2piano, https://huggingface.co/docs/transformers/model_doc/pop2piano)

### Onset
- **librosa `onset.onset_detect`** - Spectral-flux peak picking on an onset-strength envelope. Fast, ISC-licensed, good enough for percussive and most polyphonic material. (https://librosa.org/doc/main/)
- **madmom `OnsetPeakPickingProcessor` / `CNNOnsetProcessor`** - CNN trained on ~26k annotated onsets, 100 fps frame rate; distinguishes percussive vs. harmonic onsets. Generally more accurate than librosa, especially on soft/legato attacks. (https://madmom.readthedocs.io/en/v0.16/modules/features/onsets.html)
- **Essentia `OnsetDetection` / `OnsetDetectionGlobal`** - Multiple detection functions (HFC, complex, melflux); useful as a second opinion. (https://essentia.upf.edu/)
- **Music structure / segmentation**: **MSAF** (Nieto & Bello, 2015) is the canonical Python framework for boundary detection + structural grouping (verse/chorus). It plugs into librosa for features. (https://github.com/urinieto/msaf, https://ccrma.stanford.edu/~urinieto/MARL/publications/NietoBello-ISMIR2015.pdf)

## Pipeline: Combining with Pitch Transcription

A practical Play-by-Ear pipeline layers the context features around a pitch transcriber (e.g. Basic Pitch, MT3, Onsets-and-Frames):

1. **Preprocessing / source separation (optional)** - Run Demucs or Spleeter to isolate the melodic/harmonic stem before transcription. This dramatically improves both pitch transcription and chord recognition on full mixes. The All-In-One analyzer already operates on demixed spectrograms internally.
2. **Onset detection** - Either madmom CNN onsets or librosa onset_detect. Onsets are reused both as prior probabilities for the pitch transcriber and as candidate note start times for quantization.
3. **Beat + downbeat + tempo tracking** - Beat This! or BeatNet+ for accuracy on diverse material; madmom or librosa for a permissive-license baseline. Output: a list of beat times with downbeat flags and a global tempo (BPM).
4. **Time-signature inference** - Read the meter directly from the joint tracker (BeatNet/Beat This!/All-In-One), or infer from the modal inter-downbeat / inter-beat ratio. Confirm via a meter classifier if needed.
5. **Key estimation** - Compute a chroma feature (CQT-based, beat-synchronous) and run Krumhansl-Schmuckler / Temperley correlation, or call `essentia.standard.KeyExtractor`. Use a sliding window if modulations are likely.
6. **Chord recognition** - Run Chordino or BTC on the full audio. Output is a frame-level chord sequence; align it to beats to get one chord per beat (or per half-bar).
7. **Pitch transcription** - Run your AMT model (Basic Pitch / MT3 / Onsets-and-Frames) to get a list of (onset, offset, pitch, velocity) tuples in seconds.
8. **Quantization to the beat grid** - Convert each note's onset/offset from seconds to beat fractions using the beat times; snap to the nearest sensible subdivision (e.g. 16th or triplet 8th) using a HMM/DP quantizer. Downbeats define barlines.
9. **Enharmonic spelling and accidentals** - Use the detected key signature to choose F# vs. Gb etc. Use the chord at each beat to disambiguate borderline pitches (a B over a G major chord should be spelled B, not Cb) and to correct octave/edge errors from the pitch model.
10. **Voice/staff assignment and engraving** - Split into treble/bass voices, group notes into beats and bars per the time signature, and emit MusicXML/LilyPond/MEI for rendering by Verovio, MuseScore, or LilyPond.

The Oh-Sheet open-source project illustrates a similar end-to-end flow (Basic Pitch + beat tracking + key detection + two-hand arrangement + RL-trained engraving). ScoreCloud's documented three-stage pipeline (separation -> analysis -> rule-based music-cognition model) follows the same shape. (https://github.com/Oh-Sheet-Team/oh-sheet, https://scorecloud.com/learn/how-to-convert-audio-to-sheet-music/)

## Sources
- librosa beat docs: https://librosa.org/doc/main/generated/librosa.beat.beat_track.html
- librosa beat/tempo overview: https://deepwiki.com/librosa/librosa/5.2-beat-tracking-and-tempo-estimation
- madmom GitHub: https://github.com/CPJKU/madmom
- madmom PyPI / license: https://pypi.org/project/madmom/
- madmom paper (arXiv 1605.07008): https://arxiv.org/abs/1605.07008
- madmom downbeats source: https://github.com/CPJKU/madmom/blob/main/madmom/features/downbeats.py
- madmom onsets docs: https://madmom.readthedocs.io/en/v0.16/modules/features/onsets.html
- BeatNet GitHub: https://github.com/mjhydri/BeatNet
- BeatNet+ paper (TISMIR 2024): https://transactions.ismir.net/articles/10.5334/tismir.198
- Beat This! GitHub: https://github.com/CPJKU/beat_this
- ReaBeat (Beat This! in REAPER): https://github.com/b451c/ReaBeat
- All-In-One Music Structure Analyzer: https://github.com/mir-aidj/all-in-one
- All-In-One paper (arXiv 2307.16425): https://arxiv.org/abs/2307.16425
- Krumhansl-Schmuckler explainer: http://rnhart.net/articles/key-finding/
- Temperley critique of K-S: https://davidtemperley.com/wp-content/uploads/2015/11/temperley-mp99.pdf
- Essentia homepage: https://essentia.upf.edu/
- Essentia.js (TISMIR): https://transactions.ismir.net/articles/10.5334/tismir.111
- BPM/key accuracy reference: https://stemsplit.io/blog/bpm-key-detection-feature
- NNLS Chroma / Chordino: https://code.soundsoftware.ac.uk/projects/nnls-chroma/
- Chordino / NNLS Chroma GitHub: https://github.com/c4dm/nnls-chroma
- BTC GitHub: https://github.com/jayg996/BTC-ISMIR19
- BTC paper (arXiv 1907.02698): https://arxiv.org/abs/1907.02698
- Harmony-Transformer-v2: https://github.com/Tsung-Ping/Harmony-Transformer-v2
- Pop2Piano GitHub: https://github.com/sweetcocoa/pop2piano
- Pop2Piano on Hugging Face: https://huggingface.co/docs/transformers/model_doc/pop2piano
- Time-signature detection survey: https://pmc.ncbi.nlm.nih.gov/articles/PMC8512143/
- Time-signature ResNet18 paper: https://link.springer.com/article/10.1186/s13636-024-00346-6
- MSAF GitHub: https://github.com/urinieto/msaf
- MSAF paper (ISMIR 2015): https://ccrma.stanford.edu/~urinieto/MARL/publications/NietoBello-ISMIR2015.pdf
- Open-source pipeline reference (Oh-Sheet): https://github.com/Oh-Sheet-Team/oh-sheet
- ScoreCloud pipeline overview: https://scorecloud.com/learn/how-to-convert-audio-to-sheet-music/
- Open-source beat detection comparison: https://biff.ai/a-rundown-of-open-source-beat-detection-models/
