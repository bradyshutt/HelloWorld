# Local-First vs Cloud Architecture

## Summary

A voice journal sits at an awkward seam: recordings are large binary blobs that benefit from local capture, while transcription and stylistic narration are compute-heavy and cheaper in the cloud. The pragmatic 2026 default is a **local-first** capture layer (device owns the audio + journal text, app works offline) paired with a **hybrid compute** model where Whisper-class STT runs on-device when possible and falls back to a cloud API for long sessions or weak hardware. Sync engines like PowerSync, ElectricSQL, and Replicache (or CRDT libs like Automerge/Yjs) handle the structured journal metadata, while audio files travel through a separate object-store path (S3/R2/Supabase Storage). This split keeps per-user costs near-zero on the hot path and avoids both vendor lock-in and the privacy footgun of streaming raw audio.

## Architectural Options

1. **Pure cloud** — record on device, upload audio, transcribe and store server-side. Simplest backend story, but offline UX is poor, every minute of audio costs money, and recordings traverse the network even when users never share them.
2. **Pure local** — everything (audio, transcription, narration) on-device. Maximum privacy (see [Whisper Notes](https://whispernotes.app/)), but rules out a multi-device journal, fun shareable narrations, and weaker phones.
3. **Local-first hybrid (recommended)** — device is the source of truth for journal entries; cloud is a sync target and an optional compute backend. Aligns with [Ink & Switch's seven ideals](https://www.inkandswitch.com/essay/local-first/): fast, multi-device, offline, collaborative-ready, longevity, privacy, user-owned data.

## Sync Engines

For the structured layer (entry rows, tags, timestamps, transcripts):

- **[PowerSync](https://www.powersync.com/)** — Postgres/Mongo/MySQL backend synced to client SQLite, true bidirectional sync with an upload queue, and a built-in [attachment helper](https://docs.powersync.com/client-sdks/advanced/attachments) that pairs metadata sync with S3/R2/Supabase blob storage. Best fit for a mobile-first journal with audio files.
- **[ElectricSQL](https://electric-sql.com/docs/reference/alternatives)** — read-path streaming from Postgres via "Shapes"; writes go through your own API. Lighter and simpler, but you build the audio-upload path yourself.
- **[Replicache / Zero](https://queryplane.com/docs/blog/electricsql-vs-powersync-vs-replicache)** — client-library only, you implement push/pull endpoints. Excellent UX latency, more backend code to write.
- **[Yjs / Automerge](https://velt.dev/blog/best-crdt-libraries-real-time-data-sync)** — CRDT libraries, ideal if a transcript becomes a collaboratively-editable document. [Automerge 3.0 cut memory ~10x](https://biggo.com/news/202508071934_Automerge_3.0_Memory_Improvements), making it viable on phones, but it is overkill for append-only journal entries.

Audio itself should not live in the CRDT/SQLite stream — it should ride a separate object-storage channel keyed by entry ID, as [PowerSync's attachment pattern](https://docs.powersync.com/client-sdks/advanced/attachments) demonstrates.

## Trade-offs

- **Where transcription runs.** [On-device Whisper](https://whispernotes.app/blog/offline-speech-to-text-complete-guide) runs ~5x real-time on an iPhone 15 Pro and costs $0/min after install, but ships a 75 MB–3 GB model and drains battery. Cloud Whisper is [$0.006/min](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/), Deepgram/AssemblyAI similar; Google/AWS land at $0.024/min. A 5-minute daily entry per user costs ~$0.90/year on cloud Whisper — trivial unless scale is huge.
- **Where narration runs.** TTS and LLM rewrites (epic-story mode) realistically need cloud; on-device LLMs aren't yet good enough for stylized prose at journal length.
- **Privacy & ownership.** Local-first keeps raw audio off your servers, shrinking compliance surface. If transcription is cloud-only, send audio ephemerally and never persist it server-side.
- **Cost shape.** Local-first flips the cost curve from per-minute-per-user (cloud) to one-time engineering + storage. Sync engines like PowerSync charge per active device/data volume rather than per audio minute.
- **Complexity.** Sync engines remove the "build your own offline queue" burden but add a dependency. Pure CRDT stacks (Yjs/Automerge) are more flexible but require designing your own transport.

## Recommendations

For this voice journal:

1. **Local-first storage** — SQLite on device holds entries, transcripts, and metadata; audio files live in app-local storage with a pointer row.
2. **PowerSync (or ElectricSQL if you want lighter)** to sync the metadata/transcript table to Postgres; use its attachment helper to mirror audio blobs to S3/R2 only when the user opts into multi-device or backup.
3. **Hybrid STT** — try on-device Whisper (Core ML / whisper.cpp) first; fall back to the [OpenAI Whisper API at $0.006/min](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/) when the device can't keep up or the user is on an older phone.
4. **Cloud-only for narration** — stylized "epic story" rewrites and TTS run server-side, triggered on demand, cached locally so replays are free and offline.
5. **Treat the cloud as a peer, not the source of truth**, in the spirit of [Ink & Switch's local-first essay](https://www.inkandswitch.com/essay/local-first/) — the app must remain fully usable on a plane.

## Sources

- [Ink & Switch — Local-first software](https://www.inkandswitch.com/essay/local-first/)
- [PowerSync — Local-First Software](https://docs.powersync.com/resources/local-first-software)
- [PowerSync Attachments docs](https://docs.powersync.com/client-sdks/advanced/attachments)
- [ElectricSQL alternatives page](https://electric-sql.com/docs/reference/alternatives)
- [QueryPlane — ElectricSQL vs PowerSync vs Replicache](https://queryplane.com/docs/blog/electricsql-vs-powersync-vs-replicache)
- [BuildPilot — ElectricSQL vs PowerSync vs Zero (2026)](https://trybuildpilot.com/648-electric-sql-vs-powersync-vs-zero-2026)
- [Velt — Best CRDT Libraries 2025](https://velt.dev/blog/best-crdt-libraries-real-time-data-sync)
- [BigGo — Automerge 3.0 memory improvements](https://biggo.com/news/202508071934_Automerge_3.0_Memory_Improvements)
- [VocaFuse — STT API price comparison 2025](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/)
- [Whisper Notes — Offline Whisper guide](https://whispernotes.app/blog/offline-speech-to-text-complete-guide)
- [Talkio AI — On-device vs Cloud STT tradeoffs](https://voicecontrol.chat/blog/posts/on-device-speech-to-text-vs-cloud-apis-tradeoffs-for-privacy-and-performance)
- [Expo — Local-first architecture guide](https://docs.expo.dev/guides/local-first/)
