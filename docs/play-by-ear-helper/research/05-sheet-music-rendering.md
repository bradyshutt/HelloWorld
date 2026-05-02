# Sheet Music Rendering Libraries

## Summary
For a web/mobile-first Play by Ear Helper, the strongest open-source picks are **OpenSheetMusicDisplay (OSMD)** for high-level MusicXML rendering (built on VexFlow, MIT-licensed, easy React integration) and **Verovio** as a more typographically polished alternative that also handles MusicXML and emits MIDI for playback. **VexFlow** is the right choice if we need fine-grained control over engraving primitives, while **abcjs** is attractive only if we choose ABC notation as our intermediate format. Commercial SDKs like **Flat.io** and **Soundslice** offer turnkey playback/editor experiences but introduce per-view or subscription costs and vendor lock-in. The recommended pipeline is AMT-output MIDI -> `music21` (Python) for quantization and MusicXML export -> OSMD or Verovio for browser rendering, with MIDI playback via Tone.js or the built-in Verovio MIDI emitter.

## Libraries

### VexFlow
- URL: https://www.vexflow.com/ ; https://github.com/vexflow/vexflow
- License: MIT
- Formats: Native API only (JS/TS object model). No built-in MusicXML/MIDI/ABC import. Outputs SVG and HTML Canvas.
- Interactivity: Low-level engraving primitives. No built-in playback or editing; you compose measures/notes by hand in code.
- Maintenance: Very active. v5.0.0 released March 2025; ~4,500 commits on main; written in TypeScript.
- Mobile: Renders SVG/Canvas in any modern mobile browser; no native iOS/Android wrapper from upstream.
- Notes: Best as the rendering engine underneath a higher-level wrapper. Most "MusicXML for VexFlow" forks are abandoned; OSMD is the canonical VexFlow-based MusicXML pipeline.

### OpenSheetMusicDisplay (OSMD)
- URL: https://opensheetmusicdisplay.org/ ; https://github.com/opensheetmusicdisplay/opensheetmusicdisplay
- License: BSD-3-Clause (project page also references MIT-style permissive distribution)
- Formats: MusicXML (.xml, .mxl) input. Renders to SVG via VexFlow. No MIDI/ABC import.
- Interactivity: Renderer, not editor. Supports cursor/playback-position highlighting, note-coloring via SVG manipulation, and tablature display. Pairs with external playback engines (e.g., osmd-audio-player).
- Maintenance: Actively maintained by PhonicScore; available on npm as `opensheetmusicdisplay`. Native React/Kotlin/Swift wrappers available in early access.
- Mobile: Works in mobile browsers; `react-native-opensheetmusicdisplay` exists on npm. Also has a WordPress Gutenberg block.
- Notes: Pragmatic choice if AMT output is converted to MusicXML. Cannot easily edit/move notes after render.

### abcjs
- URL: https://www.abcjs.net/ ; https://github.com/paulrosen/abcjs
- License: MIT
- Formats: ABC notation input. Can export MIDI; imports only ABC (no MusicXML).
- Interactivity: SVG output with class hooks; built-in MIDI playback (`abcjs-midi`), transposition, chord grids, and limited animation. String tablature supported.
- Maintenance: Active; maintained by Paul Rosen on npm as `abcjs`.
- Mobile: Pure-JS SVG, runs in any browser. No native mobile module.
- Notes: Only useful if we adopt ABC notation as the intermediate format. ABC is concise and easy to hand-edit but has weaker support for complex scores than MusicXML/MEI.

### Verovio
- URL: https://www.verovio.org/ ; https://github.com/rism-digital/verovio
- License: LGPL v3
- Formats: Native MEI; on-the-fly conversion from MusicXML, Humdrum, **ABC**, Plaine & Easie, EsAC, MuseData. Outputs SVG and MIDI (`renderToMIDI()` returns base64 MIDI for playback).
- Interactivity: SVG output is element-addressable; supports cursor sync with MIDI for highlighting. No editor.
- Maintenance: Very active; backed by Swiss RISM Office and Swiss National Science Foundation. Distributed via npm (`verovio`) as a WebAssembly build. Recommended for React/Vue/Vite apps.
- Mobile: WASM works in modern mobile browsers; iOS Swift bindings (CocoaPod) and Java/Python/Go bindings exist.
- Notes: Stronger engraving aesthetics than OSMD; built-in MIDI output simplifies playback. LGPL means we can use it from a closed-source app via dynamic linking, but redistribution of modifications must remain LGPL.

### Flat.io Embed SDK
- URL: https://flat.io/developers ; https://github.com/FlatIO/embed-client
- License: Commercial. Free embed available with Flat branding; paid plan to remove branding, customize controls/theme, or use commercially. Custom pricing for e-commerce/high-volume (contact embed@flat.io).
- Formats: MusicXML, MIDI, MXL, GuitarPro import; renders inside an iframe.
- Interactivity: Full SDK with 60+ methods (load/export, transport control, cursor, edit). Includes a playable + editable embed.
- Maintenance: Active commercial product (`flat-embed` on npm).
- Mobile: Iframe runs in mobile browsers; full editor designed for desktop UX.
- Notes: Fastest path to a polished editor experience, but cost and lock-in are non-trivial.

### Soundslice
- URL: https://www.soundslice.com/ ; player API: https://www.soundslice.com/help/en/embedding/javascript-api/38/introduction/
- License: Commercial. Each free account gets one free embed; commercial licensing is **usage-based, billed monthly per unique pageview**. Contact required for actual rates.
- Formats: Imports MusicXML, MIDI, GuitarPro; supports synchronized audio/video alignment.
- Interactivity: JS API to control playback, speed, source switching, looping. Strong "interactive lesson" use case (audio + score sync).
- Maintenance: Active commercial product.
- Mobile: Responsive iframe; designed for mobile practice usage.
- Notes: Best fit if we want to sync the rendered score with the original audio clip the user uploaded.

### MuseScore (desktop) + webmscore
- URL: https://musescore.org/ ; https://www.npmjs.com/package/webmscore ; oEmbed: https://developers.musescore.com/
- License: MuseScore desktop is GPL v3. MuseScore.com embed via oEmbed (proprietary, free for personal embedding of public scores). `webmscore` (libmscore in WASM) is GPL v3.
- Formats: Native MSCZ/MSCX, MusicXML, MIDI, GuitarPro. `webmscore` can export MusicXML, MIDI, WAV/OGG/MP3/FLAC, PDF, SVG.
- Interactivity: oEmbed iframe is read-only with playback. `webmscore` can render in a Web Worker and produce audio.
- Maintenance: Very active.
- Mobile: oEmbed renders in mobile browsers. `webmscore` runs in any modern JS runtime (Web Worker, Node).
- Notes: GPL v3 forces our app to be GPL v3 if we link `webmscore` directly into client code. Use only if we are open-sourcing under a compatible license, or use the MuseScore.com oEmbed iframe for hosted scores.

### React/Vue components
- `react-opensheetmusicdisplay` (community wrapper around OSMD).
- `react-native-opensheetmusicdisplay` on npm (mobile RN wrapper).
- Verovio ships ES modules importable directly into React/Vue/Vite projects.
- `@music-i18n/musicxml-player` orchestrates rendering (Verovio or OSMD) plus Web Audio playback.
- `flat-embed` is framework-agnostic and works in any React/Vue app.

## MIDI -> Sheet Music Conversion

AMT models (e.g., basic-pitch, MT3, Onsets-and-Frames) typically emit MIDI. Browser sheet music libraries want MusicXML or MEI, so a conversion step is needed.

**Recommended pipeline:**
1. **AMT output -> raw MIDI** (note on/off, velocities, sometimes pitch bends).
2. **Quantization & voicing**: MIDI from real audio is unquantized and lacks enharmonic spelling. Quantize note onsets/durations to a beat grid and infer key signature.
3. **MIDI -> MusicXML**: convert the quantized representation.
4. **Render**: load MusicXML in OSMD or Verovio; offer MIDI playback via Tone.js, Web Audio, or Verovio's `renderToMIDI()`.

**Tools:**
- **music21** (Python, BSD): `converter.parse('file.mid')` then `.write('musicxml')`. Has tunable quantization (`quantizePost`, `quarterLengthDivisors`). Best on machine-generated MIDI; live-recording MIDI needs more cleanup. Docs: https://music21.org/
- **MuseScore CLI / webmscore**: opens MIDI and exports MusicXML; applies its own notation cleanup heuristics. GPL v3.
- **midixmljs** (npm): pure-JS MIDI <-> MusicXML, but README marks it as in-development / not production-ready. https://github.com/turnerhayes/midixmljs
- **musicxml-midi** (infojunkie): MusicXML-to-MIDI plus accompaniment generator; useful for the reverse direction (playback). https://github.com/infojunkie/musicxml-midi
- **MusPy** (Python): symbolic music I/O bridging MIDI, MusicXML, ABC, and music21.
- **pretty_midi** (Python): convenient MIDI manipulation/quantization before handoff to music21.

**Practical recommendation:** run a Python service (music21 + pretty_midi) for the MIDI -> MusicXML hop, since the JS-native options are immature; ship the resulting MusicXML to the browser for rendering.

## Recommendation

- **Primary renderer:** **OpenSheetMusicDisplay** for fastest path to MusicXML rendering with broad community support and a React wrapper, or **Verovio** if we want better engraving and built-in MIDI playback. Both are viable; Verovio's LGPL is acceptable for a closed-source product when used as a library.
- **Avoid GPL-only paths** (`webmscore` linked into client) unless we open-source the app.
- **Fallback / premium:** evaluate **Flat.io** if we need an in-browser editor (user can correct AMT mistakes), or **Soundslice** if synchronized audio+score practice is a core feature.
- **Conversion service:** Python microservice using `music21` (with `pretty_midi` preprocessing) to convert AMT MIDI -> MusicXML; cache MusicXML output and serve to the renderer.
- **Playback:** Tone.js or Verovio's MIDI export to produce browser audio aligned with cursor highlighting.

## Sources
- VexFlow: https://www.vexflow.com/
- VexFlow GitHub: https://github.com/vexflow/vexflow
- OSMD home: https://opensheetmusicdisplay.org/
- OSMD GitHub: https://github.com/opensheetmusicdisplay/opensheetmusicdisplay
- OSMD npm: https://www.npmjs.com/package/opensheetmusicdisplay
- "Reasons we built OSMD on VexFlow": https://opensheetmusicdisplay.org/blog/reasons-javascript-engine-musicxml-vexflow/
- "List of Sheet Music Display Libraries": https://opensheetmusicdisplay.org/blog/sheet-music-display-libraries-browsers/
- abcjs: https://www.abcjs.net/
- abcjs GitHub: https://github.com/paulrosen/abcjs
- Verovio: https://www.verovio.org/
- Verovio GitHub: https://github.com/rism-digital/verovio
- Verovio reference book: https://book.verovio.org/
- Verovio JS/WASM guide: https://book.verovio.org/installing-or-building-from-sources/javascript-and-webassembly.html
- Verovio MIDI output: https://book.verovio.org/interactive-notation/playing-midi.html
- Flat.io developer docs: https://flat.io/developers
- Flat embed client: https://github.com/FlatIO/embed-client
- Flat embed v2 announcement: https://blog.flat.io/embed-sheet-music-music-notation-sdk-v2/
- Soundslice plans: https://www.soundslice.com/plans/
- Soundslice licensing: https://www.soundslice.com/licensing/
- Soundslice JS API: https://www.soundslice.com/help/en/embedding/javascript-api/38/introduction/
- MuseScore developers: https://developers.musescore.com/
- webmscore: https://www.npmjs.com/package/webmscore
- music21 converter docs: https://music21.org/music21docs/moduleReference/moduleConverter.html
- music21 PyPI: https://pypi.org/project/music21/
- midixmljs: https://github.com/turnerhayes/midixmljs
- musicxml-midi: https://github.com/infojunkie/musicxml-midi
- musicxml-player: https://github.com/infojunkie/musicxml-player
- MusPy: https://hermandong.com/muspy/doc/muspy.html
- react-native-opensheetmusicdisplay: https://libraries.io/npm/react-native-opensheetmusicdisplay
