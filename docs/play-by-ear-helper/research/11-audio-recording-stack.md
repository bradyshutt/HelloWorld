# Audio Recording Stack (Web & Mobile)

## Summary

For the MVP, record in the browser with **MediaRecorder** (Opus-in-WebM on Chrome/Firefox/Edge, AAC-in-MP4 on iOS Safari) and on mobile with **expo-audio** (HIGH_QUALITY preset producing M4A/AAC) — both feeding raw container files to the server. Do all heavy lifting (decode, mono downmix, resample to 16 kHz, normalize, silence-trim) **server-side with ffmpeg**, since AMT models are uniformly happiest at 16 kHz mono float32 and trying to resample in-browser is a maze of WebKit sample-rate bugs. Skip RNNoise and similar speech-tuned denoisers — they actively damage music transcription. Plan for the dominant capture mode to be ambient ("phone on the piano") and warn users in the UI that solo, close-miked, single-instrument recordings produce dramatically better transcriptions than room-mic-from-across-the-room captures.

## Web (Browser) Recording

### MediaRecorder API
The mainstream choice. `navigator.mediaDevices.getUserMedia({audio: true})` returns a `MediaStream`, which feeds a `MediaRecorder` instance. Production-ready in Chrome 47+, Firefox 25+, Edge 79+, and Safari 14.1+ (with caveats below). Always probe with `MediaRecorder.isTypeSupported()` before setting `mimeType`, and fall back gracefully — hardcoding `audio/webm;codecs=opus` will silently break iOS.

Use the `timeslice` argument (`recorder.start(2000)`) to receive `dataavailable` events every N milliseconds for chunked uploads — better for long takes and flaky networks than buffering everything in memory.

### Format support per browser (May 2026)
- **Chrome/Edge/Firefox**: `audio/webm;codecs=opus` is universally supported. Opus at 64–128 kbps is plenty for transcription.
- **Safari (desktop & iOS) ≥ 14.5**: produces `audio/mp4` containing AAC. Safari 18.4 added WebM/Opus *recording* parity, but you cannot rely on it on older iOS versions still in the wild.
- Setting `audioBitsPerSecond` above ~256 kbps in Chrome can produce files that Safari refuses to play back; keep it modest.

### AudioWorklet / Web Audio API path
If you want to skip MediaRecorder entirely and produce a clean WAV (16-bit PCM), pipe the `MediaStreamAudioSourceNode` through an `AudioWorkletNode` that batches `Float32Array` chunks, clamps to [-1, 1], scales to int16, and writes a WAV header. This bypasses container/codec headaches and gives you direct PCM, but you pay in code volume and you still inherit the AudioContext sample-rate quirks below.

`OfflineAudioContext({sampleRate: 16000, ...})` can resample an already-captured `AudioBuffer` faster than realtime — but Chromium uses SincResampler and Firefox uses Speex's resampler, so cross-browser results differ subtly. For consistent results, resample server-side.

### Microphone permissions flow
- `getUserMedia` requires a **secure context** (HTTPS or `localhost`). The prompt only appears in response to user activation — call it from a click handler, never on page load, or browsers will silently block.
- **Chrome/Edge**: persistent per-origin grant. One prompt, remembered.
- **Firefox**: by default *not* persistent — re-prompts each session unless the user explicitly checks "Remember." Roughly half of Firefox prompts take >2 s to resolve, so don't tightly couple recording UI to permission resolution.
- **Safari**: per-origin prompt; remembered, but Safari is more aggressive about revoking on inactivity. Cross-origin iframes need `allow="microphone"` plus a `Permissions-Policy: microphone=(self "https://...")` header.
- Use the `Permissions API` (`navigator.permissions.query({name:'microphone'})`) to detect prior grant/denial state and skip the prompt UX where possible.

### iOS Safari quirks (the WebKit gotchas)
1. **MIME type**: iOS Safari emits `audio/mp4` (AAC), not WebM. Build server decoders accordingly.
2. **`audioBitsPerSecond`**: iOS ignores or misapplies high values; stick to defaults.
3. **Frame duration**: WebKit records frames at the smallest possible 2.5 ms duration with no API to change it. Generally fine for transcription, but downstream chunked decoders may need to handle small fragments.
4. **Whisper / strict decoders choke on Safari MP4 blobs** — recognition has been observed to cut off after 1–3 words. Re-mux server-side with ffmpeg before passing to any AMT/ASR pipeline.
5. **AudioContext sample rate mismatch**: Safari sometimes reports 44.1 kHz but the hardware actually runs at 48 kHz, producing distortion. Workaround: explicitly construct `new AudioContext({sampleRate: 44100})`, or use `ios-safe-audio-context` (npm) which probes and resets. Always alias `window.AudioContext || window.webkitAudioContext`.
6. **AudioContext must be unlocked by user gesture** — touch/click handler. Silent ringer mode also blocks Web Audio playback (recording works, but verifying takes via playback fails).
7. **Background tab**: iOS suspends MediaRecorder when the tab loses focus; warn the user.

## Mobile Recording

### React Native / Expo
- **`expo-audio`** is the go-forward library. `expo-av` is **deprecated** — patches stopped, removal slated for SDK 55, and SDK 54 already drops it from Expo Go.
- On Expo SDK 51/52, `expo-audio` is unstable; stick with `expo-av` if pinned to those. On SDK ≥ 53, migrate to `expo-audio`.
- `expo-audio` exposes `RecordingPresets.HIGH_QUALITY` (44.1 kHz, AAC in M4A, ~128 kbps) and `LOW_QUALITY` plus per-platform overrides (`android.extension`, `ios.audioQuality`, `ios.outputFormat`, etc.) — set channels to 1 for mono.
- **`react-native-audio-recorder-player`** is the bare-RN alternative for projects that have ejected or need deeper background recording / different format options. Less actively maintained than expo-audio.
- **`@siteed/expo-audio-stream`** and **`expo-recorder` (lodev09)** are useful when you need waveform visualization or chunk-streamed PCM.

### Native iOS (AVAudioRecorder)
Settings dictionary: `AVFormatIDKey: kAudioFormatLinearPCM`, `AVSampleRateKey: 44100` (or 22050/16000), `AVNumberOfChannelsKey: 1`, `AVLinearPCMBitDepthKey: 16`, `AVEncoderAudioQualityKey: .max`. Producing LinearPCM directly in a `.wav`/`.caf` removes ambiguity and skips the AAC re-encode round-trip. AAC (`kAudioFormatMPEG4AAC` in `.m4a`) is fine if you want smaller files; the server will decode either.

### Native Android
- **`AudioRecord`** for music: gives raw PCM, full control over sample rate, bit depth, and buffer size. No file writing — you assemble the WAV header yourself. This is the right choice for transcription-quality capture.
- **`MediaRecorder`** is higher-level and writes compressed AAC/AMR directly to a file, but offers minimal customization and quality is lower. Good only for voice-memo–grade use.
- Minimum-effort production setup: `AudioRecord` at 44.1 kHz, 16-bit, mono, write to PCM-WAV, upload.

## Audio Format Pipeline

### What AMT models actually want
- **Sample rate**: 16 kHz is the de facto standard across modern AMT/ASR work (MT3, Onsets-and-Frames variants, ReconVAT, Whisper-based pipelines). Some piano-specific models prefer 22.05 kHz mel-spectrograms. **Pick one canonical rate (16 kHz) and resample everything to it.**
- **Channels**: mono. Downmix L+R = (L+R)/2, or take left only — doesn't matter much for transcription.
- **Bit depth**: float32 in memory; 16-bit PCM on disk is fine.
- **Format on disk**: WAV or FLAC (lossless). FLAC is ~50% smaller and `librosa`/`torchaudio` read both.

### Server-side preprocessing with ffmpeg
A canonical one-shot command:

```
ffmpeg -i input.{webm,mp4,m4a,wav} \
  -ac 1 -ar 16000 \
  -af "highpass=f=30, loudnorm=I=-16:TP=-1.5:LRA=11, silenceremove=start_periods=1:start_duration=0.1:start_threshold=-50dB:detection=peak,areverse,silenceremove=start_periods=1:start_duration=0.1:start_threshold=-50dB:detection=peak,areverse" \
  output.wav
```

- `-ac 1 -ar 16000` → mono, 16 kHz.
- `loudnorm` is EBU R128 normalization; for higher fidelity run two-pass (analysis → apply with measured values).
- `silenceremove` with `areverse` trims silence from both ends.
- The `slhck/ffmpeg-normalize` Python wrapper handles two-pass loudnorm cleanly if you don't want to script it.

### In-browser preprocessing — generally don't
Tempting, but:
- WebKit AudioContext sample-rate bugs make in-browser resampling unreliable.
- Resampler quality differs across Chromium/Firefox.
- You'd need to ship a WAV encoder and possibly an Opus decoder polyfill.
- Server ffmpeg is one binary, deterministic, and you control the exact pipeline.

The only browser preprocessing worth doing: client-side **silence-skip detection** to refuse-and-prompt on totally-silent uploads (cheap RMS check on the AudioBuffer), and **duration capping** to prevent absurdly long uploads.

## Ambient Capture (Microphone in a Room)

### Reality check
A phone-on-the-piano recording is dramatically lower fidelity than a DI/line-out signal:
- Built-in smartphone mics are **tuned for voice** (~50 Hz–16 kHz effective response) with aggressive low-frequency rolloff and AGC.
- Room reverberation, HVAC, traffic, and chair noise all enter the mix.
- Stereo image is collapsed (most phones record mono or near-mono from a single bottom mic).
- Pianos especially radiate from a wide soundboard and degrade noticeably when captured mono — but mono is what AMT models want anyway.

### What helps
- **External mic via USB-C/Lightning**: Shure MV88+ is the canonical recommendation for both iOS and Android. Even an $80 lavalier is a step up.
- **Mic placement**: 1–2 m from the instrument, pointed at the soundboard (piano) or 12th fret (acoustic guitar). Close enough for clean signal; far enough to avoid clipping.
- **Solo recording**: AMT accuracy collapses on polyphonic mixes with multiple instruments, vocals, or a backing track. UI should explicitly request solo instrument input.
- **Sloppy playing → sloppy transcription**: from Klangio's docs, slow and articulated > fast and muddy.
- Klangio, AnthemScore, and Piano2Notes all describe the same set of preconditions; the limits are physical, not algorithmic.

### Noise/echo reduction — should we use it?
- **RNNoise** (xiph): trained on speech vs. non-speech. It will actively *damage* musical content because the model treats sustained tones, harmonics, and reverb as noise to be suppressed. Documented fine on stationary background hum but degrades on babble, music, and transients. **Do not run RNNoise on music inputs.**
- Similarly: WebRTC's `noiseSuppression`, `echoCancellation`, and `autoGainControl` constraints in `getUserMedia` are voice-tuned and hurt transcription. **Explicitly disable them**:

```
getUserMedia({audio: {echoCancellation: false, noiseSuppression: false, autoGainControl: false}})
```

This is one of the highest-impact single decisions in the recording stack. The default `getUserMedia({audio: true})` is the wrong call for music.

- For genuine noise problems, prefer offline music-aware tools (iZotope RX, Demucs source-separation to isolate the target instrument) rather than realtime speech denoisers.

## Recommendation

**MVP recording stack:**

1. **Web**: MediaRecorder with format auto-detection (WebM/Opus → MP4/AAC fallback). Disable `echoCancellation`/`noiseSuppression`/`autoGainControl` in `getUserMedia` constraints. Use `timeslice` chunked uploads. Wrap AudioContext init with `ios-safe-audio-context` to dodge iOS sample-rate bugs.
2. **Mobile**: `expo-audio` with `HIGH_QUALITY` preset, channels=1, on Expo SDK ≥ 53. M4A/AAC out.
3. **Upload**: send the raw container to the server unchanged. Don't preprocess in-client beyond a silence/duration sanity check.
4. **Server**: ffmpeg pipeline → mono, 16 kHz, two-pass loudnorm, double-ended silence trim, output WAV (or FLAC for storage). One canonical format hits the AMT model.
5. **UX guidance**: prompt the user for solo, close-miked, single-instrument captures; warn on detected polyphony / low SNR before consuming model budget.
6. **Defer**: in-browser WAV encoding via AudioWorklet, music-source-separation preprocessing, and any speech-style noise suppression.

## Sources

- [MediaRecorder — MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder)
- [MediaRecorder.mimeType — MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/mimeType)
- [Record audio and video with MediaRecorder — Chrome for Developers](https://developer.chrome.com/blog/mediarecorder)
- [MediaRecorder API — WebKit blog](https://webkit.org/blog/11353/mediarecorder-api/)
- [Recording cross browser compatible media — Christoph Guttandin](https://media-codings.com/articles/recording-cross-browser-compatible-media)
- [How to Implement MediaRecorder for iPhone Safari — buildwithmatija](https://www.buildwithmatija.com/blog/iphone-safari-mediarecorder-audio-recording-transcription)
- [Whisper: problem with audio/mp4 blobs from Safari — OpenAI forum](https://community.openai.com/t/whisper-problem-with-audio-mp4-blobs-from-safari/322252)
- [Sample Rate bug in Safari — chrisguttandin/standardized-audio-context #489](https://github.com/chrisguttandin/standardized-audio-context/issues/489)
- [ios-safe-audio-context — npm](https://www.npmjs.com/package/ios-safe-audio-context)
- [HTML5 audio stuttering on iOS Safari caused by AudioContext sample rate — godot #36643](https://github.com/godotengine/godot/issues/36643)
- [iOS/Safari audioContext with wrong sampleRate — howler.js #1141](https://github.com/goldfire/howler.js/issues/1141)
- [Unlock JavaScript Web Audio in Safari and Chrome — Matt Montag](https://www.mattmontag.com/web/unlock-web-audio-in-safari-for-ios-and-macos)
- [getUserMedia — MDN](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia)
- [Permissions-Policy: microphone — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Permissions-Policy/microphone)
- [Microphone permissions in Chrome, Firefox, Safari, and Edge — TestMyAudio](https://testmyaudio.com/microphone-permissions)
- [Using the Permissions API with getUserMedia — addpipe](https://blog.addpipe.com/using-permissions-api-to-detect-getusermedia-responses/)
- [AudioWorklet recording sample — Google Chrome Labs](https://googlechromelabs.github.io/web-audio-samples/audio-worklet/migration/worklet-recorder/)
- [16-bit mono PCM from the browser microphone — Ragy Morkos / Medium](https://medium.com/@ragymorkos/gettineg-monochannel-16-bit-signed-integer-pcm-audio-samples-from-the-microphone-in-the-browser-8d4abf81164d)
- [OfflineAudioContext — MDN](https://developer.mozilla.org/en-US/docs/Web/API/OfflineAudioContext)
- [Audio (expo-audio) — Expo docs](https://docs.expo.dev/versions/latest/sdk/audio/)
- [Audio (expo-av) — Expo docs (deprecated)](https://docs.expo.dev/versions/v54.0.0/sdk/audio-av/)
- [expo-recorder (lodev09) — GitHub](https://github.com/lodev09/expo-recorder)
- [@siteed/expo-audio-stream — npm](https://www.npmjs.com/package/@siteed/expo-audio-stream)
- [Building a Production Audio Recorder with Expo and React Native — DEV](https://dev.to/albert_nahas_cdc8469a6ae8/building-a-production-audio-recorder-with-expo-and-react-native-3h7n)
- [Linear PCM format settings — Apple Developer](https://developer.apple.com/documentation/avfoundation/audio_track_engineering/audio_settings_and_formats/linear_pcm_format_settings)
- [How to record audio using AVAudioRecorder — Hacking with Swift](https://www.hackingwithswift.com/example-code/media/how-to-record-audio-using-avaudiorecorder)
- [Raw Audio Recording with AudioRecord — Android Cookbook](https://www.androidcookbook.info/android-media/raw-audio-recording-with-audiorecord.html)
- [Android raw PCM via AudioRecord → WAV — kmark gist](https://gist.github.com/kmark/d8b1b01fb0d2febf5770)
- [RNNoise — Jean-Marc Valin](https://jmvalin.ca/demo/rnnoise/)
- [xiph/rnnoise — GitHub](https://github.com/xiph/rnnoise)
- [Noise Suppression Guide — Picovoice](https://picovoice.ai/blog/complete-guide-to-noise-suppression/)
- [How to Get the Most Accurate Music Transcriptions — Klangio](https://klang.io/blog/how-to-get-the-most-accurate-music-transcriptions/)
- [Record Your Piano with Shure MV88 — Tommy's Piano Corner](https://tommyspianocorner.com/record-your-piano-great-smartphone-microphone/)
- [How Does Shazam Actually Work? — fonzi.ai](https://fonzi.ai/blog/shazam)
- [How Shazam Works in Python — Michael Strauss](https://michaelstrauss.dev/shazam-in-python/)
- [Audio format conversion cheat sheet — Stefaan Lippens](https://www.stefaanlippens.net/audio_conversion_cheat_sheet/)
- [Audio normalization with FFmpeg — tnonline wiki](https://wiki.tnonline.net/w/Blog/Audio_normalization_with_FFmpeg)
- [slhck/ffmpeg-normalize — GitHub](https://github.com/slhck/ffmpeg-normalize)
- [Automatic Music Transcription: An Overview — Benetos et al.](https://www.eecs.qmul.ac.uk/~simond/pub/2018/BenetosDixonDuanEwert-SPM2018-Transcription.pdf)
- [ML Techniques in Automatic Music Transcription: A Systematic Survey — arXiv 2406.15249](https://arxiv.org/html/2406.15249v1)
- [ReconVAT semi-supervised AMT — KinWaiCheuk/ReconVAT](https://github.com/KinWaiCheuk/ReconVAT)
- [Resample audio to 16kHz — Remotion docs](https://www.remotion.dev/docs/webcodecs/resample-audio-16khz)
