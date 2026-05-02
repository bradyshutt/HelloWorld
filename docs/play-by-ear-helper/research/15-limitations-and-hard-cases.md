# Limitations & Hard Cases

## Summary

AI music transcription is fundamentally a hard, unsolved problem outside narrow conditions. The single most important fact for product positioning: accuracy ranges roughly from ~96% on clean studio solo piano down to ~38% on dense polyphonic mixes (MIREX 2024-class results), with a 2025 Springer study finding that genre/recording-condition shifts alone can drop F1 scores by 20-50 percentage points. Users of leading tools (AnthemScore, Klangio, Songscription, Basic Pitch) consistently complain about three failure modes: (1) wrong rhythm/timing - chaotic note durations, especially with rubato, swing, or live recordings; (2) octave errors and spurious notes - phantom notes from harmonics, reverb tails, and distortion; and (3) catastrophic failure on dense or multi-instrument audio - overlapping instruments in similar pitch ranges cause "instrument leakage" and hallucinated notes. Vocals, distorted electric guitar, and orchestral ensembles are the hardest realistic targets; clean solo piano and isolated monophonic melodies are the realistic happy path. The honest framing is "first-draft helper" rather than "transcription engine" - every reviewer surveyed treats output as a starting point requiring substantial human correction.

## Hard Cases by Category

### Genre

- **Jazz**: One of the worst genres for AI. Improvisation, complex extended chords (9ths, 13ths, altered), microtonal blue-note bends, swing timing, and intentional silence/breath all defeat current transcription engines. State-of-the-art chord transcription systems achieve poor performance on jazz because "the way harmony is interpreted is different from many other genres." Reviewers note: "A bebop line's offbeat accents, a blues phrase's microtonal bends, the breath-like space before a triplet pickup are not artifacts to be cleaned up - they're the syntax of the language. Yet most transcription tools treat them as noise."
- **Metal / heavily distorted rock**: Distortion adds harmonics that the model misreads as additional notes; palm muting, tremolo picking, and pinch harmonics are rarely annotated correctly. Effects like flanger, reverb, and chorus "make source separation very difficult."
- **Classical (orchestral)**: Dense polyphony, sections of instruments doubling each other, hall reverb, and expressive dynamics make ensemble classical "especially challenging." Multiple instruments share overlapping ranges; section-level isolation is harder than vocal-vs-instrumental. Ornaments (trills, mordents, turns, grace notes) - common in Baroque and Classical periods - get rendered as raw fast 32nd-notes rather than as ornament symbols, producing unreadable scores.
- **Electronic / EDM**: Synthesizer patches with non-harmonic spectra, sub-bass that confuses pitch estimators, layered kick drums, and heavy sidechain compression all hurt accuracy. "EDM tracks with dense kick layers require different analysis settings than jazz." Atonal or noise-based synth lines may not even have an identifiable fundamental.
- **Vocal-heavy genres (R&B, gospel, soul)**: Melisma, runs, scoops, and heavy stylistic vibrato confound pitch detectors. "Stylized singing elements such as heavy vibrato, pitch slides, or overlapping harmonics... can mislead the AI's pitch detection, causing it to reproduce notes inaccurately."
- **World / non-Western music**: Microtonal systems (Turkish makam's 53-tone, Persian koron/sori quarter-tones, Indian raga shrutis) cannot be represented in standard Western notation. Some research systems target 20-cent resolution for Turkish music, but mainstream commercial tools assume 12-TET equal temperament and silently quantize microtonal pitches to the nearest semitone.

### Instrument

- **Piano (solo, studio)**: The "easy" case. Best AI accuracy (~96% on MIREX-class clean inputs). Still produces octave errors and pedal-induced sustain blur.
- **Guitar (acoustic clean)**: ~78% accuracy ballpark. Chord-voicing inversions, fingerings, and string/fret choice are frequently wrong. "Users... caution about mis-assigned notes, wrong string/fret choices for guitar."
- **Guitar (electric, distorted)**: Distortion adds intermodulation products read as extra notes. Bends and slides are nearly impossible to notate correctly - "bends are so subtle and nuanced that it's difficult to properly express them in a text file given their many variances." Vibrato, palm muting, and tremolo picking are rarely annotated.
- **Strings (violin, cello)**: Bowing articulation (legato/staccato/spiccato/sul ponticello) is invisible to pitch-only models. Vibrato widens the perceived pitch band and can cause the model to flicker between adjacent semitones. Glissandi get rendered as chromatic runs of discrete notes.
- **Brass with mute / vibrato**: Mutes change timbre dramatically and can mislead instrument-classification heads. Vibrato issues mirror strings/voice.
- **Woodwinds**: Generally more tractable than brass for pitch but flutter-tonguing, multiphonics, and key clicks confuse onset detection.
- **Voice**: ~52% accuracy ballpark. Vibrato, scoops/portamento, melismatic runs, breath noise, sibilants, and consonant transients all create false onsets or wandering pitch curves. Multi-tracked vocal harmonies and choirs remain nearly unsolved at production quality.
- **Drums / unpitched percussion**: Separate problem class. Best dedicated tools (DrumConvert) claim ~92% on standard kit; "for advanced songs, transcriptions are inaccurate." Ghost notes, breakbeats, double kick, brushwork, and hand percussion (congas, cajon) are commonly missed. Spotify's Basic Pitch "is unable to encode drums at all."
- **Pitched percussion (timpani, marimba, vibraphone, glockenspiel)**: A documented gap. Most consumer tools train heavily on piano/guitar and treat all percussion as unpitched. Mallet/timpani strikes have transient broadband attacks plus weak tonal components - the tonal pitch is often missed entirely or assigned to the wrong octave. No reviewed commercial tool advertises pitched-percussion accuracy.

### Recording Conditions

- **Live recordings**: Audience noise, room reverb, bleed between mics, and PA limitations all degrade transcription. AnthemScore reviewers note "accuracy is quite poor with live recordings; it often fails to capture nuances." Klangio acknowledges "occasional issues with interpreting live performances."
- **Reverb / room acoustics**: Reverb tails extend perceived note durations, smear onsets, and create overlapping harmonic content the model interprets as new notes. "Effects like reverb can degrade transcription quality."
- **Phone-mic / lo-fi recordings**: Compression artifacts, limited bandwidth (often rolled off above 8kHz), and noise floor obscure higher partials needed for pitch disambiguation.
- **YouTube rips / lossy compression**: MP3/AAC encoding removes spectral detail; cumulative re-encoding (common in user-uploaded covers) degrades pitch accuracy further.
- **Multi-instrument mixed audio**: Even with stem separation as a preprocess, residual bleed and separation artifacts cause spurious notes. AnthemScore "fails when you input a multi-track recording, with everything laid out on a piano grand staff."

### Musical Content (chords, ornaments, etc.)

- **Octave-related notes**: Documented as "a major source of errors" - the second harmonic of a low note has the same frequency as the fundamental of the note an octave above, so the model frequently merges or duplicates them.
- **Dense chords (4+ simultaneous notes)**: Missed inner voices are common. "Multiple notes are triggered simultaneously in polyphonic music, making them overlap each other in the time domain. Therefore, it is difficult to distinguish multiple notes which are occurring simultaneously, especially when there is an octave relationship."
- **Fast passages, trills, ornaments**: Trills get transcribed as alternating 32nd-notes rather than as a trill symbol; mordents and turns lose their notational identity. Minimum-note-duration filters (typically 40ms) used to suppress spurious notes can also delete legitimate fast passages.
- **Grace notes / acciaccaturas**: Often dropped (filtered as spurious) or absorbed into the following note's onset.
- **Rubato / expressive timing**: A deep failure mode. AI rhythm detectors lock onto an initial tempo with ~30-80ms tolerance and "rarely adapt mid-phrase. When you slow down for a cadence, the app doesn't reinterpret the beat - it keeps counting against its original reference, making every subsequent note appear early." Result: chaotic time signatures and shifted notes.
- **Swing / shuffle**: "'Rubato' music and swing, and anything with subtle timing can show up as messy rhythms that you will have to fix manually."
- **Polyrhythms / cross-rhythms**: Models trained on simple meters struggle to align competing pulse layers.
- **Dynamics, articulation, phrasing**: Rarely recovered at all. Most tools output flat MIDI velocities; pp/ff, accents, slurs, and staccato are not annotated.
- **Voice leading / hand assignment (piano)**: "Klangio... does not know what hands played which notes." Output is typically dumped onto a single grand staff with no left/right-hand split.

## User Complaints from Existing Tools

**AnthemScore (Lunaverus)**

- "Still too soon to get accurate transcriptions without artifacts being registered as sharp and flat notes, or the note recognition is just lousy." (2024 purchaser)
- "The transcription is horrible, notes are rhythmically all over the place, there are lots of extra notes, and transcriptions of accurately played music to a metronome wind up looking chaotic."
- "Given an A, C#, and E, AnthemScore cannot identify all of the notes - individually or combined."
- "Miss notes and write down wrong durations."
- On a mandolin recording: "the output score doubled up many of the notes."
- Fails on multi-track recordings: dumps everything onto a piano grand staff that "is not useful to most musicians."
- General theme: works acceptably on solo piano in clean conditions; degrades sharply otherwise.

**Klangio**

- One Trustpilot user reported the full transcription "was completely different in key and made no sense, unlike the sample version that was good."
- Klangio itself acknowledges: "transcription accuracy is dependent on... quality of audio, amount of instruments, mixing and effects. If a professional will struggle to transcribe a certain part of a song, then so will Klangio's AI."
- "Struggles with rhythm accuracy and lyric placement, and does not know what hands played which notes."
- "Noisy audio or overlapping instruments can cause problems."
- "Occasional issues with interpreting live performances or very dense arrangements."

**Songscription** (MusicRadar review titled "Humans will be doing all the serious music transcription for the foreseeable future")

- "Songscription gets the notes right for the most part, but it is completely lost with the rhythms, and the chord symbols are all over the place."
- In a single test, "it writes a measure of 3/4, a measure of 4/4 and a measure of 11/8 before finally settling into 6/4."
- "Everything is shifted an eighth note late" or "the rhythm is shifted over two beats."
- "Works best on beginner-level classical repertoire and very simple pop songs recorded on solo piano. If you give it other instruments, more than one instrument at a time, or music with any kind of expressive timekeeping, it struggles."
- Single-instrument-at-a-time limitation.

**Spotify Basic Pitch**

- "Not perfect for dense chords or noisy mixes."
- "Unable to encode drums at all."
- Designed for one instrument at a time; not a multi-instrument transcriber.

**Common cross-tool complaints**

- Octave errors (duplicate or wrong-octave notes from harmonics)
- Spurious short notes from reverb tails and audio noise
- Missed inner voices in dense chords
- Hallucinated instruments / instrument leakage in ensemble audio
- Wrong time signature; bars don't add up
- No dynamics, articulation, phrasing, ornaments, or hand-assignment metadata
- Chord-symbol output drifts from actual harmony, especially on jazz/extended chords

## Implications for Product Positioning

1. **Position as a first-draft assistant, not an oracle.** Every credible reviewer treats AI transcription output as a starting point. Build the editing/correction UX as a first-class feature, not an afterthought. Klangio's "Edit Mode" is repeatedly cited as what makes the tool usable despite errors - users will accept imperfect output if correction is fast.
2. **Be explicit about the supported audio "happy path" up front.** Solo piano in clean studio audio = good. Solo guitar (clean), solo voice (no vibrato), solo monophonic instrument = decent. Anything denser = warn the user. Showing an accuracy/confidence indicator per region of the score will set expectations and reduce complaints.
3. **Decide on stem separation strategy.** If targeting multi-instrument audio, integrate or recommend a separation step (Demucs, Spleeter, MDX) with an honest disclaimer that separation artifacts will hurt downstream transcription. Or scope down to solo/monophonic input and say so loudly.
4. **Avoid the rhythm-quantization trap.** The single biggest visible failure across all tools is chaotic rhythm/time-signature output. Allow users to specify or lock tempo and time signature, and prefer "render as MIDI" + tempo-track over premature notation quantization. Provide a "free time / rubato" mode that doesn't fight the user.
5. **Don't overpromise on hard genres.** Explicitly de-scope (or mark as experimental) jazz with extended chords, distorted metal, orchestral works, choral/multi-vocal harmonies, microtonal/non-Western music, and live recordings. Better to under-promise and surprise users than to ship a tool that produces nonsense on their favorite music.
6. **Pitched percussion and drums are separate problems.** Either invest in a dedicated drum model (the field treats this as its own discipline) or explicitly state that drums and pitched percussion are out of scope. Don't pretend a piano-trained model handles timpani.
7. **Capture and surface confidence.** Per-note confidence scoring would let the UI flag low-certainty regions for human review - users complain less when they know which parts to double-check.
8. **Notation vs. MIDI is a meaningful product split.** MIDI/piano-roll output forgives many transcription sins (no need to commit to time signature, ornament symbols, hand assignment). Engraved sheet music exposes every weakness. Consider making MIDI export the primary deliverable and notation a secondary "best effort" view.

## Sources

- [AnthemScore Reviews 2025 (aitools.xyz)](https://aitools.xyz/tools/anthemscore/reviews)
- [AnthemScore Reviews (Slashdot)](https://slashdot.org/software/p/AnthemScore/)
- [AnthemScore Reviews 2026 (SourceForge)](https://sourceforge.net/software/product/AnthemScore/)
- [AnthemScore Review (ai-productreviews.com)](https://ai-productreviews.com/anthemscore-by-lunaverus-review/)
- [AnthemScore on MuseScore forum](https://musescore.org/en/node/272456)
- [AnthemScore on Scribd](https://www.scribd.com/document/956195805/Anthemscore-of-Lunaverus)
- [AnthemScore comparisons (Lunaverus)](https://www.lunaverus.com/compare)
- [Klangio (klang.io)](https://klang.io/)
- [Klangio Trustpilot reviews](https://www.trustpilot.com/review/klang.io)
- [Klangio AI review (dillipai.blog)](https://dillipai.blog/klangio-ai-review/)
- [Klangio review (Dr. James Frankel)](https://www.musictechhelper.com/blog/audio-to-sheet-music-meet-klangio)
- [Klangio Drum2Notes blog](https://klang.io/blog/ai-drum-transcriptions/)
- [Klangio distorted electric guitar blog](https://klang.io/blog/solo-electric-guita/)
- [MusicRadar Songscription review - "Humans will be doing all the serious music transcription"](https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review)
- [Songscription review (Dr. James Frankel)](https://www.musictechhelper.com/blog/audio-gt-sheet-music-meet-songscription)
- [Spotify Basic Pitch (GitHub)](https://github.com/spotify/basic-pitch)
- [Spotify Basic Pitch issue: drums support](https://github.com/spotify/basic-pitch/issues/30)
- [Meet Basic Pitch (Spotify Engineering)](https://engineering.atspotify.com/2022/6/meet-basic-pitch)
- [Automatic Music Transcription: An Overview (C4DM, Queen Mary)](http://c4dm.eecs.qmul.ac.uk/spm-amt-overview/)
- [Automatic Music Transcription: An Overview (PDF, Rochester)](https://labsites.rochester.edu/air/publications/benetatos19automaticmusic.pdf)
- [Automatic music transcription: challenges and future directions (Springer)](https://link.springer.com/article/10.1007/s10844-013-0258-3)
- [Sound and Music Biases in Deep Music Transcription Models (arXiv 2025)](https://arxiv.org/pdf/2512.14602)
- [Machine Learning Techniques in Automatic Music Transcription Survey (arXiv 2024)](https://arxiv.org/html/2406.15249v1)
- [Advancing Multi-Instrument Music Transcription (OpenReview)](https://openreview.net/pdf?id=NG187AZ71W)
- [Towards Automatic Transcription of Polyphonic Electric Guitar Music (arXiv)](https://arxiv.org/pdf/2202.09907)
- [Isolated guitar transcription using a deep belief network (PeerJ)](https://peerj.com/articles/cs-109/)
- [A Review of Automatic Drum Transcription (BCU)](https://www.open-access.bcu.ac.uk/6180/1/Wu-et-al.-2018-A-review-of-automatic-drum-transcription.pdf)
- [DrumConvert (La Touche Musicale)](https://latouchemusicale.com/en/apps/drumconvert/)
- [Drumscrib](https://drumscrib.com/)
- [AI Generative Drum Transcriptions: A Comparative Analysis (Francis' Drumming Blog)](https://francisdrummingblog.com/2024/01/23/ai-generative-drum-transcriptions-a-comparative-analysis/)
- [Automatic transcription of Turkish microtonal music (PubMed)](https://pubmed.ncbi.nlm.nih.gov/26520294/)
- [The transcription of vocal microtonality (Francisco Camas)](https://www.franciscocamas.com/science-music-interactions/the-transcription-of-vocal-microtonality/)
- [Can AI Create Microtonal Music? (Soundverse)](https://www.soundverse.ai/blog/article/can-ai-create-microtonal-music-1054)
- [AI Jazz Improvisation: Current Limitations (Soundverse)](https://www.soundverse.ai/blog/article/ai-jazz-improvisation-current-limitations-1048)
- [Transcribing Lead Sheet-Like Chord Progressions of Jazz Recordings (MIT Press)](https://direct.mit.edu/comj/article/44/4/26/108550/Transcribing-Lead-Sheet-Like-Chord-Progressions-of)
- [Why Does My AI Piano App Mark Rubato As Rhythm Error (Alibaba product insights)](https://www.alibaba.com/product-insights/why-does-my-ai-powered-piano-app-mark-expressive-rubato-as-rhythm-error-every-time.html)
- [AI vs Human Music Transcription (Music Notation Hub)](https://musicnotationhub.com/blog/human-vs-ai-music-transcription/)
- [Can AI voice tools generate realistic vocal runs and melismas? (Sonarworks)](https://www.sonarworks.com/blog/learn/can-ai-voice-tools-generate-realistic-vocal-runs-and-melismas)
- [AI Harmonizer (arXiv 2025)](https://arxiv.org/html/2506.18143v1)
- [Mel-RoFormer for Vocal Separation and Vocal Melody Transcription](https://www.aimodels.fyi/papers/arxiv/mel-roformer-vocal-separation-vocal-melody-transcription)
- [A Metric for Music Notation Transcription Accuracy (ISMIR 2017)](https://archives.ismir.net/ismir2017/paper/000131.pdf)
- [Best Music Transcription Software (ScoreCloud)](https://scorecloud.com/learn/best-music-transcription-software/)
- [Songscription launches an AI-powered Shazam for sheet music (TechCrunch)](https://techcrunch.com/2025/06/30/songscription-launches-an-ai-powered-shazam-for-sheet-music/)
