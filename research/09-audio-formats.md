# Audio Formats & Compression

## Summary

For a voice journal, **Opus is the clear winner for storage of recorded entries**: at 24–32 kbps mono it produces speech that sounds as good as MP3 at 96+ kbps, yielding files of roughly 180–240 KB per minute. AAC-LC is a strong second choice when broader device/native compatibility matters (iOS, podcast tooling). For long-term archival of irreplaceable original recordings, keep a FLAC (or WAV) master alongside the lossy distribution copy. Container choice tracks the codec: Opus in WebM (browser/MediaRecorder default) or Ogg, AAC in M4A/MP4.

## Codec Comparison

| Codec | Type | Voice sweet spot | Notes |
|---|---|---|---|
| **Opus** | Lossy | 16–32 kbps mono | Best speech quality per bit; built for real-time + storage; 5 ms latency floor; royalty-free ([Opus Wikipedia](https://en.wikipedia.org/wiki/Opus_(audio_format))) |
| **AAC-LC** | Lossy | 48–64 kbps mono | Native on Apple platforms; ~30% more efficient than MP3 ([Timbrica Blog](https://timbrica.com/en/blog/audio-codec-guide-aac-opus-vorbis-flac)) |
| **MP3** | Lossy | 96–128 kbps mono | Universal compatibility, but inefficient at low bitrates ([HitPaw](https://www.hitpaw.com/other-audio-formats-tips/opus-vs-aac.html)) |
| **FLAC** | Lossless | n/a (~600–900 kbps for 16-bit mono speech) | ~50–60% smaller than WAV with full metadata; best for archival masters ([Cloudinary](https://cloudinary.com/guides/front-end-development/flac-vs-wav-4-key-differences-and-how-to-choose)) |
| **WAV** | Uncompressed PCM | n/a | Bit-perfect, simplest to process; large; weak metadata ([Storii](https://www.storii.com/blog/best-audio-formats-for-long-term-preservation)) |

Listening tests rank Opus higher than MP3, AAC, HE-AAC, and Vorbis at every bitrate up to transparency; at 96 kbps Opus matches AAC at 128 kbps ([Opus Codec comparison](https://www.opus-codec.org/comparison/)).

## Sizing Estimates (1 minute, mono speech)

Using `bytes/min = bitrate_kbps × 60 / 8`:

| Codec @ bitrate | Size / min | Size / 10-min entry | Size / year (10 min/day) |
|---|---|---|---|
| Opus @ 16 kbps | ~120 KB | ~1.2 MB | ~430 MB |
| Opus @ 24 kbps | ~180 KB | ~1.8 MB | ~640 MB |
| Opus @ 32 kbps | ~240 KB | ~2.4 MB | ~870 MB |
| AAC-LC @ 64 kbps | ~480 KB | ~4.8 MB | ~1.7 GB |
| MP3 @ 128 kbps | ~960 KB (~1 MB) | ~9.6 MB | ~3.5 GB |
| FLAC (16-bit/16 kHz mono) | ~5–7 MB | ~50–70 MB | ~20 GB |
| WAV (16-bit/16 kHz mono) | ~1.9 MB | ~19 MB | ~7 GB |

Note: 16 kHz / 16-bit is the standard target for speech-to-text pipelines and captures the full vocal range without waste ([AssemblyAI](https://www.assemblyai.com/blog/best-audio-file-formats-for-speech-to-text)).

## Container Formats

- **Ogg Opus (`.opus`, `.ogg`)** — canonical container for Opus; great for files at rest.
- **WebM (`.webm`)** — what browser `MediaRecorder` produces for Opus; broadly supported on Chrome/Firefox/Edge, and Safari 14.1+ ([MDN Containers](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Containers)).
- **M4A / MP4 (`.m4a`)** — preferred for AAC; iOS-friendly. Safari 18.4+ also accepts Opus in Ogg ([media-codings](https://media-codings.com/articles/recording-cross-browser-compatible-media)).
- **CAF** — Apple-specific; usable for Opus on iOS but niche.
- **FLAC/WAV** — their own self-describing containers.

For cross-browser web capture, WebM/Opus is the path of least resistance; the `opus-media-recorder` polyfill fills gaps where needed ([opus-media-recorder](https://github.com/kbumsik/opus-media-recorder)).

## Trade-offs

- **Lossy vs lossless for archival**: Opus is excellent perceptually but is a "fringer" format vs. AAC/MP3 in commercial standards; if entries are emotionally irreplaceable, retain a FLAC master and ship Opus copies for sync/playback ([Picsart Audio Codec Guide](https://docs.picsart.io/docs/audio-codecs)).
- **Bitrate vs intelligibility**: Opus stays intelligible down to 6–8 kbps; below 24 kbps speech becomes audibly artifacted. 24 kbps mono is the practical floor for "natural-sounding" daily journaling.
- **Compatibility vs efficiency**: MP3 plays everywhere but doubles storage cost vs. Opus for the same speech quality. AAC is the compromise: nearly universal on phones and 30% better than MP3.
- **Re-encoding loss**: never feed lossy → lossy (e.g. Opus → MP3). Keep originals; transcode only from a lossless or the original encode.

## Recommendations for the Voice Journal

1. **Capture**: record at 16 kHz or 24 kHz mono, 16-bit. Browser: `MediaRecorder` with `audio/webm;codecs=opus` at ~32 kbps VBR. Native iOS: AAC-LC 64 kbps in M4A, then re-encode server-side if desired.
2. **Storage (default)**: **Opus in Ogg, 24–32 kbps VBR mono** — ~200 KB/min, transparent for speech, royalty-free, supported by ffmpeg, Whisper, and most STT engines.
3. **Archival tier (optional)**: keep a FLAC copy of entries the user marks as significant; store cold (e.g., S3 Glacier).
4. **AI-narrated playback (TTS output)**: 32–48 kbps Opus mono is plenty; if the "epic story" mode uses music/SFX, bump to 64 kbps stereo.
5. **STT pipeline**: many ASR services accept Opus directly; otherwise decode to 16 kHz PCM WAV in-memory — avoid an MP3 intermediate.
6. **Metadata**: prefer Ogg/FLAC/M4A over WAV so date, mood tags, and transcript references travel with the file.

Sources:
- [Opus (audio format) — Wikipedia](https://en.wikipedia.org/wiki/Opus_(audio_format))
- [Opus Codec comparison page](https://www.opus-codec.org/comparison/)
- [Opus vs AAC 2026 — HitPaw](https://www.hitpaw.com/other-audio-formats-tips/opus-vs-aac.html)
- [Audio Codec Guide — Timbrica](https://timbrica.com/en/blog/audio-codec-guide-aac-opus-vorbis-flac)
- [Best audio formats for speech-to-text — AssemblyAI](https://www.assemblyai.com/blog/best-audio-file-formats-for-speech-to-text)
- [FLAC vs WAV — Cloudinary](https://cloudinary.com/guides/front-end-development/flac-vs-wav-4-key-differences-and-how-to-choose)
- [Best audio formats for long-term preservation — Storii](https://www.storii.com/blog/best-audio-formats-for-long-term-preservation)
- [MDN: Media container formats](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats/Containers)
- [Recording cross-browser compatible media — media-codings](https://media-codings.com/articles/recording-cross-browser-compatible-media)
- [opus-media-recorder polyfill](https://github.com/kbumsik/opus-media-recorder)
- [Audio Codec Guide — Picsart Docs](https://docs.picsart.io/docs/audio-codecs)
