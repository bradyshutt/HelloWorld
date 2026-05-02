# Privacy & Security for Voice Journals

## Summary

Voice journal entries are doubly sensitive: they contain raw biographical content *and* a biometric voiceprint, which under Illinois BIPA, GDPR (Art. 9 special category data), and CCPA/CPRA (sensitive personal information) requires explicit, informed, opt-in consent and a published retention/destruction schedule. The 2025 wave of BIPA voiceprint class actions (Microsoft Teams, Otter.ai, Nuance) and reports that OpenAI's Whisper API silently retains audio for ~30 days for "abuse monitoring" make third-party transcription a serious liability surface. The defensible architecture for a personal journal is on-device capture + on-device or self-hosted transcription, client-side encryption (AES-256 + per-user keys) before any sync, short default retention windows, and consent UX that separates storage, transcription, and AI-narration purposes. Treat voice as biometric data by default and design for data minimization, easy export, and one-tap deletion.

## Regulatory Landscape

- **BIPA (Illinois)**: A "voiceprint" is an enumerated biometric identifier. Written, informed consent is required *before* capture, and the operator must publish a retention schedule with a guaranteed destruction date (the earlier of purpose-fulfilled or 3 years after last interaction). Statutory damages are $1,000 (negligent) / $5,000 (reckless) per violation, though the 2024 amendment (held retroactive by the Seventh Circuit in 2026) caps recovery to one violation per person. 2025 saw 100+ new BIPA class actions, including voiceprint cases against Microsoft Teams and Otter.ai ([Privacy World 2025 Year-In-Review](https://www.privacyworld.blog/2025/12/2025-year-in-review-biometric-privacy-litigation/), [YPAI BIPA Voice Guide 2025](https://ypai.ai/en/guides/bipa-voice-data-compliance/)).
- **GDPR**: Voice recordings are personal data; voiceprints used for unique identification are Art. 9 special-category data requiring explicit consent. Article 30 ROPA, a DPIA for biometric processing, encryption in transit and at rest, and easy withdrawal are mandatory ([GDPR Advisor – Voice-to-Text Compliance](https://www.gdpr-advisor.com/gdpr-compliance-for-voice-to-text-services-and-transcription-platforms/), [Speechmatics 2026 Voice AI Compliance Guide](https://www.speechmatics.com/company/articles-and-news/your-essential-guide-to-voice-ai-compliance-in-todays-digital-landscape)).
- **CCPA/CPRA**: Voiceprints are "sensitive personal information"; users gain rights to know, delete, correct, and limit use, plus a "Limit the Use of My Sensitive Personal Information" link ([TermsFeed – CCPA Biometrics](https://www.termsfeed.com/blog/ccpa-biometrics/)).

## Threat Model

- **Cloud provider exfiltration / silent retention**: A November 2025 investigation found OpenAI's Whisper API stores audio for up to 30 days despite "zero retention" marketing ([Basil AI report](https://basilai.app/articles/2025-11-17-chatgpt-whisper-api-storing-voice-recordings-basil-ai-private-alternative.html)).
- **Device compromise**: Lost/stolen phone with unencrypted journal cache.
- **Account takeover**: Reused passwords / no MFA exposes the entire life log.
- **Voiceprint extraction by upstream vendors**: Speaker diarization features can derive a biometric identifier even when the developer never asked for one — the Otter.ai and Teams suits both turn on this ([Hunton – Whole Foods voiceprint settlement](https://www.hunton.com/privacy-and-cybersecurity-law-blog/whole-foods-settles-bipa-voiceprint-class-action0)).
- **Replay / spoofing / "voice squatting"**: Adversary triggers entries or extracts data via crafted audio ([ACM Survey on Voice Assistant Security](https://dl.acm.org/doi/10.1145/3527153)).
- **PII leakage in transcripts**: Names, addresses, health details get embedded in plaintext logs and analytics pipelines.
- **Model-training reuse**: Provider trains future models on user audio without separable consent.

## Architectural Patterns

- **On-device first**: Apple Speech, whisper.cpp, MLX-Whisper, and similar engines keep raw audio on the handset, eliminating cross-border transfer and provider retention risk; trade-off is ~1-2 min processing per 10-min clip on a recent phone ([Talkio AI – On-Device vs Cloud](https://voicecontrol.chat/blog/posts/on-device-speech-to-text-vs-cloud-apis-tradeoffs-for-privacy-and-performance), [VoicePrivate](https://voiceprivate.com/blog/on-device-vs-cloud-transcription)).
- **Client-side / E2EE storage**: AES-256-GCM for blobs, X25519 for key exchange, per-user keys derived from a passphrase + device secure enclave; the server stores ciphertext only (Day One, Standard Notes patterns) ([Day One E2EE FAQ](https://dayoneapp.com/guides/day-one-sync/end-to-end-encryption-faq/), [DevTechInsights E2EE 2025](https://devtechinsights.com/wp-content/uploads/2025/08/End-to-End-Encryption-for-Developers-Best-Practices-in-2025-1.pdf)).
- **Hybrid with redaction**: If cloud transcription is required for quality, run automatic PII redaction before storage (Deepgram offers this at $0.002/min, SOC2 Type II + ISO 27001) and send only redacted text to the narration LLM ([Deepgram Speech-to-Text Privacy](https://deepgram.com/learn/speech-to-text-privacy)).
- **Disable voiceprint/diarization** unless the user explicitly opts in — this is the specific feature driving BIPA suits.
- **Zero-Data-Retention (ZDR) endpoints + DPAs** with any LLM/STT vendor; explicitly forbid training on user content.
- **Tiered retention**: raw audio purged in 24-72 h once transcript is verified; transcripts retained per user preference (default 90 days for "logs", user-controlled for journal entries); automated deletion jobs with retention labels per [SecurePrivacy GDPR Consent 2026](https://secureprivacy.ai/blog/gdpr-consent-management).

## Recommendations

1. **Default to on-device transcription**; treat cloud STT as an opt-in upgrade with a clear notice naming the vendor.
2. **Layered, granular consent**: separate toggles for (a) recording, (b) cloud transcription, (c) voiceprint/speaker-ID, (d) AI narration, (e) model-training. No pre-checked boxes; capture and timestamp each consent event.
3. **Encrypt before sync**: client-side AES-256-GCM with keys derived from user passphrase + device keystore; server never sees plaintext or keys.
4. **Publish a BIPA-style retention & destruction schedule** even if you don't think you have voiceprints — courts have been finding incidental voiceprint creation in transcription pipelines.
5. **Run a DPIA** before launch (mandatory under GDPR for biometric/large-scale personal processing) and maintain a ROPA.
6. **Vendor due diligence**: signed DPA, ZDR confirmation in writing, SOC2/ISO 27001, EU data residency option, no-training clause. Avoid Whisper-API-as-a-service for raw audio; prefer self-hosted Whisper or Deepgram/AssemblyAI with ZDR ([AssemblyAI alternatives 2025](https://www.assemblyai.com/blog/google-cloud-speech-to-text-alternatives)).
7. **One-tap export and delete** (CCPA/GDPR DSAR-ready), plus a persistent in-app consent dashboard.
8. **Local-only "incognito entry"** mode that bypasses sync entirely.
9. **Auth hygiene**: device biometric unlock for the app itself, MFA on the account, rate-limit and re-prompt before any AI narration share.
10. **Be explicit in the privacy policy** that journal data and voiceprints are never used to train models — this is the single biggest trust signal users look for in 2025-2026 voice products.
