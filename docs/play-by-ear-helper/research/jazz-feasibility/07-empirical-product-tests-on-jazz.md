# Empirical Product Tests on Jazz

## Summary

The honest story from users and reviewers is consistent across every consumer transcription product surveyed: **all of them break on real jazz**, and they break in the same places. They handle slow, well-recorded solo piano or a clean monophonic melody passably (often quoted in the 80–96% pitch-accuracy range), but the moment you feed them something resembling a real 15-second jazz quartet clip — bebop eighth-note lines, swing/rubato timing, comping voicings with chord extensions, and a horn sharing space with piano/bass/drums — accuracy collapses. The most often-cited failure modes are (a) rhythm/swing quantization, (b) extended chord recognition (most tools refuse to emit anything beyond maj/min/7), (c) inability to separate the horn from the comping instrument, and (d) collapsing multi-instrument input onto one piano grand staff. Reviewers and even the products' own marketing now openly steer jazz users toward manual tools (Transcribe!, Soundslice, Amazing Slowdowner) rather than auto-transcription, and the only academic pipelines that actually work on jazz (e.g., the QMUL Charlie Parker Omnibook reconstruction, Jazz Trio Database) require *jazz-specific* trained models plus source separation — they are not what shipped consumer products are doing.

## Tool-by-Tool Reports

### AnthemScore (Lunaverus)

What it claims to be good at: solo piano, slower tempos. Lunaverus' own marketing says "It works best when the audio file is simple, e.g., solo piano." Independent reviews put solo-piano pitch accuracy "around 96% if the music file consists only of a piano instrument alone in slower speed."

What users and reviewers actually report on jazz / multi-instrument material:

- "AnthemScore fails when you input a multi-track recording. For example, with an audio track comprising vocals, piano, and drums, the output is a transcription in which everything is laid out on a piano grand staff, which won't be useful to the majority of musicians." (Verbit; echoed by multiple review aggregators.)
- "AnthemScore excels with single-instrument or simple tracks but may struggle with complex polyphonic audio, requiring manual corrections." (Musician Stack / Verbit consensus.)
- On rubato / ballad / swing material: AnthemScore's transcription comes out with "only a trace of rhythm, with lots of notes present but not strictly in rigid beat" and with note durations "all over the place, with lots of 1/16th notes, 1/8th notes, quarter notes, and so on" — i.e., it cannot quantize swing, only literal onsets. (MuseScore forum thread on rubato transcription.)
- Forum users on jazz solos: "transcribing jazz solos, particularly ballads, is noted as being more challenging for rhythm accuracy than pitch detection" — pitch sometimes lands, rhythm is usable only after heavy hand-editing.
- For mandolin and other doubled-string sources, "the output score doubled up many of the notes" — same class of problem you'd see with two horns playing in unison or octaves in a small combo.

Net: works as a pitch suggester for an isolated piano stem at moderate tempo. Will not give you a usable lead sheet from a 15-second quartet clip.

### Klangio / Melody Scanner

What it claims: multi-instrument support including Piano2Notes, Wind2Notes (trumpet, sax, clarinet, etc.), and lead-sheet style output with chord symbols — explicitly markets toward jazz.

What reviewers actually report:

- Klangio's own honest framing: "Transcription accuracy is dependent on many different factors, such as quality of audio, amount of instruments, mixing and effects, and if a professional will struggle to transcribe a certain part of a song, then so will Klangio's AI."
- Important practical limit even when it detects multiple instruments: "for non-pop music, Klangio can detect instruments but won't be able to discern separate parts for instruments (Trumpet 1 & 2, for example)." So a quartet with two horns gets merged.
- Piano2Notes: "Klangio accurately captured piano notes, [but] does not know what hands played which notes, and has some trouble with the rhythm accuracy" — i.e., no voice/staff separation, and rhythm in jazz piano comping comes out scrambled.
- Melody Scanner is described by its own ecosystem as: "Melody Scanner's core feature is to transform a song into notes, rather than providing an accurate transcription" — explicitly not a precision tool.
- Positive reports tend to be on clean monophonic input ("This app pretty much dead on hits each note it hears"), not on quartet recordings.

Net: best of the consumer tools at *attempting* lead-sheet style and at horn input, but it does not separate two horns, struggles with rhythm, and chord symbols on jazz are unreliable.

### ScoreCloud (DoReMIR)

What users and reviewers report:

- Official scope from ScoreCloud's own docs and reviewers: "ScoreCloud works best with monophonic audio (one note at a time), such as melodies and bass lines, while chords, multiple voices, and songs with accompaniment do not work well." That alone disqualifies it for a jazz-quartet input.
- Microphone input on ensemble: "users reported that ScoreCloud failed to properly transcribe multiple instrument recordings from the microphone option, and even when the track of interest was much higher in the mix, it could not properly transcribe the piece."
- Polyphonic guitar even when isolated: "using large chords (5 or 6 strings) was very difficult for the program to notate, and not playing each chord tone simultaneously — or even rolling the chord slightly — different inflections, minor note omissions, and string buzzing can really mess up ScoreCloud's transcription." Jazz guitar comping is exactly this.
- Stability complaints: "the program can be somewhat buggy when transcribing audio, with users tending to get errors and timeouts when recording over 60 seconds of music."

Net: unsuitable for any jazz beyond single-note melody captured very cleanly.

### Songscription (2025 launch)

This is the most candid recent data point because the MusicRadar review ("Humans will be doing all the serious music transcription for the foreseeable future") was published at launch and tested it on real material including Monk:

- On a Monk piano piece, the reviewer found "Songscription gets the notes right for the most part, [but] it is completely lost with the rhythms, and the chord symbols are all over the place."
- General verdict from the same review: "For any kind of real-world use, cleaning up its output would be harder work than writing charts the old-fashioned way." And: "Real-world ensemble audio, noisy recordings, mixed instrumentation, expressive timing — it's likely to produce errors."
- Hard product limit: "Songscription currently supports only one instrument at a time. For full band scoring (woodwinds + brass + percussion), this means you'll either need to extract single-instrument parts or wait for future updates."
- The vendor itself markets the use case as "jazz piano players wanting to transcribe improvisations" but the launch review specifically calls out that the rhythm and chord output on jazz piano is unusable.

Net: 2025-state-of-the-art consumer AMT, tested on jazz piano at launch, fails at rhythm and chord symbols on Monk; no quartet support at all.

### Chordify

Chord recognition only — but most directly relevant to "user wants piano chords."

- Documented accuracy gap: "Accuracy drops from ~85% on pop tracks to under 40% on unaccompanied jazz guitar solos." The root cause cited: "chord recognition algorithms trained on pop and rock corpora assume full voicings with clear bass notes and stable durations, while jazz guitar breaks all three assumptions: voicings are sparse, bass is often absent or shared with another instrument, and chords pulse rhythmically — sometimes lasting only an eighth note."
- Vocabulary limit: "Chordify doesn't produce complex chord suggestions at all, just majors and minors and very rarely a 7th chord." So even if it identified the right root, you would not get the alterations or extensions that define a jazz chord.
- Chordify's own published research acknowledges the ceiling: human annotators agree "only 76 percent of the time over simple chords … [and on] complex experimental jazz harmonies the level of agreement went down to 59 percent" — and Chordify's model is trained against the agreed labels, so jazz is upper-bounded by that disagreement floor.
- Reviewer summary: "Most chord recognition tools treat jazz standards like pop songs: they reduce chord progressions to four generic block chords, ignoring inner motion, inversions, and harmonic color."

Net: cannot handle bebop changes, cannot emit jazz vocabulary, sub-40% on unaccompanied jazz guitar — essentially useless for Bird-style harmony.

### Moises.ai

Two separate features matter here: stem separation and chord detection.

Stem separation:
- "Wind instruments still have problems, as horns and saxophones often get lumped into the wrong stems or sound unnatural when isolated."
- "The track separation can be problematic because it may clump horns and aux keys and sometimes guitar parts as a nondescript 'other' track." This is exactly what would happen to a trumpet in a piano-bass-drums quartet: you would not get a clean trumpet stem.
- Jazz-forum users report some success making practice tracks ("Inner Urge, Isotope, and Joy Spring … created versions with horn and no keys to practice comping") but also that "Moises has been found fairly ineffective in separating piano or guitar from other instrument tracks."
- Verdict from a 2026 review: "Moises AI is considered the best all-around music practice tool available for the price … if you're a jazz or classical musician, it's probably not the ideal tool."

Chord detection:
- "Moises' chord detection isn't reliable enough for jazz musicians learning complex harmony, and users often end up frustrated — it's better to transcribe by ear or use specialized jazz transcription tools."
- Quantified: "for jazz or anything with extensions and alterations, the chord detection is only 40-50% accurate at best, compared to 85-90% accuracy for basic pop/rock chord progressions." On a "complex jazz arrangement featuring trumpet, sax, piano, guitar, bass, and drums all going simultaneously, the chord detection was almost useless for anything beyond basic jazz standards."

Net: best-in-class for "pull horn out of small combo" but still not clean; chord detection on jazz is in the 40–50% range — same regime as Chordify.

### LALAL.AI

- Now offers explicit Wind and String stems including "trumpets, trombones, horns, tubas … flutes, clarinets, saxophones."
- Real-world results are mixed: "When testing jazz tracks with complex brass sections, LALAL.AI showed inconsistent results with jazz guitar and wind instruments, with some tracks having residual sounds remaining after separation."
- One sax player on CafeSaxophone: "very useful to isolate the sax, with the primary purpose being to hear the phrasing for more accurate transcription" — i.e., it is good enough as a *listening aid* even when not clean enough as a stem.
- General consensus: "For pop, rock, and hip-hop tracks, LALAL.AI delivers consistently clean stems. However, some inconsistency exists with wind instruments and complex jazz guitar."

Net: similar to Moises — usable as a slow-down/listening aid for the horn, leaks badly enough that the resulting stem will trip downstream pitch trackers.

### Transcribe! / Soundslice / Amazing Slowdowner / Anytune (manual aids)

These are not auto-transcribers, and that is exactly why jazz musicians keep recommending them. Across forums:

- "Transcribe! is recommended as an excellent choice for musicians working with jazz or vocal pieces, where a more hands-on approach to transcription can capture the nuances of these genres."
- "Soundslice has been described as transformative, allowing users to work through swing-era chord melody solos after 20 years of struggling."
- Sonic Visualiser (free) and Reaper are also commonly cited as the actually-used jazz transcription stack.

Net: this is what working jazz musicians use because they have given up on auto-AMT for jazz.

## Comparative Tests

The most useful comparative data points:

- **MusicRadar / Songscription launch review (2025)**: Tested Songscription on Thelonious Monk piano. Conclusion: notes mostly right, "completely lost with the rhythms, and the chord symbols are all over the place." Generalized verdict that fixing the output is harder than transcribing by hand.
- **Lunaverus' own comparison page** (lunaverus.com/compare) tacitly admits AnthemScore is not for ensemble transcription — it positions Transcribe! as the alternative for nuanced/jazz/vocal work.
- **Holzapfel et al., ISMIR 2019 user study** (archives.ismir.net/ismir2019/paper/000082.pdf): a controlled comparison cited as showing "the ScoreCloud system performs significantly better than a competing system for the extra- and missing note rates, and for the mean error rate" — but this was on ethnomusicological monophonic material, not jazz.
- **QMUL Saxophone Transcription Pipeline / Charlie Parker Omnibook reconstruction (arXiv 2405.16687)**: a research-only pipeline with a *jazz-saxophone-specific* source separator + a saxophone MIDI model + monophonic MIDI-to-score. Even with all that, "it hasn't been possible yet to transcribe the complete Omnibook," "downbeat estimation remains challenging on this source material," and the authors caution that even *human* Omnibook transcriptions are unreliable on faster passages. This is the strongest available evidence that the consumer tools, which use generic models, are nowhere near solving jazz.
- **Jazz Trio Database (TISMIR 2024)** and **PiJAMA** (Piano Jazz with Automatic MIDI Annotations): purpose-built jazz datasets exist precisely *because* generic AMT models trained on MAESTRO (classical piano) underperform on jazz. UT Austin's "All That Jazz" project (2021) makes the same point: existing datasets "contained only classical music and were less successful when attempting to transcribe other genres."

## Common Failure Modes Across Tools

1. **Rhythm/swing quantization fails.** Universal across AnthemScore, Songscription, Klangio. Outputs over-fragmented note grids (lots of stray 16ths and 8ths) or strip swing entirely. Ballads with rubato are worst.
2. **Extended/altered chord vocabulary not emitted.** Chordify and Moises both cap out around triads and dominant 7s. On Bird changes (rapid ii-V's, tritone subs, altered dominants) accuracy is reported in the 40–50% range.
3. **No instrument separation in output.** AnthemScore collapses multi-instrument audio onto a single piano grand staff. Klangio cannot split two horns. Songscription is one-instrument-at-a-time by design.
4. **Source separation leakage.** Moises and LALAL.AI both leak piano/guitar comping into the horn stem, or dump horn into "other," especially in small-combo recordings where the horn shares register with piano right hand.
5. **Polyphony/voicings.** ScoreCloud explicitly fails on chordal/comping input. Klangio Piano2Notes does polyphony but cannot do hand/voice separation, so jazz piano voicings come out as a single tangled grand staff.
6. **Training-data domain shift.** The academic literature is unanimous: the MAESTRO/POP-trained AMT models that ship in consumer tools were not trained on jazz, and degrade hard on jazz harmony, syncopation, and swing.
7. **Two horns / unison / octaves.** Klangio explicitly cannot tell Trumpet 1 from Trumpet 2; AnthemScore "doubles up" notes on doubled-string instruments — same failure mode would apply to a quartet with horn doubling piano right hand.

## Verdict

For the specific use case (15-second jazz quartet clip → trumpet melody notes or piano comping chords), no shipping consumer tool will deliver a usable result end-to-end:

- **Trumpet melody from quartet audio**: best path is Moises or LALAL.AI to extract a (leaky) horn stem, then Klangio Wind2Notes or Songscription on that stem. Expect correct pitches on simple lines, mangled rhythm, and bad results on bebop eighth-note lines or fast articulations.
- **Piano chord symbols**: every available tool (Chordify, Moises chord finder, Klangio lead sheet, Songscription) is reported in the 40–50% range on jazz vocabulary and is structurally incapable of emitting the alterations/extensions a jazz user actually wants. This is the harder of the two requests, not the easier.
- **The honest market signal**: even Songscription's 2025 launch review told jazz users to keep transcribing by hand, and AnthemScore's vendor recommends Transcribe! (a manual tool) for jazz. Working jazz musicians on jazzguitar.be, CafeSaxophone, and Sax on the Web converge on Transcribe!, Soundslice, Amazing Slowdowner, Sonic Visualiser, and Reaper — none of which auto-transcribe.

A "play by ear helper" that targets jazz quartet input therefore cannot rely on chaining existing consumer APIs and expecting a clean lead sheet; it needs jazz-specific models (à la Jazz Trio DB, PiJAMA, QMUL sax pipeline), and even those research systems are still not solved.

## Sources

- AnthemScore reviews / vendor: https://www.lunaverus.com/, https://www.lunaverus.com/compare, https://verbit.ai/transcription/the-best-music-transcription-software/, https://themusicrealm.com/anthemscore/, https://ai-productreviews.com/anthemscore-by-lunaverus-review/, https://musicianstack.com/music-transcription-software/, https://forum.pianoworld.com/ubbthreads.php/topics/3053831/, https://musescore.org/en/node/272456, https://musescore.org/en/node/279528, https://www.oreateai.com/blog/anthemscore-vs-transcribe-choosing-the-right-music-transcription-tool/30da32697950b79ac7a0a92e0335ae7c
- Klangio / Melody Scanner: https://klang.io/, https://klang.io/piano2notes/, https://klang.io/wind2notes/, https://klang.io/blog/top-5-transcription-tools/, https://www.musictechhelper.com/blog/audio-to-sheet-music-meet-klangio, https://declom.com/klangio/, https://melodyscanner.com/, https://edimakor.hitpaw.com/subtitle-tips/transcribe-music-software.html
- ScoreCloud: https://scorecloud.com/learn/best-music-transcription-software/, https://scorecloud.com/support/, https://www.soundonsound.com/reviews/doremir-scorecloud, https://musicedmagic.com/tales-from-the-podium/11733-scorecloud-studio-music-notation-software-review, https://composerstoolbox.com/2018/08/28/scorecloud-pro-review/, https://www.theguitarjournal.com/how-to-easily-automatically-transcribe-fingerstyle-guitar-scorecloud-4-review/, https://archives.ismir.net/ismir2019/paper/000082.pdf
- Songscription: https://www.songscription.ai/, https://www.songscription.ai/faq, https://www.musicradar.com/music-tech/humans-will-be-doing-all-the-serious-music-transcription-for-the-foreseeable-future-songscription-review, https://musically.com/2025/07/01/songscription-uses-ai-to-automate-sheet-music-transcription/, https://techcrunch.com/2025/06/30/songscription-launches-an-ai-powered-shazam-for-sheet-music/, https://www.musictechhelper.com/blog/audio-gt-sheet-music-meet-songscription, https://www.producthunt.com/products/songscription-ai/reviews
- Chordify: https://chordify.net/pages/live-chord-detection/, https://chordify.net/pages/subjectivity-in-chord-recognition/, https://www.trustpilot.com/review/chordify.net, https://www.hooktheory.com/blog/chordify-alternatives/, https://www.alibaba.com/product-insights/ai-powered-guitar-chord-recognizer-vs-chordify-for-complex-jazz-progressions-which-reads-voicings-correctly.html, https://www.quora.com/Is-the-Chordify-app-accurate-Im-yours-by-Jason-Mraz-is-in-key-of-C-major-but-the-app-shows-chords-in-key-of-B-major
- Moises.ai: https://moises.ai/, https://help.moises.ai/hc/en-us/articles/360010972019-Which-instruments-can-be-separated-on-Moises, https://moises.ai/blog/latest/advanced-chord-detection/, https://aisongcreator.pro/blog/moises-ai-review-2026, https://stemsplit.io/blog/moises-ai-review, https://www.jazzguitar.be/forum/everything-else/99548-moises.html
- LALAL.AI: https://www.lalal.ai/, https://www.lalal.ai/blog/wind-string-instruments/, https://www.lalal.ai/guides/how-to-remove-wind-instruments/, https://teds-list.com/review/lalal-ai-review-is-this-the-best-stem-splitter-on-the-market/, https://www.fahimai.com/lalal-ai, https://cafesaxophone.com/threads/ai-to-remove-instruments-from-recordings.35727/, https://www.youtube.com/watch?v=zzPqr1HcMow
- Manual jazz transcription tools (Transcribe!, Soundslice, etc.): https://www.jazzguitar.be/forum/ear-training-transcribing-reading/96036-transcribing-software-2023-a.html, https://www.jazzguitar.be/forum/recording-music-software/91871-good-app-transcribing.html, https://www.saxontheweb.net/threads/transcription-workflow-and-tools-what-do-you-use.385561/, https://www.freejazzlessons.com/jazz-transcription/
- Academic / benchmarks on jazz: https://arxiv.org/abs/2405.16687 and https://arxiv.org/html/2405.16687v1 (Charlie Parker Omnibook reconstruction), https://aim-qmul.github.io/SaxTranscriptionPipeline/, https://transactions.ismir.net/articles/10.5334/tismir.186 (Jazz Trio Database), https://transactions.ismir.net/articles/10.5334/tismir.162 (PiJAMA), https://www.cs.utexas.edu/news/2021/all-jazz-improving-automated-piano-note-transcription, https://www.mdpi.com/2076-3417/13/21/11882, https://arxiv.org/html/2406.15249v1, https://labsites.rochester.edu/air/publications/benetatos19automaticmusic.pdf
