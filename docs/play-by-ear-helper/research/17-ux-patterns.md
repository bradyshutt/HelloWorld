# UX Patterns from Existing Apps

## Summary
The strongest existing-app patterns to adopt are: (1) a single big "record / upload / paste link" entry point that defers all configuration until *after* the clip is captured, (2) an explicit confidence channel in the rendered score (AnthemScore's white "candidate notes" + likelihood slider is the gold standard), (3) a moving playhead synced to scrolling notation (Soundslice / Chordify) so the user can verify by ear immediately, (4) progressive disclosure of paid features — let the user transcribe a free demo clip end-to-end before hitting the paywall (Klangio, Ivory, Songscription all do this), and (5) a built-in editor with audio scrubbing because *every* AI transcription needs corrections and reviews punish apps that ship results as read-only. The dominant failure modes to avoid are: opaque multi-minute waits with no feedback, hiding fundamental playback behind a paywall, and lossy export (PDF only, no MIDI/MusicXML).

## App Walkthroughs

### Klangio / Melody Scanner
Flow: Home -> choose source (Record / Upload audio / Paste YouTube link) -> pick a *mode* (Lead Sheet, Arrangement, or Universal) -> AI transcription runs in the cloud (~10-30s for short clips, longer for full songs) -> result opens in a built-in editor with three view types: classical notation, piano roll, and guitar tab. Users can correct notes inline, then export to PDF, MIDI (quantized + unquantized), MusicXML, or GuitarPro. Free demo transcriptions are unlimited but truncated/watermarked; full-length export and Edit Mode require Pro. Reviews praise the speed and three-view toggle but complain about (a) accuracy on polyphonic / multi-instrument input, (b) features being aggressively gated behind the paywall, and (c) playback timing drift after transcription.

### AnthemScore (Lunaverus)
Desktop-only (Win/Mac/Linux), one-time purchase rather than subscription. The main window is a *spectrogram* — a color heatmap of frequency over time — with detected notes overlaid as blue rectangles labeled with pitch. The killer UX feature: "candidate notes" rendered in **white** indicate spots where the CNN was uncertain. A **likelihood slider** in the side panel lets the user drag toward + or - to globally promote/demote candidates into committed notes; a separate **threshold slider** tunes detection sensitivity, and right-clicking individual piano keys lets you set per-pitch thresholds. This makes uncertainty an interactive control, not just a visual hint. Reviewers say the interface is powerful but overwhelming for beginners.

### ScoreCloud (Express + Studio)
Express (mobile, EuroBest 2013 Gold for UX) is built around "sing/whistle/hum -> get notation". One big record button, monophonic capture, immediate notation output in standard staff form with auto-detected key/time signature/tempo. Studio (desktop) supports MIDI input directly, but **microphone input is iOS-only and paid** — a frequent reviewer complaint. Accuracy is strong on clean monophonic input, weak on pitch-drift singing (lots of accidentals to fix). Editing happens in the desktop app; mobile sessions sync via cloud account. Common complaint: time-signature mis-detection requires a manual fix.

### Chordify
Web + mobile. User pastes a YouTube/Spotify/SoundCloud URL or uploads audio. Output is a grid of chord boxes (no staff notation), with a black square **playhead** that steps from chord to chord in sync with the audio. Practice features: BPM slowdown, capo/transpose, looping. The newer "Chords & Lyrics" view shows lyrics scrolling in real time alongside chords. MIDI piano "Chords audio" track plays the simplified harmonic skeleton so users can hear what the chords *should* sound like even if they can't play yet. Praised for being approachable; criticized for paywalled features (transpose, MIDI export) and metronome lag at extreme BPM.

### Yousician / Simply Piano
Both use device microphone (or MIDI) to **listen in real time** while the user plays along to a scrolling score. The UI scrolls notation horizontally with a fixed playhead; correctly played notes are colored green, missed/wrong notes red, and rhythm errors flash. Yousician advertises a polyphonic engine so chords can be evaluated. Skill-tree onboarding: short interactive lessons, gamified streaks, leaderboards. Both gate the curriculum aggressively behind subscription but allow a 7-day trial. The key UX takeaway is the *real-time per-note correctness signal* — green/red feedback mapped onto the actual notation glyph, not a separate dashboard.

### Soundslice
Web-based player + editor. Notation scrolls with a smooth orange-line **playhead** during synced audio/video playback. User can choose whether the playhead stays at top, middle, or whether scrolling is disabled (useful while editing). Click any note in the score to jump audio to that moment. Notably, Soundslice **does not auto-transcribe** ("no software does it with reasonable accuracy" — their words) but does auto-align *existing* notation to audio via "syncpoints" placed on the waveform, with automatic guesses the user can drag-correct. Excellent model for the playback half of our app.

### Hooktheory / TheoryTab
65k+ analyzed songs. Display is *not* standard notation — chords are rendered as **color-coded blocks labeled with Roman numerals** (I, IV, V), each color tied to a scale degree, and melody notes float on a scale-relative staff with note duration encoded as block width. This makes pattern recognition across keys trivial. The "Trends" tool builds progressions interactively, sizing each suggested next-chord by its database probability — a good model for showing AI confidence without numbers.

### Songscription / Ivory (modern AI piano transcribers)
Both demonstrate the current consumer paywall pattern: free tier transcribes the first 30-45 seconds of any clip (Songscription = 30s unlimited, Ivory = 45s of any YouTube piano video), upgrade unlocks full-length output and richer export formats (MusicXML, MIDI, PDF). Lets users feel the magic on a real song before paying.

## Recommended UX Patterns for Play by Ear Helper

1. **One big capture button on home; configure later.** Mirror Klangio/ScoreCloud: the home screen is "Record / Upload / Paste link", nothing else. Don't ask the user to pick instruments, key, or tempo upfront — detect what you can, then let them refine on the result screen.

2. **Show a *spectrogram or waveform* during the wait, not a generic spinner.** A 5-30s wait feels twice as long with a featureless progress bar. AnthemScore's spectrogram doubles as both progress feedback and an explanation of what the AI is "seeing". Even a faked one with progressive notes appearing as they're detected would beat a spinner.

3. **First-class confidence channel in the score.** Adopt AnthemScore's pattern: high-confidence notes solid, low-confidence notes rendered in a muted/outlined style (white-on-staff, dashed stems, or a subtle yellow tint). Add a global **likelihood slider** the user can drag to expand/contract which uncertain notes are "committed". This is the single most important differentiator most competitors miss.

4. **Per-instrument selection happens *after* transcription, on the result page.** Show the detected instruments as toggleable chips ("Piano", "Guitar", "Bass", "Drums") above the score, with each chip showing a confidence indicator. User taps a chip to see that part's sheet music. Avoid Yousician's upfront-instrument-pick pattern — we're a transcription tool, not a learning tool.

5. **Synced scrolling playhead with click-to-seek.** Steal directly from Soundslice: orange/colored vertical line over the staff, smooth scroll (not page-flip), click any note to jump audio to that beat. Provide playhead-position options later (top vs middle anchored) but ship with middle-anchored as default.

6. **Inline editor with audio scrubbing, not a separate "edit mode".** Reviewers consistently punish read-only output. Tap a note to play just that beat, drag vertically to change pitch, drag horizontally to change duration. Keep the audio source available so users can A/B their correction against the original.

7. **Free demo: full pipeline on a 30-second clip.** Follow Songscription/Ivory: the user must experience a complete transcribe -> view -> playback cycle for free, on *their own* clip, before any paywall. Gate full-length transcription, MusicXML/MIDI export, and the likelihood slider behind Pro. Do **not** gate basic playback or PDF preview — that's the Chordify mistake.

8. **Export menu must include MIDI and MusicXML, not just PDF.** Klangio's PDF + MIDI (quantized & unquantized) + MusicXML + GuitarPro is the right minimum bar. Serious users will go to a competitor the moment they hit a PDF-only wall.

9. **Show, don't number, AI uncertainty in chord labels.** Where chord detection is fuzzy, render the chord smaller / lighter / with an alternate suggestion stacked beneath (cf. Hooktheory's probability-sized chord suggestions). Avoid "73% confidence" numerical labels — they read as bug reports to non-technical users.

10. **Onboarding = three screens max, then immediate hands-on.** ScoreCloud Express's award-winning onboarding is essentially: welcome -> "tap to record a melody" -> result. Don't tutorialize features the user hasn't asked for yet; surface them contextually (e.g., the likelihood slider gets a coach-mark the *first time* the result has low-confidence notes).

## Sources
- Klangio / Melody Scanner overview: https://klang.io/melodyscanner/
- Melody Scanner getting started: https://klang.io/help/ms-getting-started/
- Melody Scanner Universal Mode: https://klang.io/help/universal-mode/
- Melody Scanner App Store: https://apps.apple.com/us/app/melody-scanner/id6472921068
- Melody Scanner negative review: https://www.soundonsound.com/forum/viewtopic.php?t=93755
- AnthemScore homepage: https://www.lunaverus.com/
- AnthemScore documentation: https://www.lunaverus.com/documentation
- AnthemScore review (features, UI): https://ai-productreviews.com/anthemscore-by-lunaverus-review/
- AnthemScore user manual (candidate notes, likelihood/threshold sliders): https://www.scribd.com/document/891816275/Anthemscore-Manual-Support
- ScoreCloud homepage: https://scorecloud.com/
- ScoreCloud Express App Store: https://apps.apple.com/us/app/scorecloud-express-hd/id763658394
- ScoreCloud Wikipedia (EuroBest UX award): https://en.wikipedia.org/wiki/ScoreCloud
- ScoreCloud Sound on Sound review: https://www.soundonsound.com/reviews/doremir-scorecloud
- ScoreCloud negative reviews: https://appsupports.co/566535238/scorecloud-express/negative-reviews
- Chordify homepage: https://chordify.net/
- Chordify lyrics + live chord detection (MusicRadar): https://www.musicradar.com/news/chordify-lyrics-live-chord-detection
- Chordify review (Guitar Chalk): https://www.guitarchalk.com/chordify-review/
- Chordify Premium features: https://support.chordify.net/hc/en-us/articles/360002164538-How-to-use-Premium-features
- Yousician homepage: https://yousician.com/
- Yousician AI / Smart Recognition: https://tools.aiformusic.org/knowledgebase/articles/yousician-interactive-ai-driven-music-learning-for-guitar-piano-ukulele-bass-voice
- Simply Piano vs Yousician (Stuff): https://www.stuff.tv/sponsored/best-piano-learning-apps-2026-how-to-pick-up-the-piano-in-no-time-at-all/
- AI piano practice tools roundup: https://pianomode.com/explore/piano-accessories-setup/piano-apps-tools/ai-powered-practice-assistants-what-works-and-whats-still-missing/
- Soundslice features: https://www.soundslice.com/features/
- Soundslice playhead scrolling options: https://www.soundslice.com/help/en/player/advanced/116/playhead-scrolling-options/
- Soundslice smooth scrolling: https://www.soundslice.com/blog/28/new-smooth-scrolling-during-playback/
- Soundslice transcribing: https://www.soundslice.com/transcribe/
- Soundslice automatic syncpoints: https://www.soundslice.com/help/en/creating/syncing/296/automatic-syncpoints/
- Hooktheory TheoryTab: https://www.hooktheory.com/theorytab
- Hooktheory homepage / Trends tool: https://www.hooktheory.com/
- Hooktheory blog (free resources, color/Roman numeral system): https://www.hooktheory.com/blog/free-hooktheory-resources/
- Songscription AI: https://www.songscription.ai/
- Ivory AI piano transcription: https://ivory-app.com/
- Onboarding/paywall best practices: https://www.airbridge.io/blog/subscription-app-onboarding
- Paywall design examples: https://dev.to/paywallpro/paywall-design-examples-from-top-productivity-apps-3fdc
