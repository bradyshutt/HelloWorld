# Instrument Classification & Recognition

## Summary
Instrument recognition for polyphonic music is a mature but still imperfect problem. The strongest open-source baselines come from large pretrained audio backbones (PANNs, AST, PaSST) fine-tuned on music-specific multi-label datasets like OpenMIC-2018 (20 classes, 10-sec clips) and MTG-Jamendo (40 instrument tags). Reported state-of-the-art mAP sits around 0.70-0.85 for clip-level multi-label tagging — good enough to suggest "this clip probably contains piano and guitar" but not reliable enough to be the sole source of truth in a transcription pipeline. Time-localized "which instrument is playing WHEN" detection (frame-level / instrument activity detection) is significantly harder; MedleyDB's stem-derived activation annotations are the standard reference, and accuracy drops noticeably vs. clip-level. For a user-facing product like Play by Ear Helper, instrument classification is best treated as a UX assist (preselect, confirm) rather than an authoritative gate — the user almost always already knows which instrument they want transcribed.

## Models / Tools

### YAMNet (Google)
- Repo: https://github.com/tensorflow/models/tree/master/research/audioset/yamnet
- Backbone: MobileNetV1, ~3.7M params, runs on 960ms frames
- Training: AudioSet (~1.57M YouTube 10-sec excerpts), 521 classes
- Accuracy: balanced mAP 0.306, d-prime 2.318, lwlrap 0.393 on the 20k AudioSet eval set (across all 521 classes, not music-specific)
- Notes: includes many instrument classes (piano, guitar, violin, flute, drums, etc.) but it is a general audio event classifier, not a music-tuned tagger. Good for "is there music? is there a piano-ish sound?" but coarse for fine-grained polyphonic music. Often used as a feature extractor / embedding backbone.

### PANNs (Pretrained Audio Neural Networks)
- Paper: https://arxiv.org/abs/1912.10211
- Repo: https://github.com/qiuqiangkong/audioset_tagging_cnn (and inference-only: https://github.com/qiuqiangkong/panns_inference)
- Best variant: Wavegram-Logmel-CNN, 0.439 mAP on AudioSet tagging (527 classes) — SOTA at time of publication
- Notes: Heavily used as a transfer-learning backbone for music tagging. Pip-installable inference package. Good baseline if you want to tag instruments without training anything.

### AST — Audio Spectrogram Transformer (MIT)
- Paper: https://arxiv.org/abs/2104.01778
- Repo: https://github.com/YuanGongND/ast
- HF: https://huggingface.co/MIT/ast-finetuned-audioset-10-10-0.4593
- Accuracy: 0.485 mAP on AudioSet, 95.6% on ESC-50, 98.1% on Speech Commands V2
- Notes: First convolution-free purely-attention model for audio. The HF checkpoint is the canonical "one-line instrument tagger" — pipe a wav in, get AudioSet labels (which include musical instruments) out. Practical drop-in.

### PaSST (Patchout faSt Spectrogram Transformer, JKU)
- Paper: https://arxiv.org/pdf/2110.05069
- Repo: https://github.com/kkoutini/PaSST
- Accuracy: SOTA on AudioSet (>0.495 mAP), beats AST while training ~4x faster with patchout regularization. Frequently used as a backbone for music tagging fine-tunes including MTG-Jamendo instrument subset.
- Notes: Best general-purpose backbone if you plan to fine-tune. Many community fine-tunes exist for music-specific tasks.

### EfficientAT / MN models
- Repo: https://github.com/fschmid56/EfficientAT
- Notes: Knowledge-distilled CNNs from PaSST teacher; near-SOTA mAP on AudioSet at a fraction of the compute. Good for on-device or low-latency.

### Essentia + Discogs-EffNet (MTG-UPF)
- Models: https://essentia.upf.edu/models.html
- Multi-label instrument tagger trained on MTG-Jamendo's 40-instrument subset: accordion, acoustic guitar, bass, bells, brass, cello, drums, flute, guitar, harp, keyboard, organ, piano, saxophone, strings, synthesizer, trumpet, violin, voice, etc.
- Notes: Production-grade C++/Python library, easy to call. Discogs-EffNet embeddings + small downstream classifier head. This is probably the most pragmatic music-domain instrument tagger available open source today.

### OpenMIC-2018 baselines
- Dataset: https://zenodo.org/records/1432913 (Spotify Research)
- Paper: https://brianmcfee.net/papers/ismir2018_openmic.pdf
- 20,000 10-second Free Music Archive clips, 20 instrument classes (multi-label, partial labels)
- Baselines: Random Forest (~0.66 mAP); attention-based DNN (Gururani et al. 2019) 0.70 mAP; hierarchical residual attention with multi-spectrogram features pushes higher (~0.80+ mAP per recent work).
- Notes: This is THE standard benchmark for "which instruments are in this 10-sec clip" multi-label.

### MedleyDB (NYU MARL)
- https://medleydb.weebly.com/ , https://steinhardt.nyu.edu/marl/research/resources/medleydb
- Provides per-stem activation envelopes (half-wave rectify, compress, smooth, downsample) → time-localized binary instrument activity at frame level. Source ID `.lab` files give [start, end, instrument] intervals.
- Notes: Standard for *time-localized* instrument detection (instrument activity detection / IAD). Smaller than OpenMIC (~120 multitracks) but richer per-clip annotation.
- Related: Medley-solos-DB (Lostanlen et al.) for solo instrument recognition cross-dataset.

### Hugging Face community models
- `dima806/musical_instrument_detection` — small fine-tuned audio classifier, single-label, limited instrument set, trained on monophonic / clean samples. Not great on real polyphonic music.
- `onnx-community/Musical-Instrument-Classification-ONNX` — ONNX export of a wav2vec2-based 9-class classifier. Same caveat: solo recordings.
- General pattern: most HF "instrument classification" checkpoints are single-label trained on clean isolated samples (Philharmonia, NSynth-style), which transfers poorly to real mixed music. For polyphonic real-world tagging, prefer AST/PaSST AudioSet checkpoints or Essentia's MTG-Jamendo head.

### Commercial APIs
- **AudioShake** (https://www.audioshake.ai/, https://developer.audioshake.ai/) — primarily a stem-separation product (vocals, drums, bass, guitar, piano, winds, strings, up to 14 stems). Doesn't market a standalone "instrument detection" endpoint, but their separation implicitly identifies which stems are present (a stem comes back empty if absent). Real-time SDK available.
- **Music.ai** (https://music.ai/modules/classification/instruments-detection/) — explicitly offers an "Instruments Detection" classification module plus general audio tagging. Pay-per-use API. Likely the most direct commercial fit for "what instruments are in this clip".
- **Cyanite.ai** — music auto-tagging with instrument tags among genre/mood. B2B catalog use case; less ideal for per-clip user uploads.

### Predominant / lead instrument detection
- "Predominant instrument recognition in polyphonic music" line of work (Han et al. 2016 CNN; later CRNN, transformer-ensemble variants). Reports 92.8% accuracy on real-world polyphonic excerpts in some setups (Solanki & Pandey 8-layer CNN), and per-instrument precision 0.86-0.99 in specialist-CNN ensembles (Blaszke & Kostek). Useful when you want "what's the LEAD instrument" rather than "what's all in there".
- Lead Instrument Detection from Multitrack Music (2025, https://arxiv.org/html/2503.03232v1) — newer framing for time-varying lead identification.

## Do We Need It?

Honest answer: **probably not as a hard pipeline step, but yes as UX polish.**

Arguments AGAINST building it in:
- The user opens the app *because* they want to learn a specific part. They almost always already know "I want the piano part" or "transcribe the bass". Asking the model to guess what they want is solving a problem they didn't have.
- Klangio and AnthemScore (the closest commercial analogs) work this way: user selects instrument up front, system runs an instrument-specific transcription model. No auto-detection in the user's flow.
- Adding classification adds latency (one more model pass), failure modes (wrong instrument detected → user confused), and scope.
- Instrument-specific transcription models (basic-pitch for general, MT3 for multi-instrument, drum-specific, bass-specific) are more accurate when conditioned on a known instrument than a general "transcribe whatever" model. The selection IS the conditioning.

Arguments FOR including it:
- Onboarding nudge: after upload, show "We detected: piano, drums, vocals. Which one do you want?" This is a much friendlier UX than a blank dropdown of 20 instruments — it filters the menu down to the 2-3 that actually matter for this clip.
- Sanity check: if the user picks "saxophone" and the classifier is 95% confident there's no sax in the clip, warn them. Saves a wasted transcription run.
- If we add stem separation later (e.g., AudioShake / Demucs), instrument detection naturally tells us which stems are worth extracting.
- Accessibility / discovery: a user might not know what they're hearing ("is that a clarinet or an oboe?"). Detection helps them name it.

UX implication summary: instrument classification is a **suggestion layer**, not a gating layer. It should never block the user from picking what they want, but it can pre-fill the picker, surface the most likely options first, and warn on obvious mismatches. Treat it like spell-check, not like authentication.

Time-localized detection ("piano is playing from 0:12 to 0:38") is a *separate*, harder feature. It only matters if we're doing per-section transcription or visualizing instrument timelines. For MVP: skip it.

## Recommendation

For MVP:
1. **Don't build instrument classification into the critical path.** Let the user pick the instrument from a dropdown. That picker drives which transcription model we run.
2. **Optional v1.1 polish:** run one of the following on upload to pre-rank the dropdown:
   - Easiest: HF `MIT/ast-finetuned-audioset-10-10-0.4593` (one-line transformers pipeline, ~0.485 AudioSet mAP, includes instrument labels) — filter outputs to the AudioSet "Music"/"Musical instrument" subtree.
   - Better music-specific: Essentia + Discogs-EffNet + MTG-Jamendo instrument head (40 music instrument classes, properly multi-label, designed for real music).
   - Don't roll your own training unless we hit accuracy ceilings.
3. **Threshold conservatively.** Only auto-promote an instrument to the suggestions list at high confidence (e.g., p > 0.5 multi-label, calibrated on a held-out set). False positives are worse than false negatives here.
4. **Skip time-localized IAD for now.** Reconsider only if the product grows into "highlight where the trumpet plays" features.
5. **If we ever go commercial-API:** Music.ai's instrument detection module is the most direct fit; AudioShake is overkill unless we also want stems.

## Sources
- [OpenMIC-2018 paper (ISMIR 2018)](https://brianmcfee.net/papers/ismir2018_openmic.pdf)
- [OpenMIC-2018 Zenodo dataset](https://zenodo.org/records/1432913)
- [OpenMIC-2018 GitHub](https://github.com/cosmir/openmic-2018)
- [Spotify Research: OpenMIC-2018](https://research.atspotify.com/publications/openmic-2018-an-open-dataset-for-multiple-instrument-recognition)
- [Gururani et al., Attention Mechanism for Musical Instrument Recognition (ISMIR 2019)](https://arxiv.org/abs/1907.04294)
- [Hierarchical Residual Attention Network, MDPI 2024](https://www.mdpi.com/2076-3417/14/23/10837)
- [MedleyDB project](https://medleydb.weebly.com/)
- [MedleyDB at NYU MARL](https://steinhardt.nyu.edu/marl/research/resources/medleydb)
- [MedleyDB Python API docs](https://medleydb.readthedocs.io/en/latest/api.html)
- [Lead Instrument Detection from Multitrack Music (2025)](https://arxiv.org/html/2503.03232v1)
- [YAMNet on TF Hub](https://www.tensorflow.org/hub/tutorials/yamnet)
- [YAMNet GitHub](https://github.com/tensorflow/models/tree/master/research/audioset/yamnet)
- [YAMNet class map CSV](https://github.com/tensorflow/models/blob/master/research/audioset/yamnet/yamnet_class_map.csv)
- [PANNs paper (arXiv)](https://arxiv.org/abs/1912.10211)
- [PANNs GitHub](https://github.com/qiuqiangkong/audioset_tagging_cnn)
- [panns_inference pip package](https://github.com/qiuqiangkong/panns_inference)
- [AST paper (arXiv)](https://arxiv.org/abs/2104.01778)
- [AST GitHub](https://github.com/YuanGongND/ast)
- [AST on Hugging Face](https://huggingface.co/MIT/ast-finetuned-audioset-10-10-0.4593)
- [PaSST paper (arXiv)](https://arxiv.org/pdf/2110.05069)
- [PaSST GitHub](https://github.com/kkoutini/PaSST)
- [EfficientAT (Transformer-to-CNN distillation)](https://github.com/fschmid56/EfficientAT)
- [Essentia models](https://essentia.upf.edu/models.html)
- [MTG-Jamendo dataset GitHub](https://github.com/MTG/mtg-jamendo-dataset)
- [MTG-Jamendo project page](https://mtg.github.io/mtg-jamendo-dataset/)
- [MTG-Jamendo Zenodo](https://zenodo.org/records/3826813)
- [HF: dima806/musical_instrument_detection](https://huggingface.co/dima806/musical_instrument_detection)
- [HF: onnx-community/Musical-Instrument-Classification-ONNX](https://huggingface.co/onnx-community/Musical-Instrument-Classification-ONNX)
- [AudioShake home](https://www.audioshake.ai/)
- [AudioShake developer docs](https://developer.audioshake.ai/)
- [Music.ai Instruments Detection module](https://music.ai/modules/classification/instruments-detection/)
- [Klangio Transcription Wizard (instrument selection UX)](https://klang.io/blog/transcription-wizard-update/)
- [AnthemScore](https://www.lunaverus.com/)
- [Hierarchical Classification for Instrument Activity Detection in Orchestral Music (TASLP 2023)](https://dl.acm.org/doi/10.1109/TASLP.2023.3291506)
- [Han et al., Deep CNNs for predominant instrument recognition](https://arxiv.org/pdf/1605.09507)
- [Cross-Attentive CNNs for predominant instrument recognition (2025)](https://www.mdpi.com/2227-7080/14/1/3)
