# Commercial Transcription APIs and SaaS

## Summary
For the Play by Ear Helper pipeline (audio -> sheet music), three commercial offerings are clearly viable for production-grade integration: **Klangio API** (the only mature audio-to-notation API exporting MIDI/MusicXML/PDF/GP5 directly), **Music.ai** (the B2B/developer arm of Moises, offering granular per-minute pricing for chords, BPM, sections, and stem separation), and **AudioShake** (credit-based source separation and lyric transcription, useful as a pre-processor). For raw stem separation in front of the transcriber, **LALAL.AI** ($15/month Pro tier with API access) and **Music.ai** are the most developer-friendly. Consumer-tier tools (AnthemScore, ScoreCloud, Melody Scanner, Songscription) are mostly subscription/desktop products with no public REST API, although AnthemScore has a CLI suitable for server-side batch use. Expected developer pricing lands roughly in the range of $0.03-$0.20 per minute of audio for transcription tasks and around $0.10-$0.20 per minute for stem separation.

## Services

### Klangio (klang.io)
- URL: https://klang.io/api/ ; docs at https://api-docs.klang.io/
- API: Yes - REST API explicitly marketed at developers ("AI Music Analysis API"). Separate consumer products (Piano2Notes, Guitar2Tabs, Drum2Notes, Transcription Studio, Melody Scanner) sit on the same engine.
- Pricing:
  - Consumer subscriptions use a "ticket" system: Universe = 250 tickets/month, Single App = 50 tickets/month. Bundle subscription roughly $8.49/mo annual or $19.99/mo monthly.
  - API plans are separate and billed per recurring interval; specific public tiers are not posted - sales contact required. Overage either incurs charges or suspends the account per their docs.
- Supported instruments: piano, guitar, bass, vocals (also drums via Drum2Notes). Multi-instrument simultaneous transcription is the headline feature of Transcription Studio.
- Output formats: **MIDI, MusicXML, PDF, GP5 (Guitar Pro)**. This is the broadest output format coverage of any service surveyed.
- Accuracy: Klangio markets state-of-the-art accuracy but does not publish numbers; their Melody Scanner piano mode is cited at ~86% polyphonic accuracy.
- Licensing: Non-exclusive, non-transferable, non-sublicensable license during subscription. Klangio retains IP in the service. API output usage rights for commercial products require checking their commercial API agreement directly.
- Notes: Best fit as the core transcription engine. Handles audio -> notation end-to-end without a separate notation step.

### AnthemScore (Lunaverus)
- URL: https://www.lunaverus.com/
- API: No public REST API. Has a **command-line interface** (headless mode) that can output MusicXML and spectrogram data; supports batch processing of folders.
- Pricing: One-time perpetual license. Lite ~$32, Standard/Professional ~$57, Studio/Ultimate ~$107 (regional pricing varies, e.g. £25.59/£34.32). 30-day full-feature free trial.
- Supported instruments: General polyphonic transcription of MP3/WAV; not instrument-specific.
- Output formats: MusicXML, MIDI, PDF (via notation export), printable sheet music.
- Accuracy: Marketed as high; based on a CNN. No published number.
- Licensing: Single-user perpetual license; check EULA for headless server / commercial-service redistribution rights (typically not permitted without enterprise license).
- Notes: Cheapest option for self-hosted batch transcription if running on our own servers via CLI is acceptable. Not a true API/SaaS.

### ScoreCloud
- URL: https://scorecloud.com/
- API: No public API.
- Pricing: Free (10 saved songs, watermarked). Plus $4.99/mo (no watermark, MIDI export). Songwriter $10.99/mo. Pro $19.99/mo (MusicXML export, full features).
- Supported instruments: Singing/melody input, polyphonic audio import, MIDI input.
- Output formats: PDF, MIDI (Plus+), MusicXML (Pro).
- Accuracy: Not published; reviewed as decent for monophonic/lead-sheet use.
- Licensing: Consumer subscription; commercial/embedded use not advertised.
- Notes: End-user product; not suitable as a backend dependency.

### Melody Scanner
- URL: https://melodyscanner.com/ (a Klangio property: https://klang.io/melodyscanner/)
- API: No standalone API - shares the Klangio backend.
- Pricing: Free tier (1 minute, microphone/YouTube only, 40-bar limit). Premium subscriptions €6.99/mo to €47.99/yr. In-app purchases $4.99-$14.99.
- Supported instruments: Piano (most mature, ~86% polyphonic accuracy claimed), other instruments rougher.
- Output formats: MIDI, MusicXML, PDF (via Klangio export pipeline).
- Licensing: Consumer EULA.
- Notes: Effectively a consumer skin on Klangio - prefer Klangio API directly for our use case.

### Songscription AI
- URL: https://www.songscription.ai/
- API: Custom/team plans only - "Custom solutions are available for teams and organizations with API access." No self-serve developer tier.
- Pricing: Free (10 x 3-minute transcriptions/month). Plus $9.99/mo (5 x 6-minute). Pro $29.99/mo (100 x 15-minute).
- Supported instruments: General polyphonic, with specific guitar-tab generation. Stanford-trained models.
- Output formats: Sheet music, MIDI, guitar tabs, MusicXML.
- Accuracy: Marketed as "publication-ready"; MusicRadar review was mixed ("humans will be doing all the serious music transcription for the foreseeable future").
- Licensing: Consumer ToS at https://www.songscription.ai/terms.
- Notes: Newer entrant (launched 2025, backed by Reach Capital). API access requires sales contact - likely too early-stage to commit to.

### Noteshift AI
- URL: https://noteshift.ai/pricing
- API: No.
- Pricing: Free (20 one-time generations). Pro $15/mo (100/mo). Pro Plus $49/mo (500/mo).
- Notes: This is an AI music **composition/generation** product (text -> sheet music), not audio -> notation transcription. Not relevant for our pipeline.

### AudioShake
- URL: https://www.audioshake.ai/ ; developer portal https://developer.audioshake.ai/
- API: Yes - REST API plus a real-time SDK (recently launched).
- Pricing: Credit-based. 10 free credits on signup. Public references put one-off rates near **$1/minute**, with bulk discounts; Transcription + Alignment is "Premium" at **1.5 credits/minute**. Cents-per-minute rates available at scale via sales.
- Supported tasks: Stem separation (vocals, drums, bass, guitar, dialogue, music+SFX, etc.), lyric transcription with word-level alignment, content analysis. Not full music notation.
- Output formats: Audio stems (WAV), transcription/alignment JSON. No MusicXML/MIDI directly.
- Accuracy: Industry-leading for stem separation in post-production / karaoke.
- Licensing: B2B; commercial use is the explicit target market. Customers retain rights to their inputs/outputs per typical SaaS terms.
- Notes: Best used as a pre-processor (split stems before transcription) rather than the transcriber itself.

### LALAL.AI
- URL: https://www.lalal.ai/api/ ; pricing https://www.lalal.ai/pricing/
- API: Yes - API v1 with OpenAPI/Swagger docs, designed for batch and bulk workloads.
- Pricing: Free Starter tier (10 minutes). Lite $7.50/mo (90 fast minutes). **Pro $15/mo (250 fast minutes, includes API + VST access).** Pay-as-you-go and enterprise tiers available via support@lalal.ai.
- Supported tasks: Multi-stem separation - vocals, drums, bass, acoustic guitar, electric guitar, piano, synths, strings, wind instruments, plus voice/background noise. Voice cloning recently added.
- Output formats: Audio stems (WAV/MP3/FLAC).
- Accuracy: Generally regarded as top-tier for vocal isolation; benchmarks vs Moises/AudioShake are close.
- Licensing: Commercial use permitted on paid tiers per their API ToS.
- Notes: Strong, low-cost source separation API. Pair with Klangio or Music.ai for actual notation.

### Moises.ai (consumer) / Music.ai (developer)
- URLs: Consumer https://moises.ai/ ; Developer https://music.ai/ ; legacy dev portal https://developer-legacy.moises.ai/ ; API ref https://music.ai/docs/api/reference/
- API: Consumer Moises has limited/no public API; the **Music.ai** brand is the official developer/B2B platform with REST API, no-code Orchestrator UI, and an embedded SDK for local deployment.
- Pricing (Music.ai, per minute):
  - Stem separation: vocals (lead + backing) **$0.10/min**.
  - BPM: $0.03/min. Chord detection: $0.04/min. Sections: $0.04/min.
  - General transcription: $0.08/min. Lyric transcription: $0.17/min. Word/syllable alignment: $0.09/min. Diarization: $0.09/min. Auto language ID: $0.02/min.
  - Pay-as-you-go, no commitment.
- Pricing (Moises consumer): Free (5 tracks/month). Premium $3.99/mo (~$2.99/mo annual). Pro $9.99/mo (Hi-Fi separation, 180-min uploads, advertised "API" access).
- Supported tasks: Stem separation, chord detection, BPM, key (via add-ons), lyric transcription, sections. Music.ai is a "modules" marketplace - you compose pipelines.
- Output formats: Audio stems, JSON (chords, sections, BPM, lyrics with timestamps). No native MusicXML.
- Licensing: Music.ai is explicitly B2B with commercial-use terms.
- Notes: Cheapest fine-grained per-minute pricing of the surveyed APIs. Excellent for chord/BPM/structure analysis as inputs to a notation step. Doesn't produce MusicXML directly.

### Other AMT-as-a-service mentions
- **Melodyne (Celemony)** - desktop only, $99-$699 perpetual licenses; no API.
- **AudioScore Ultimate (Neuratron / Sibelius)** - desktop only; outputs MusicXML/MIDI; no API.
- **NutifAI** (https://nutif.ai/) - small AI transcription site; no documented API.
- **Music Demixer** (https://freemusicdemixer.com/) - in-browser demixer; not an API.

## Recommendation
1. **Evaluate Klangio API first** - it is the only commercial offering that returns MusicXML/MIDI/PDF directly for audio input across multiple instruments. Contact sales for API tier pricing and confirm commercial output rights.
2. **Pair with LALAL.AI ($15/mo Pro) or Music.ai ($0.10/min stems) as a pre-processing step** for stem separation when the audio is dense polyphonic mix - transcribers perform better on isolated stems.
3. **Use Music.ai for chord/BPM/section analysis** ($0.03-$0.04/min) to enrich the score (key signature, time signature, chord symbols above the staff) regardless of which transcriber renders the notes.
4. **Hold AnthemScore CLI as the self-hosted fallback** if API costs blow up or if data residency/privacy requires keeping audio off third-party servers; verify its license permits server-side batch use in our deployment.
5. Skip ScoreCloud, Melody Scanner, Songscription, and Noteshift for backend integration - they are consumer products without self-serve APIs.

## Sources
- Klangio API page: https://klang.io/api/
- Klangio API docs: https://api-docs.klang.io/
- Klangio subscriptions/tickets: https://klang.io/help/klangio-subscriptions/ , https://klang.io/help/tickets/
- Klangio export formats: https://klang.io/help/download-formats/
- Klangio Transcription Studio: https://klang.io/transcription-studio/
- Klangio Terms of Use: https://klang.io/terms/
- AnthemScore: https://www.lunaverus.com/ ; docs: https://www.lunaverus.com/documentation ; FAQ: https://www.lunaverus.com/faq
- ScoreCloud: https://scorecloud.com/ ; support: https://scorecloud.com/support/
- Melody Scanner: https://melodyscanner.com/ ; https://klang.io/melodyscanner/
- Songscription pricing: https://www.songscription.ai/pricing ; FAQ: https://www.songscription.ai/faq ; ToS: https://www.songscription.ai/terms
- MusicRadar Songscription review: https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review
- TechCrunch Songscription launch: https://techcrunch.com/2025/06/30/songscription-launches-an-ai-powered-shazam-for-sheet-music/
- Noteshift: https://noteshift.ai/ ; https://noteshift.ai/pricing
- AudioShake developer portal: https://developer.audioshake.ai/
- AudioShake stem separation legacy API: https://developer.audioshake.ai/legacy-api/server-to-server
- AudioShake transcription: https://developer.audioshake.ai/legacy-api/lyrics-transcription/transcription
- AudioShake SDK: https://www.audioshake.ai/products/sdk
- AudioShake FAQ: https://www.audioshake.ai/faq
- LALAL.AI API: https://www.lalal.ai/api/
- LALAL.AI pricing: https://www.lalal.ai/pricing/
- LALAL.AI business solutions: https://www.lalal.ai/business-solutions/
- LALAL.AI multi-stem + voice cloning expansion: https://www.newswire.com/news/lalal-ai-expands-its-developer-api-with-ai-multi-stem-separation-voice-22734584
- Moises consumer: https://moises.ai/
- Music.ai pricing: https://music.ai/pricing/
- Music.ai API reference: https://music.ai/docs/api/reference/
- Moises legacy developer platform: https://developer-legacy.moises.ai/
- Moises Extension API: https://extensions.moises.ai/api-reference
- Stem splitter benchmarks (StemSplit vs LALAL.AI vs Moises): https://dev.to/stevecase430/ai-stem-splitter-api-comparison-2026-stemsplit-vs-lalalai-vs-moises-with-benchmarks-372l
