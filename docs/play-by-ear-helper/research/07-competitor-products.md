# Competitor Product Landscape

## Summary
The audio-to-sheet-music space is fragmented across three tiers: (1) heritage desktop tools (AnthemScore, ScoreCloud, Neuratron) that are mature but creaky; (2) mobile-first AI apps (Klangio/Melody Scanner, Piano2Notes) that excel at solo instruments but struggle with full mixes; and (3) a fresh wave of 2025 AI startups (Songscription, Klangio Transcription Studio) racing to crack multi-instrument polyphonic transcription. Klangio currently has the broadest product portfolio and was first to market with a marketed "multi-instrument simultaneous" tool (mid-2025), while Songscription (TechCrunch-launched June 2025, "Shazam for Sheet Music") is the most credible new entrant with venture backing. The dominant user complaints across every product are the same: poor accuracy on polyphonic / multi-instrument / live recordings, paywalls hiding mediocre output, and rhythm/quantization errors that force heavy manual cleanup. Adjacent learning apps (Yousician, Simply Piano) and chord-only tools (Chordify, Hooktheory) are not direct competitors but shape user price expectations. The key gap: nobody yet offers reliable, per-instrument stem-separated transcription targeted at a *user-specified* set of instruments with clean, ready-to-play output.

## Products

### AnthemScore (Lunaverus)
- URL / Platform: https://www.lunaverus.com/ — Windows, Mac, Linux desktop; AnthemScore Web (browser subscription) also exists.
- Pricing: One-time purchase. Lite ~£25.59 (~$32), Professional ~£34.32 (~$43), Studio higher tier. 30-day free trial running as Professional. Web tier is subscription.
- What it does: AI-based automatic transcription of MP3/WAV to sheet music or guitar tab. Outputs notation with adjustable note-detection sensitivity (slider). Supports MIDI/MusicXML export.
- Strengths: One-time purchase model (rare in this space); cross-platform desktop; good for clean solo piano (~96% accuracy reported on clean piano); deep editing tools.
- Weaknesses: Struggles badly with polyphonic / full-band / orchestral audio; reviewers describe output as "rhythmically disorganized" with extra phantom notes; live recordings poorly handled; UI is dated; no mobile app.

### ScoreCloud (DoReMIR)
- URL / Platform: https://scorecloud.com/ — Desktop (ScoreCloud Studio, Win/Mac), iOS app, web.
- Pricing: Freemium. Plus $4.99/mo, Songwriter $10.99/mo, Pro $19.99/mo. Free tier includes real-time transcription + cloud sync.
- What it does: Real-time and file-based audio-to-notation. Best at MIDI input and monophonic instruments. Songwriter-oriented — sing or play and get a score.
- Strengths: Real-time transcription is fast and impressive for MIDI/keyboard input; cloud-syncs across devices; songwriter workflow is well-designed.
- Weaknesses: Vocal pitch transcription unforgiving (lots of accidentals if singer is off-pitch); polyphonic audio file transcription weaker than competitors; subscription pricing.

### Klangio — Melody Scanner & Transcription Studio
- URL / Platform: https://klang.io/ — iOS, Android, Web, plus DAW plugin (Transcription Plugin). Multiple sub-products: Melody Scanner, Piano2Notes, Guitar2Tabs, Voice2Notes, Drums2Notes, and the new Transcription Studio.
- Pricing: Freemium. Melody Scanner ~$10/mo, with promo annual at $34.99/yr. Transcription Studio ~£4/mo (promo) up to $19.99/mo. Free demos with limited length (typically 20-30 seconds).
- What it does: Upload audio, paste YouTube link, or record live. Produces PDF sheet music, MIDI (quantized & unquantized), MusicXML, GuitarPro. Transcription Studio (launched 2025) is marketed as "world's first" multi-instrument simultaneous AI transcription per MusicRadar.
- Strengths: Broadest product surface area in the category; mobile-first; multiple instrument-specific apps; recent multi-instrument tool is a real differentiator; strong export format coverage.
- Weaknesses: Melody Scanner Android rating only 3.34/5 (3.1k ratings, 830k downloads); users complain it's accurate only for slow/simple songs; cannot separate simultaneous instruments well in solo apps; complex audio yields "mishmash of notes"; paywall before users can verify quality; UI bugs reported (saved files losing translated letter notation).

### Piano2Notes (Klangio, technology originally ByteDance)
- URL / Platform: https://piano2notes.klang.io/ — iOS app, web. (The underlying ByteDance piano transcription model is open source on GitHub: bytedance/piano_transcription — PyTorch, MAESTRO-trained.)
- Pricing: Free demo of first 20 seconds; subscription beyond that (rolled into Klangio bundles).
- What it does: Piano-only audio → MIDI + MusicXML + sheet music. Built on ByteDance's high-resolution piano transcription (detects onsets, offsets, velocities, pedal).
- Strengths: ByteDance model is research-grade, ~SOTA for solo piano transcription (note onset/velocity/pedal); good for piano students.
- Weaknesses: Solo piano only — no multi-instrument; mobile UX limited; demo cap of 20 seconds frustrates evaluation.

### PhotoScore & NotateMe (Neuratron)
- URL / Platform: https://www.neuratron.com/photoscore.htm — Desktop (Win/Mac), tablet apps. Bundled with Avid Sibelius.
- Pricing: Desktop Ultimate ~$250 one-time; NotateMe app + PhotoScore in-app purchase ~$70. Lite version free with Sibelius.
- What it does: OMR (optical music recognition) — scans printed/PDF sheet music and converts to editable notation. Different from audio transcription; included for landscape completeness.
- Strengths: 99.5%+ accuracy on most clean PDFs; long-established; tight Sibelius integration; handles handwritten notation via touch/stylus.
- Weaknesses: Not audio transcription — different problem; expensive; aging product; requires already-existing sheet music as input.

### Soundslice
- URL / Platform: https://www.soundslice.com/ — Web-based.
- Pricing: Free tier (YouTube transcription only); paid Plus plan unlocks MP3/video uploads.
- What it does: Primarily a "living sheet music" platform — sync sheet music to audio/video for practice (slow-down, looping, mute parts). Has audio-aware transcription tools and an AI-powered sheet-music-photo scanner. More score-sharing than auto-transcription.
- Strengths: Best-in-class learning/practice UX; loops snap to beats/notes; instant transposition; clean web app; AI-trained sheet-music scanner.
- Weaknesses: Primary value is manual transcription assistance, not full automatic audio-to-notation; small audience compared to learning apps.

### GarageBand & Logic Pro (Apple)
- URL / Platform: macOS / iOS desktop DAW. Logic Pro $199.99 one-time (Mac) or $4.99/mo (iPad); GarageBand free.
- What it does: Logic Pro's Flex Pitch can extract MIDI from monophonic audio (vocal lines, leads, bass). GarageBand has Flex Time but not Flex Pitch — no audio-to-MIDI.
- Strengths: Logic's Flex Pitch is reliable for monophonic vocal/lead → MIDI conversion within an existing pro DAW workflow; included if user already owns Logic.
- Weaknesses: Mac-only; Logic is a full DAW (heavy install for casual transcribers); monophonic only — won't handle chords or full mixes; output is MIDI piano-roll, not engraved sheet music; requires DAW expertise.

### Chordify
- URL / Platform: https://chordify.net/ — Web, iOS, Android.
- Pricing: Free tier (limited); Premium ~$3.49/mo annual / $6.99/mo monthly.
- What it does: Chord-only auto-detection from YouTube/SoundCloud/Deezer/uploads. Displays chord chart synchronized to playback.
- Strengths: Massive song catalog via YouTube; cheap; great for guitar/keyboard busking and casual learning; long-running brand.
- Weaknesses: Chords only — no melody, no notation; accuracy mixed on complex/jazz songs; doesn't transcribe individual instruments; no MusicXML/sheet music output.

### Hooktheory / Hookpad
- URL / Platform: https://www.hooktheory.com/ — Web.
- Pricing: Free tier; Standard $7.99/mo or $199 lifetime (also $4.99/mo / $49/yr tiers historically); Aria AI add-on $14.99/mo.
- What it does: Songwriting / chord-progression tool with theory analysis. Has a database of 40,000+ analyzed songs (community-transcribed chords + melody). Not auto-transcription from audio — analysis of curated/manually entered songs.
- Strengths: Excellent music-theory pedagogy; chord-progression suggestions backed by analyzed song corpus; lifetime pricing option.
- Weaknesses: Not an audio-input transcription tool; songs must already be in their database or manually entered; songwriting-focused, not transcription-focused.

### Riffstation (DEFUNCT)
- URL / Platform: Discontinued. iOS/web shut down May 2018; desktop made free, then fully discontinued early 2019 (after Fender acquisition).
- Notes: Created a clear gap in the chord-detection-with-key/tempo space that Chordify, ChordU, and Song Surgeon have partially filled.

### Yousician / Simply Piano (Adjacent — learning, not transcription)
- URL / Platform: https://yousician.com/ ; https://www.joytunes.com/simply-piano — iOS, Android, web.
- Pricing: Yousician Premium $89.99/yr, Premium+ $139.99/yr, family $209.99/yr; Simply Piano ~$119–$179/yr individual, $209.90/yr family.
- What it does: Real-time pitch detection while user plays along with pre-existing sheet music for gamified learning. NOT transcription.
- Strengths: Polished consumer apps; large song libraries (licensed); strong gamified pedagogy; massive user bases.
- Weaknesses: Not transcription products — they consume existing sheet music. Sets a high consumer-UX bar that transcription tools have not matched. Pricing teaches users that music apps cost ~$10–$20/mo.

### Songscription (2025 Startup)
- URL / Platform: https://www.songscription.ai/ — Web (mobile app implied/forthcoming). Backed by Reach Capital and Stanford StartX.
- Pricing: Free (10 × 3-min transcriptions/mo), Plus $9.99/mo (5 × 6-min), Pro $29.99/mo (100 × 15-min).
- What it does: Upload MP3/WAV/M4A or paste YouTube/Instagram/TikTok link → sheet music PDF, MIDI, MusicXML, Guitar Pro tabs. Supports piano, guitar, bass, violin, flute, trumpet, sax, drums, vocals. Built-in editor and piano roll.
- Strengths: Most modern UI in the category; broadest instrument list at launch; investor-backed with stem-separation and arrangement features on roadmap; good "Shazam for sheet music" positioning.
- Weaknesses: New product (June 2025) — accuracy not yet proven at scale; MusicRadar's review headline: *"Humans will be doing all the serious music transcription for the foreseeable future"* — suggesting current quality is not pro-grade; usage caps tight on free/Plus tiers.

### Other notable mentions
- **Transcribe! (Seventh String)** — Veteran desktop slow-down/loop tool, manual transcription aid, $39 one-time. No auto-transcription.
- **Capo (Supermegaultragroovy)** — macOS/iOS, audio analysis with chord detection and slow-down. Manual-assist focus.
- **Song Surgeon** — Desktop, automatic chord/key/BPM detection with slow-down. Often cited as Riffstation replacement.
- **ChordU** — Free YouTube chord auto-detection (web). Direct Chordify competitor.
- **Melodyne (Celemony)** — DCC-grade pitch editor; can extract MIDI from polyphonic audio (DNA Direct Note Access). Pro-tier pricing (~$99–$849).
- **MIDI Agent** — Newer audio-to-MIDI plugin for Logic.
- **MT3 (Google Magenta)** — Open-source research model for multi-instrument transcription; not a product but underlying tech reference.
- **Neural Note** — Free audio-to-MIDI VST.

## Market Gap & Positioning

**Gaps identified across the field:**

1. **User-specified target instrumentation.** No competitor lets the user say "I want piano + cello parts, ignore the drums and vocals." Everyone either dumps everything into one staff or transcribes a single instrument. *This is the Play by Ear Helper sweet spot.*

2. **Reliable polyphonic / multi-instrument separation.** AnthemScore and Melody Scanner explicitly fail here per user reviews. Klangio Transcription Studio and Songscription claim it but reviews suggest output still needs heavy cleanup. A product that does this *well* would be unique.

3. **Engraving quality / playable output.** Most tools produce notation that's "raw MIDI dumped on a staff" — non-musical rhythms, no phrasing, no dynamics, no fingering. Output that a real musician can sight-read without 30 minutes of cleanup is rare.

4. **Honest free tier.** Almost every product locks accuracy testing behind a paywall, then users complain after paying. A generous free tier with full-quality short clips would build trust.

5. **No dominant mobile-first product.** Melody Scanner is the closest but rated only 3.34/5. Songscription is web-only at launch. A polished, accurate iOS/Android app is open territory.

6. **Sheet-music-as-output, not just MIDI.** Many tools' real strength is MIDI/piano-roll; the engraved-PDF output is an afterthought. Musicians-who-read-music are underserved.

**Positioning recommendations for Play by Ear Helper:**

- Lead with **"choose your instruments"** as the hero feature — frame it as the only tool that respects the user's actual needs (e.g. "I'm a violinist learning the lead line — I don't care about the drums").
- Compete on **accuracy honesty**: publish accuracy benchmarks vs. AnthemScore/Songscription/Klangio on standardized test clips.
- Target the **$5–$10/mo price band** — below Songscription Pro, in line with Chordify Premium and Melody Scanner.
- Differentiate from learning apps (Yousician/Simply Piano) by being explicitly **transcription-first**, not gamified practice.
- Avoid the "one staff dump" failure mode — invest in **per-stem separation + per-instrument engraving** as core architecture.

## Sources

- [AnthemScore — Lunaverus](https://www.lunaverus.com/)
- [AnthemScore Pricing](https://www.lunaverus.com/transcribe/pricing)
- [AnthemScore Review (AI Product Reviews)](https://ai-productreviews.com/anthemscore-by-lunaverus-review/)
- [AnthemScore Reviews 2025 (aitools.xyz)](https://aitools.xyz/tools/anthemscore/reviews)
- [ScoreCloud — Free Music Notation Software](https://scorecloud.com/)
- [ScoreCloud Reviews (OpenTools, April 2026)](https://opentools.ai/tools/scorecloud)
- [DoReMIR ScoreCloud — Sound on Sound review](https://www.soundonsound.com/reviews/doremir-scorecloud)
- [Klangio — klang.io](https://klang.io/)
- [Melody Scanner — klang.io](https://klang.io/melodyscanner/)
- [Melody Scanner — App Store](https://apps.apple.com/us/app/melody-scanner/id6472921068)
- [Melody Scanner — Google Play](https://play.google.com/store/apps/details?id=com.melodyscanner.app)
- [Klangio Transcription Studio](https://klang.io/transcription-studio/)
- [Klang.io launches Transcription Studio — MusicRadar](https://www.musicradar.com/music-tech/klang-io-says-transcription-studio-is-the-worlds-first-ai-music-tool-that-can-transcribe-multiple-instruments-simultaneously)
- [Piano2Notes — klang.io](https://klang.io/piano2notes/)
- [bytedance/piano_transcription — GitHub](https://github.com/bytedance/piano_transcription)
- [PhotoScore — Neuratron](https://www.neuratron.com/photoscore.htm)
- [PhotoScore & NotateMe Lite — Sibelius](https://www.sibelius.com/products/photoscore/lite.html)
- [Neuratron PhotoScore & NotateMe Ultimate 2020 — Sound on Sound](https://www.soundonsound.com/reviews/neuratron-photoscore-notateme-ultimate-2020)
- [Soundslice — Transcribe](https://www.soundslice.com/transcribe/)
- [Soundslice Sheet Music Scanner](https://www.soundslice.com/sheet-music-scanner/)
- [Logic Pro — MIDI from Audio using Flex Pitch (Apple)](https://support.apple.com/guide/logicpro/create-midi-from-audio-recordings-lgcpe2fd1b83/mac)
- [Audio-to-MIDI tool comparison — Arranger For Hire](https://arrangerforhire.com/we-compared-automatic-audio-to-midi-transcription-tools-to-aural-transcription-by-a-music-arranger/)
- [Chordify Premium](https://chordify.net/premium)
- [Chordify Review — Guitar Chalk](https://www.guitarchalk.com/chordify-review/)
- [Chordify on Trustpilot](https://www.trustpilot.com/review/chordify.net)
- [Hookpad Pricing — Hooktheory](https://www.hooktheory.com/hookpad/pricing)
- [Hookpad Review — Produce Like A Pro](https://producelikeapro.com/blog/hooktheory-hookpad-review/)
- [What replaced Riffstation?](http://mybubbaandme.com/common-questions/what-replaced-riffstation/)
- [Fender Riffstation Pro is now free — TechRadar](https://www.techradar.com/news/fenders-riffstation-pro-is-now-free-get-the-chords-for-any-song-on-your-desktop)
- [Yousician](https://yousician.com/)
- [Yousician Plans](https://account.yousician.com/plans)
- [Simply Piano review — Pianoers](https://pianoers.com/simply-piano-review-the-honest-truth-about-learning-piano-with-an-app/)
- [Songscription AI](https://www.songscription.ai/)
- [Songscription Pricing](https://www.songscription.ai/pricing)
- [Songscription launches 'Shazam for sheet music' — TechCrunch](https://techcrunch.com/2025/06/30/songscription-launches-an-ai-powered-shazam-for-sheet-music/)
- [Songscription review — MusicRadar](https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review)
- [Songscription — Music Ally](https://musically.com/2025/07/01/songscription-uses-ai-to-automate-sheet-music-transcription/)
- [2025 Automatic Music Transcription Challenge](https://ai4musicians.org/transcription/2025transcription.html)
- [YourMT3+ — multi-instrument transcription paper (arXiv)](https://arxiv.org/html/2407.04822v1)
