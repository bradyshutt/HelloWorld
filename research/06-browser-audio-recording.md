# Browser Audio Recording

## Summary

Modern browsers offer a mature stack for capturing voice in the browser: `getUserMedia` for microphone access, `MediaRecorder` for encoded capture (Opus/WebM, MP4/AAC, and now PCM/ALAC on Safari 18.4+), and the Web Audio API plus `AudioWorklet` for low-latency, sample-level processing. Cross-browser support is finally broad enough to ship a voice journal as a PWA, though iOS Safari still has codec and permission quirks that demand format detection and graceful fallbacks. For a voice journal, the practical pattern is `getUserMedia` + `MediaRecorder` with timeslice chunking for buffered upload, optionally combined with an `AudioWorklet` if you want live VAD, waveform display, or streaming transcription.

## Core APIs

- **getUserMedia (`navigator.mediaDevices.getUserMedia`)** prompts the user and returns a `MediaStream` from the microphone. It requires a secure context (HTTPS or `localhost`) and a user gesture on iOS. ([MDN: getUserMedia](https://developer.mozilla.org/en-US/docs/Web/API/MediaDevices/getUserMedia))
- **MediaRecorder API** consumes a `MediaStream` and emits encoded Blobs via the `dataavailable` event. Pass a `timeslice` argument to `start(ms)` to receive periodic chunks suitable for incremental upload. ([MDN: MediaRecorder](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder), [MDN: dataavailable](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/dataavailable_event))
- **Web Audio API + AudioWorklet** lets you tap the same `MediaStream` via `MediaStreamAudioSourceNode` and run JS/WASM DSP on a dedicated audio thread, giving sub-10ms latency for level meters, VAD, or PCM streaming to a server. ([MDN: Using AudioWorklet](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Using_AudioWorklet), [web.dev: Process microphone audio](https://web.dev/patterns/media/microphone-process))
- **WebRTC `RTCPeerConnection`** is for peer-to-peer real-time media; `RTCDataChannel` carries arbitrary data but is not the standard choice for recording. For a voice journal, MediaRecorder + WebSocket/HTTPS upload is simpler and equally adequate. ([BlogGeek.me: WebRTC recording](https://bloggeek.me/webrtc-recording/))

## Cross-Browser Compatibility

- **Chrome, Edge, Firefox**: full MediaRecorder support. Chrome/Edge default to `audio/webm;codecs=opus`; Firefox supports `audio/ogg;codecs=opus` and `audio/webm;codecs=opus`. ([caniuse: MediaRecorder](https://caniuse.com/mediarecorder))
- **Safari (desktop & iOS)**: MediaRecorder shipped in Safari 14.1, originally only producing `audio/mp4` (AAC). Safari 18.4 (2025) added lossless **ALAC and PCM** in MediaRecorder, plus better codec selection. Some recent iOS Safari builds also support `audio/webm;codecs=opus`, but it should never be assumed. ([WebKit blog: MediaRecorder API](https://webkit.org/blog/11353/mediarecorder-api/), [Pipe: ALAC/PCM in Safari](https://blog.addpipe.com/record-high-quality-audio-in-safari-with-alac-and-pcm-support-via-mediarecorder/))
- **Codec detection**: always probe with `MediaRecorder.isTypeSupported()` before constructing the recorder; Safari frequently returns `false` for `webm/opus` and `true` for `mp4` or `mp4;codecs=mp4a.40.2`. ([MDN: isTypeSupported](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder/isTypeSupported_static))
- **iOS Safari quirks**: HTTPS is mandatory; mic access requires a user gesture; standalone PWAs added to the Home Screen historically lost mic permission and re-prompt sporadically (WebKit bug 185448), though Safari 26 beta improved spurious `devicechange` events. Background tabs may suspend the AudioContext. ([WebKit bug 185448](https://bugs.webkit.org/show_bug.cgi?id=185448), [WebKit: Safari 26 beta news](https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/))
- **Polyfills**: `opus-media-recorder` (WASM Opus encoder) and `RecordRTC` smooth over codec gaps if you must guarantee Opus output everywhere. ([opus-media-recorder](https://github.com/kbumsik/opus-media-recorder), [RecordRTC](https://recordrtc.org/))

## Streaming vs Buffered

- **Buffered (one Blob at stop)**: call `recorder.start()` with no argument; you get one well-formed container at `stop()`. Simplest and most compatible — ideal for short journal entries.
- **Chunked timeslice**: `recorder.start(timesliceMs)` fires `dataavailable` every N ms. Each chunk is a fragment of the same container, so the server must concatenate them in order to produce a playable file. Browsers don't honor the interval exactly, and very small timeslices (<250 ms) can produce huge or missing chunks. ([Pipe: huge MediaRecorder slices](https://blog.addpipe.com/dealing-with-huge-mediarecorder-slices/))
- **True real-time streaming (e.g., for live transcription)**: bypass MediaRecorder and use an AudioWorklet to ship raw 16-kHz PCM frames over WebSocket — this is the pattern used for streaming Whisper/Transcribe pipelines. ([AWS: Stream audio to Transcribe](https://aws.amazon.com/blogs/machine-learning/stream-multi-channel-audio-to-amazon-transcribe-using-the-web-audio-api/), [Streaming Whisper over WebSocket](https://medium.com/@david.richards.tech/how-to-build-a-streaming-whisper-websocket-service-1528b96b1235))

## Recommendations

1. **Default capture**: `getUserMedia({ audio: { echoCancellation: true, noiseSuppression: true, autoGainControl: true } })` triggered from a tap/click.
2. **Format strategy**: probe `isTypeSupported` in this order — `audio/webm;codecs=opus` → `audio/ogg;codecs=opus` → `audio/mp4;codecs=mp4a.40.2` → `audio/mp4` → default. Store the negotiated MIME alongside the recording so the server knows how to transcode.
3. **Upload model**: for journal entries under ~2 minutes, record buffered and upload one Blob; for longer sessions, use a 5–10 s timeslice and stream chunks over `fetch` or WebSocket.
4. **Live features**: add an `AudioWorklet` tap for waveform/VAD UI without disturbing the MediaRecorder pipeline (both can read the same `MediaStream`).
5. **PWA hardening**: serve over HTTPS, require a user gesture before `getUserMedia`, handle `permissions.query({ name: 'microphone' })`, and re-request access on iOS standalone mode. Test "Add to Home Screen" flows explicitly because they regress most often.
6. **Server-side transcoding**: normalize all uploads to Opus-in-Ogg or 16-kHz mono WAV before feeding ASR — never assume the browser-chosen container.

Sources:
- [MDN: MediaRecorder](https://developer.mozilla.org/en-US/docs/Web/API/MediaRecorder)
- [MDN: MediaStream Recording API](https://developer.mozilla.org/en-US/docs/Web/API/MediaStream_Recording_API)
- [MDN: AudioWorklet](https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet)
- [WebKit: MediaRecorder API](https://webkit.org/blog/11353/mediarecorder-api/)
- [WebKit: Safari 26 beta](https://webkit.org/blog/16993/news-from-wwdc25-web-technology-coming-this-fall-in-safari-26-beta/)
- [Pipe: ALAC/PCM in Safari](https://blog.addpipe.com/record-high-quality-audio-in-safari-with-alac-and-pcm-support-via-mediarecorder/)
- [caniuse: MediaRecorder](https://caniuse.com/mediarecorder)
- [opus-media-recorder polyfill](https://github.com/kbumsik/opus-media-recorder)
