# Music File Format Standards

## Summary
The Play by Ear Helper pipeline moves data through four representational layers: raw audio, AI-inferred performance data (MIDI-like), symbolic score (MusicXML or MEI), and a rendered visual (SVG/PNG via VexFlow/OSMD/Verovio). MIDI is the natural output of an AI transcription model but lacks the notational semantics (enharmonic spelling, beaming, voicing, staff splits, key/time signatures) needed to draw a readable score. **MusicXML 4.0 is the de facto interchange format** supported by 270+ notation programs and is the only realistic target for a 2026 product. **MNX** (the W3C JSON-based successor) is still a draft as of 2026 and not production-ready, while **MEI** is academically rich but requires extra transformation. **ABC notation** is useful as a lightweight text format for simple monophonic outputs, and **MuseScore's MSCZ/MSCX** is proprietary and unsuitable as an interchange target.

## Formats

### MIDI (Musical Instrument Digital Interface)
- **Captures:** Note on/off events, pitch (as integer 0-127, no enharmonic spelling), velocity, channel, tempo, basic time signature meta-events, program changes, control changes (pedal, mod wheel). It is fundamentally a *performance/playback* format.
- **Does NOT capture:** Enharmonic spelling (C# vs Db are identical), key signature (only as a hint meta-event), staff layout, voicing, beaming, stem direction, slurs, articulations as notation, dynamics as marks (only velocity), rests as explicit symbols, ties vs slurs distinction, lyrics alignment, page layout.
- **Tooling:** Universal — every DAW, every AI transcription model emits MIDI. Python: `mido`, `pretty_midi`, `music21`. JavaScript: `tone.js`, `midi-parser-js`. AI models (Klangio, AnthemScore, Spotify's Basic Pitch, Google's MT3) typically emit MIDI as their primary output.
- **Use case in our pipeline:** First-stage AI transcription output. We must enrich it heavily before rendering.

### MusicXML
- **Captures:** Notes with explicit pitch + accidental + octave (so D# vs Eb is preserved), key signatures, time signatures, clefs, beaming, stem direction, voicing (multiple `<voice>` per part), staves, dynamics (`<dynamics>` element with `<f>`, `<mf>`, `<other-dynamics>`, etc.), articulations (`<articulations>` with staccato, accent, tenuto), slurs, ties, tuplets, lyrics, chord symbols, layout hints (`default-x`, `default-y`, page breaks). Two encoding modes: `<score-partwise>` (organized by part then measure) and `<score-timewise>` (by measure then part).
- **Tooling:** Read/write supported by 270+ programs (MuseScore, Sibelius, Finale, Dorico, Notion). Python: `music21` (via `converter.parse()`), `partitura`, `muspy`. JS/Browser: `OpenSheetMusicDisplay` (renders MusicXML to SVG via VexFlow), `Verovio` (also accepts MusicXML and converts internally to MEI). Compressed variant `.mxl` is a zip of the XML + a META-INF manifest.
- **Use case in our pipeline:** **Primary symbolic output target.** This is what we hand to the renderer and what users export.

### MEI (Music Encoding Initiative)
- **Captures:** Everything MusicXML captures plus rich semantic/scholarly metadata — variant readings, editorial markup, manuscript provenance, neume/mensural notation for early music, hierarchical analytical annotations. XML-based, modular schema (TEI-like).
- **Tooling:** **Verovio** is the flagship engraver — fast, portable C++ with WASM build for browsers, renders MEI to SVG natively while preserving xml:id round-trips (you can click an SVG note and trace back to the MEI element). Python: `music21` (limited), `converter21` (full read/write). Most non-academic notation editors require MEI to be transformed to/from MusicXML first.
- **Use case in our pipeline:** Overkill for "play by ear" — its scholarly features (critical apparatus, editorial layers) aren't relevant to consumer transcription. Only worth considering if we adopt Verovio as the renderer and feed it MEI directly.

### ABC Notation
- **Captures:** Pitch (letters A-G with octave/accidental modifiers), durations, key signature (`K:`), time signature (`M:`), tempo (`Q:`), tune title/composer headers, chord symbols, basic ornaments, lyrics (`w:`). Originally for folk/traditional monophonic tunes; extended to support multi-voice (`V:`) and multi-staff but awkwardly.
- **Tooling:** `abcjs` (browser rendering, very lightweight ~200KB), `EasyABC` (desktop editor with MIDI export and SVG render), `abc2xml` (convert to MusicXML), `music21` import. Plain text, ~10x smaller than equivalent MusicXML.
- **Limitations:** Slow/breaks on large collections, no automatic page formatting, broken-rhythm markers undefined for unequal-length notes, weak for complex classical scores with multiple staves, dynamics, and detailed articulation.
- **Use case in our pipeline:** Possible secondary export for users wanting compact text; useful for a quick web preview if memory budget is tight. Not the primary symbolic format.

### MNX (W3C successor to MusicXML)
- **Captures:** A JSON document with a top-level `mnx` key, designed to be the next-generation interchange format. Builds on MusicXML's semantic model but enforces a *single canonical encoding* per passage (MusicXML allows many equivalent encodings, which has caused interoperability pain).
- **Status as of 2026:** **Still a Working Draft, not stable.** The W3C Music Notation Community Group (Adrian Holovaty, Daniel Spreadbury, Karim Ratib as co-chairs) holds bi-weekly meetings; recent activity in February-March 2026 covered ID character rules and finalizing the `.mnx.json` extension. Robert Patterson contributed an MNX import/export PR to MuseScore that was accepted, marking the first major notation editor to support MNX experimentally. There is no stable spec ratification date announced.
- **Tooling:** Experimental MuseScore branch, reference implementation in the W3C MNX repo. No mainstream Python or JS library reads MNX in production.
- **Use case in our pipeline:** **Watch but do not adopt yet.** Plan for future migration; emit MusicXML now.

### MuseScore MSCX / MSCZ
- **Captures:** Full MuseScore project including score, layout, custom palettes, instruments, embedded images, fonts, and (in `.mscz`) thumbnails. `.mscx` is XML; `.mscz` is a ZIP containing the MSCX plus images/JSON metadata.
- **Status:** **Proprietary and explicitly unstable.** MuseScore documentation states the format is "for MuseScore internal use" and "can change in every new MuseScore version" with no published spec.
- **Tooling:** Only MuseScore itself reads/writes natively. Some third-party importers reverse-engineer it.
- **Use case in our pipeline:** **Avoid as an interchange target.** Use MusicXML as the bridge to MuseScore.

### JSON-based formats
- **MNX** (covered above) is the only standards-track JSON music notation format.
- **Custom JSON envelopes:** Many ML transcription pipelines (Magenta NoteSequence, MAESTRO-style records) use protobuf or JSON for note lists `{pitch, start, end, velocity}`. These are essentially "MIDI in JSON" — equivalent expressive power, no notational semantics.
- **Tone.js / VexFlow** accept ad-hoc JSON for in-app rendering, but these are renderer-specific, not interchange formats.
- **Use case in our pipeline:** Internal API representation between AI service and notation service is a fine place for a custom JSON shape (essentially a NoteSequence). Don't expose it externally.

## MIDI → MusicXML Pipeline

The transcription model produces a **timed note list** (essentially MIDI). Turning that into a *readable score* is a separate, hard problem often called "MIDI-to-score" or "performance-to-score" transcription. The conversion must **infer** every notational semantic that MIDI omits:

| Inference Step | What's needed | Typical approach |
|---|---|---|
| **Beat / downbeat tracking** | Find the pulse and bar lines in the performance | Beat tracking model (madmom, librosa) or transformer trained on ASAP-style aligned data |
| **Time signature** | Group beats into bars (4/4, 3/4, 6/8, etc.) | Histogram of beat strengths; ML classifier; user confirmation |
| **Tempo** | BPM curve, separate from rhythmic content | Extract from beat tracker; smooth |
| **Quantization** | Snap performed onsets/durations to notation grid | Music21 default snaps to 16th or triplet-8th; transformer-based methods (arxiv 2604.22290) quantize to 1/24 fractions to cover 98.6% of ASAP notes |
| **Key signature** | Choose # or b key to minimize accidentals | Krumhansl-Schmuckler or learned classifier; affects enharmonic spelling |
| **Enharmonic spelling** | C# vs Db, F vs E# | Determined by key signature + voice-leading rules (e.g., raised 7th in minor) |
| **Voicing / staff split** | Which notes belong to which hand or voice | Pitch-based heuristic (split at C4 for piano), or hand-assignment ML model |
| **Beaming** | Group eighth/sixteenth notes by beat | Rule-based from time signature once durations are quantized |
| **Stem direction** | Up/down per note | Rule-based: middle line and below = up, above = down (or by voice) |
| **Rests** | Explicit silence symbols | Inferred from gaps within voice timelines |
| **Tie vs slur** | Same pitch across barline = tie; different pitches under one phrase = slur | MIDI gives neither; tie inferable from quantization, slur requires phrase model |
| **Dynamics** | f, mf, p marks vs MIDI velocity | Bucket velocities into mark categories; place at phrase boundaries |
| **Articulation** | Staccato, accent, tenuto | Compare actual note duration to nominal; high velocity = accent |

**What we lose going MIDI → MusicXML:** velocity granularity (collapses into discrete dynamic marks), micro-timing/expression, exact pedal curves, channel/program data unless we map to staff names.

**What we gain:** human-readable notation that can be displayed, edited, and re-exported.

**Recommended Python tooling:** `music21` for the round-trip skeleton (`converter.parse('foo.mid', quantizePost=True, quarterLengthDivisors=(4,3))` then `.write('musicxml')`); `partitura` for more aggressive performance-to-score with beat tracking integration; or a custom transformer model (Score Transformer, arXiv 2112.00355) for state-of-the-art end-to-end conversion. Quality benchmark to aim for: ~97% onset F1, ~83% note-value accuracy on ASAP-class material.

## Recommendation

For Play by Ear Helper:

1. **Internal API (AI → Notation service):** Custom JSON note-list (NoteSequence-style) — fast, simple, debuggable.
2. **Symbolic interchange / persistence / user export:** **MusicXML 4.0** (use the `.musicxml` extension; offer compressed `.mxl` for downloads). This is non-negotiable for interop with MuseScore/Sibelius/Dorico.
3. **Browser rendering:** **OpenSheetMusicDisplay** (MusicXML → VexFlow → SVG) for full-featured rendering. Consider **Verovio (WASM)** as an alternative if we want SVG with stable element IDs for click-to-edit interactions; it accepts MusicXML directly and is faster for large scores.
4. **Optional secondary export:** ABC text export for power users / clipboard sharing of simple monophonic transcriptions.
5. **MNX, MEI, MSCX:** Skip for v1. Re-evaluate MNX in 12-18 months once the spec stabilizes and library support catches up.
6. **Conversion engine:** Use `music21` for the heavy lifting (quantization, key inference, MusicXML output). Plan for upgrading to a learned MIDI-to-score model later as accuracy on free-tempo/expressive performances will be the limiting factor in user satisfaction.

## Sources

- [MusicXML 4.0 Specification - W3C](https://www.w3.org/2021/06/musicxml40/)
- [MusicXML official site](https://www.musicxml.com/)
- [MusicXML - Wikipedia](https://en.wikipedia.org/wiki/MusicXML)
- [The articulations element - MusicXML 4.0](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/articulations/)
- [The dynamics element - MusicXML 4.0](https://www.w3.org/2021/06/musicxml40/musicxml-reference/elements/dynamics/)
- [The MIDI-Compatible Part of MusicXML 4.0 Tutorial](https://www.w3.org/2021/06/musicxml40/tutorial/midi-compatible-part/)
- [MNX Specification - W3C Draft](https://w3c.github.io/mnx/docs/)
- [Comparing MNX and MusicXML](https://w3c.github.io/mnx/docs/comparisons/musicxml/)
- [W3C Music Notation Community Group](https://www.w3.org/community/music-notation/)
- [W3C Music Notation Co-chair meeting minutes February 10, 2026](https://www.w3.org/community/music-notation/2026/02/10/co-chair-meeting-minutes-february-10-2026/)
- [W3C Music Notation Co-chair meeting minutes March 24, 2026](https://www.w3.org/community/music-notation/2026/03/24/co-chair-meeting-minutes-march-24-2026/)
- [Change MNX to be a JSON format - GitHub Issue #290](https://github.com/w3c/mnx/issues/290)
- [Music Encoding Initiative homepage](https://music-encoding.org/about/)
- [MEI - Wikipedia](https://en.wikipedia.org/wiki/Music_Encoding_Initiative)
- [Verovio reference book](https://book.verovio.org/advanced-topics/internal-structure.html)
- [Verovio engraving library](https://www.verovio.org/index.xhtml)
- [ABC notation v2.1 standard](https://abcnotation.com/wiki/abc:standard:v2.1)
- [ABC notation - Wikipedia](https://en.wikipedia.org/wiki/ABC_notation)
- [ABC Notation home](https://abcnotation.com/)
- [MuseScore file formats handbook](https://musescore.org/en/handbook/3/file-formats)
- [MuseScore file format reference (music-notation.info)](http://www.music-notation.info/en/formats/MuseScore.html)
- [MSCZ File Format documentation](https://docs.fileformat.com/audio/mscz/)
- [music21 PyPI](https://pypi.org/project/music21/)
- [music21 converter module documentation](https://music21.org/music21docs/moduleReference/moduleConverter.html)
- [music21 subConverters source](https://github.com/cuthbertLab/music21/blob/master/music21/converter/subConverters.py)
- [converter21 - PyPI (Humdrum/MEI extensions)](https://pypi.org/project/converter21/)
- [OpenSheetMusicDisplay homepage](https://opensheetmusicdisplay.org/)
- [OpenSheetMusicDisplay GitHub](https://github.com/opensheetmusicdisplay/opensheetmusicdisplay)
- [Open Sheet Music Display: MusicXML introduction and comparison](https://opensheetmusicdisplay.org/blog/blog-music-xml-introduction-comparison/)
- [Transformer-Based Rhythm Quantization of Performance MIDI - arXiv 2604.22290](https://arxiv.org/abs/2604.22290)
- [End-to-End Piano Performance-MIDI to Score Conversion with Transformers - arXiv 2410.00210](https://arxiv.org/html/2410.00210v1)
- [Score Transformer: Generating Musical Score from Note-level Representation - arXiv 2112.00355](https://arxiv.org/pdf/2112.00355)
- [ASAP Dataset paper (ISMIR 2020)](https://archives.ismir.net/ismir2020/paper/000127.pdf)
- [Improving Automatic Music Transcription Through Key Detection](https://www.researchgate.net/publication/265545709_Improving_Automatic_Music_Transcription_Through_Key_Detection)
- [Klangio AI music transcription](https://klang.io/)
- [AnthemScore automatic transcription](https://www.lunaverus.com/)
- [Lessons from MusicXML Adoption - Michael Good](https://wpmedia.musicxml.com/wp-content/uploads/2012/11/xml2006-paper.pdf)
