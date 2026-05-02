# Trumpet/Horn Separation from Jazz Combo

## Summary

Realistic SDR for an isolated trumpet from a 4-piece jazz combo (piano + drums + bass + trumpet) in 2026 is roughly **4-7 dB**, with the upper bound only attainable on clean studio audio fed through query-based or commercial 10+ stem models. The most defensible recommendation today is **AudioShake API or LALAL.AI Perseus** for production use, or **Banquet (query-bandit, MIT)** for a self-hosted open-source path - both treat brass as a first-class stem instead of dumping it into "other". The biggest gap is that **no widely-deployed model is trained primarily on jazz-quartet audio**: most brass-aware models were trained on pop/rock multitracks (MoisesDB) where horn sections are rarer than they are in jazz, so trumpet-against-piano scenarios remain the worst case (frequency overlap, simultaneous melodic lines). Output quality is good enough for **monophonic pitch tracking on the trumpet's solo passages**, but expect smearing and pitch confusion when the piano is comping in the same register as the trumpet.

## Tools / Approaches

### Demucs `htdemucs_6s` (Meta / facebookresearch)
- Type: hybrid transformer (waveform + spectrogram), 6 stems: vocals, drums, bass, other, piano, guitar.
- License: MIT.
- Accuracy on jazz/horns: **No brass stem.** Trumpet falls into `other` along with anything not piano/guitar. Community reports note the piano stem is "not working great" with substantial bleed; guitar is "okay". For trumpet isolation it is effectively no better than the standard 4-stem `htdemucs` model - you would still need a second pass.
- API or self-hosted: self-hosted (PyTorch); also exposed by MVSep, UVR, and Replicate.
- Notes: Standard 4-stem MUSDB-HQ SDR is ~9.0 dB (vocals/drums/bass/other). The 6-stem variant only adds piano/guitar; brass remains lumped into "other".

### Banquet / query-bandit (Watcharasupat & Lerch, 2024)
- Type: query-based bandsplit RNN with a single decoder driven by PaSST instrument-recognition embeddings. 24.9M params.
- License: MIT.
- Accuracy on jazz/horns: Trained/evaluated on MoisesDB, which includes `brass` and `reeds` as fine-level stems. Paper shows it can extract long-tail stems including reeds, organs, brass; "balanced sampling... performed the best for bass synth, pitched percussion, reeds, and brass." Approaches 6-stem HTDemucs on VDBO, beats it on guitar/piano. Specific brass SDR on MoisesDB is in the low single digits (the long-tail stems are the hardest case in the paper).
- API or self-hosted: self-hosted (https://github.com/kwatcharasupat/query-bandit).
- Notes: Brass in MoisesDB is a *category* - includes trumpets, trombones, French horns, tuba; the model is not trained to single out trumpet specifically. For a quartet with only one brass instrument, the brass stem ~= trumpet stem.

### Hyperellipsoidal Queries (Watcharasupat & Lerch, Jan 2025, arXiv:2501.16171)
- Type: follow-up to Banquet; query-by-region using hyperellipsoidal regions in instrument-embedding space, allowing "this trumpet AND everything around it" or tight queries on a single instrument.
- License: presumed MIT (same lab); code release status unverified.
- Accuracy on jazz/horns: Claims SOTA on MoisesDB long-tail stems; designed specifically to handle the "trumpet + nearby timbres" case better than fixed-class models.
- Notes: This is the most promising published direction for our exact use case but is a research artifact; not yet packaged for end-user use.

### Bandit-v2 (Watcharasupat, 2024)
- Type: cinematic source separation: outputs `dialogue / music / effects`. Trained on Divide and Remaster v3.
- License: open (Zenodo weights, GitHub code).
- Accuracy on jazz/horns: **Wrong tool.** Trumpet is in the `music` stem along with everything else musical. Useful only if jazz is mixed with dialogue/SFX (e.g., film scenes).

### LALAL.AI Perseus (commercial)
- Type: commercial transformer, 10 stems including `wind instruments` (which covers brass: trumpet, trombone, French horn, tuba, plus woodwinds: sax, flute, clarinet).
- License: paid SaaS; API available; per-minute billing.
- Accuracy on jazz/horns: Marketed at exactly the jazz/orchestral use case ("jazz... would not exist without wind instruments"). Andromeda (the higher tier) claims ~10% SDR uplift over Perseus. No public per-instrument SDR figures, but anecdotal reports rate it among the best for brass.
- API or self-hosted: cloud-only.
- Notes: The "wind" stem mixes brass and woodwinds. In our quartet there's only a trumpet, so this collapses to a clean trumpet stem.

### AudioShake (commercial)
- Type: commercial multi-stem separator. Stems include vocals, lead/backing vocals, drums, bass, acoustic/electric guitar, piano, keys, strings, **wind instruments**.
- License: paid API + on-device SDK (iOS/macOS/Android/Windows/Linux). $19.99/mo entry; per-stem credit packs (4/10/20 stems).
- Accuracy on jazz/horns: Earned highest SDR in Sony Music Demixing Challenges across music and film tracks. Used in production by UMG, Disney Music, Warner. Specific brass SDR not published, but consistently reported as on par with or better than LALAL.AI on instrument extraction in head-to-head reviews.
- API or self-hosted: cloud API; SDK for on-device.
- Notes: Strongest production-grade choice; real-time SDK lets you avoid round-tripping a 15-second clip to the cloud.

### Moises.ai (commercial)
- Type: SaaS, 7 stems via VST plugin: vocals / keys / drums / guitar / bass / strings / other.
- License: paid SaaS.
- Accuracy on jazz/horns: **No dedicated brass/wind stem.** User reports specifically call out that horns get clumped into "other" or "strings". Not appropriate for our use case.

### Spleeter 5stems (Deezer, 2019)
- Type: U-Net spectrogram model. 5 stems: vocals / drums / bass / piano / other.
- License: MIT.
- Accuracy on jazz/horns: No brass stem. Trumpet ends up in `other`. Known to underperform Demucs on jazz/acoustic recordings (more bleed, weaker on transients). 5-stem SDRs in the original paper are in the 4-6 dB range and several dB behind Demucs on equivalent stems.
- Notes: Largely superseded; only relevant if you need extreme speed and are happy with low quality.

### MVSep / UVR community models
- Type: hosted suite of 30+ models: BS-Roformer, Mel-Band Roformer, MDX23C, SCNet XL, Demucs4, plus community-trained specialized models.
- License: mix of open and paid; MVSep has both free tiers and paid algorithms.
- Notable for our case:
  - **MVSep "Wind" / 53-stem BS-Roformer (Eddycrack864 / ZFTurbo)**: open-source BS-Roformer trained on a 53-instrument taxonomy including `brass`, `trumpet`, `trombone`, `french-horn`, `saxophone`, `clarinet`, `flute`. This is the only publicly released open model that names trumpet as its own output. Available via MVSep web UI and via `Music-Source-Separation-Training` toolkit on GitHub.
  - Community feedback: usable but inconsistent on real recordings; trained on a synthesized/multitrack-augmented corpus, so it generalizes unevenly to live jazz miking.
- Notes: Quality is not as polished as commercial Perseus/AudioShake but is the best open option that explicitly outputs `trumpet`.

### AudioSep / LASS (Liu et al., 2023-24)
- Type: language-queried universal separator. Trained on AudioSet/VGGSound/MUSIC/AudioCaps.
- License: research code, weights available; non-commercial CC for some checkpoints.
- Accuracy on jazz/horns: SDRi of **10.51 dB on the MUSIC dataset** (which includes solo trumpet, sax, etc.) - but MUSIC mixes are typically synthetic 2-source mixes, not 4-instrument live jazz. Real-world performance on a dense quartet drops materially.
- Notes: Lets you literally type "trumpet" as a query. Useful as an ensemble / second-stage cleanup tool.

### OmniSep (ICLR 2025)
- Type: omni-modal query separator (text, image, audio queries; supports query-mixup and negative queries).
- License: research (https://github.com/Exgc/OmniSep).
- Accuracy on jazz/horns: SOTA on MUSIC and VGGSOUND-CLEAN+ for query-based separation, but again primarily evaluated on lighter mixtures.
- Notes: The negative-query feature ("separate trumpet, suppress piano") is particularly interesting for our use case.

### GuideSep (Wen, Kim, Smaragdis - ISMIR 2025, arXiv:2507.01339)
- Type: diffusion-based generative MSS conditioned on (a) hummed/played mimicry of the target line, and (b) mel-spectrogram masks.
- License: open (https://github.com/YutongWen/GuideSep).
- Accuracy on jazz/horns: Designed exactly for arbitrary instrument extraction beyond 4 stems; user can hum the trumpet line to guide it. No published SDR-by-instrument numbers yet, but qualitatively strong on demos.
- Notes: Generative (diffusion), so it may *hallucinate* notes that weren't in the input - dangerous if downstream you trust the output for transcription.

### Cadenza / EnsembleSet (ConvTasNet, classical)
- Type: per-instrument ConvTasNet models trained on synthesized small classical ensembles.
- License: open (research).
- Accuracy on jazz/horns: Reports **9.0 dB SDR on strings and 4.5 dB on woodwinds** in small ensembles (arXiv:2505.17823). Trumpet not directly evaluated, but a brass instrument in a 4-piece would likely fall in the 4-6 dB range by analogy. Highlights that woodwind/brass in dense ensembles is materially harder than VDBO pop separation.

## Recent Research (2024-2026)

- **Banquet (Watcharasupat & Lerch, 2024)** - query-based, single-decoder; first model to credibly extract MoisesDB `brass` and `reeds` stems. arXiv:2406.18747.
- **Hyperellipsoidal Queries (Watcharasupat & Lerch, Jan 2025)** - claims SOTA on MoisesDB long-tail with region-based queries. arXiv:2501.16171.
- **OmniSep (ICLR 2025)** - omni-modal queries with positive/negative composition; relevant for "isolate trumpet, remove piano". arXiv:2410.21269.
- **GuideSep (ISMIR 2025)** - hum-to-separate via diffusion; novel UX for jazz where user can sing the trumpet line. arXiv:2507.01339.
- **Source Separation of Small Classical Ensembles (May 2025)** - quantifies that woodwind/brass-heavy small ensembles sit at ~4.5-6.9 dB SDR, far below pop-music VDBO numbers. arXiv:2505.17823.
- **Moises-Light (Oct 2025)** - lightweight band-split U-Net hitting ~9.96 dB average SDR with far fewer params; lifts open-source baselines. arXiv:2510.06785.
- **ACMID 7-stem dataset (Oct 2025)** - automatic curation of an instrument dataset for 7-stem MSS, hinting at brass-aware 7-stem models in the near future. arXiv:2510.07840.
- **Spheres dataset (Nov 2025)** - multitrack orchestral recordings, includes brass. arXiv:2511.21247.

## Verdict for Our Use Case

A 15-second jazz quartet clip (piano + bass + drums + trumpet) sent to a current brass-aware separator will produce a **trumpet-dominant stem with audible bleed**, primarily from piano (frequency overlap on the right hand) and ride cymbal (broadband high-frequency energy). Concretely:

- Best realistic SDR today: **~5-7 dB** for trumpet from a typical small-combo jazz mix (commercial Perseus / AudioShake), **~3-5 dB** from open-source (Banquet, ZFTurbo 53-stem BS-Roformer).
- This is **good enough for monophonic pitch tracking** (CREPE, SPICE, Basic Pitch) on the trumpet during solo passages where it's the lead voice. Pitch error rates rise sharply when piano is comping melodically in the trumpet's register or when the trumpet drops out.
- It is **not clean enough to feed a polyphonic transcriber expecting an isolated monophonic source** with confidence below 90% F1; expect false positives from piano bleed.

Practical recommendation:
1. **Hosted path (best quality, simplest)**: AudioShake API "wind instruments" stem, or LALAL.AI Perseus. ~$0.02-0.10 per 15-second clip.
2. **Self-hosted path (good enough, free)**: Banquet (query-bandit) with the `brass` query, or ZFTurbo's BS-Roformer 53-stem model with `trumpet` as the target. Plan for ~1-3 seconds inference per 15-second clip on a modest GPU.
3. **Pre-process before pitch tracking**: high-pass at ~150 Hz (kills bass bleed), apply spectral gate keyed to the trumpet's amplitude envelope, then run CREPE/Basic Pitch.
4. **Hedge against bleed**: run the same input through a second model with a *negative piano query* (OmniSep) or run htdemucs_6s and subtract its piano stem from the trumpet stem.

The honest expectation: this will work well for clear, well-recorded clips where the trumpet is foregrounded, and degrade noticeably for live ambient recordings or passages where piano is in the trumpet's range. There is no 2026 model that solves the trumpet-vs-piano interference problem cleanly on dense jazz.

## Sources

- [facebookresearch/demucs (htdemucs_6s)](https://github.com/facebookresearch/demucs)
- [HTDemucs Variant Comparison](https://stemsplitter.github.io/research/model-comparison/)
- [Banquet / query-bandit (GitHub)](https://github.com/kwatcharasupat/query-bandit)
- [Banquet paper (arXiv:2406.18747)](https://arxiv.org/abs/2406.18747)
- [Banquet HTML](https://arxiv.org/html/2406.18747v1)
- [Hyperellipsoidal Queries (arXiv:2501.16171)](https://arxiv.org/abs/2501.16171)
- [Bandit-v2 (GitHub)](https://github.com/kwatcharasupat/bandit-v2)
- [Bandit-v2 on MVSep](https://mvsep.com/algorithms/45)
- [LALAL.AI Wind & String stems](https://www.lalal.ai/blog/wind-string-instruments/)
- [LALAL.AI Perseus blog](https://www.lalal.ai/blog/perseus-ai-lalalai-transformer-neural-network/)
- [AudioShake instrument separation](https://www.audioshake.ai/instrument-stem-separation)
- [AudioShake Developers / API](https://developer.audioshake.ai/)
- [AudioShake real-time SDK](https://www.audioshake.ai/products/sdk)
- [Moises supported instruments](https://help.moises.ai/hc/en-us/articles/360010972019-Which-instruments-can-be-separated-on-Moises)
- [Moises Stems VST](https://moises.ai/features/stems-vst-plugin/)
- [Spleeter (Deezer GitHub)](https://github.com/deezer/spleeter/)
- [MVSep algorithms](https://mvsep.com/en/algorithms)
- [MVSep wind algorithm 61](https://mvsep.com/algorithms/61)
- [53-stem BS-Roformer thread](https://vi-control.net/community/threads/new-open-source-53-stems-audio-splitting-model-available.171774/)
- [ZFTurbo Music-Source-Separation-Training](https://github.com/ZFTurbo/Music-Source-Separation-Training)
- [Eddycrack864/Music-Source-Separation-Training (HF)](https://huggingface.co/Eddycrack864/Music-Source-Separation-Training/tree/main)
- [AudioSep (GitHub)](https://github.com/Audio-AGI/AudioSep)
- [AudioSep / "Separate Anything You Describe" (arXiv:2308.05037)](https://arxiv.org/abs/2308.05037)
- [DCASE 2024 Task 9 (LASS)](https://dcase.community/challenge2024/task-language-queried-audio-source-separation)
- [OmniSep (arXiv:2410.21269)](https://arxiv.org/abs/2410.21269)
- [OmniSep demo](https://omnisep.github.io/)
- [GuideSep (arXiv:2507.01339)](https://arxiv.org/abs/2507.01339)
- [GuideSep ISMIR 2025 page](https://ismir2025program.ismir.net/poster_147.html)
- [Source Separation of Small Classical Ensembles (arXiv:2505.17823)](https://arxiv.org/abs/2505.17823)
- [Jazz Trio Database (TISMIR)](https://transactions.ismir.net/articles/10.5334/tismir.186)
- [URMP dataset](https://labsites.rochester.edu/air/projects/URMP.html)
- [MoisesDB (ISMIR 2023)](https://archives.ismir.net/ismir2023/paper/000073.pdf)
- [Moises-Light (arXiv:2510.06785)](https://arxiv.org/html/2510.06785v1)
- [ACMID 7-stem dataset (arXiv:2510.07840)](https://arxiv.org/html/2510.07840)
- [Spheres orchestral dataset (arXiv:2511.21247)](https://arxiv.org/html/2511.21247v1)
- [BS-RoFormer (arXiv:2309.02612)](https://arxiv.org/abs/2309.02612)
- [Mel-Band RoFormer (arXiv:2310.01809)](https://arxiv.org/abs/2310.01809)
