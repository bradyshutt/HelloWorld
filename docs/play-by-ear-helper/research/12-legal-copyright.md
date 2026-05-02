# Legal & Copyright Considerations

## Summary

Automatic transcription of copyrighted recordings into sheet music is **legally murky but operationally tolerated** for personal, non-distributed use. The transcription itself is almost certainly a "derivative work" under US 17 U.S.C. §106(2) and equivalent EU rules — fair use is a plausible (but unproven) defense for end-users, *not* for the service that creates and hands them the score. The real risk is not "is this technically legal" but "does a major rightsholder (NMPA, MPA, RIAA, a publisher) decide to send a takedown / lawsuit." Existing competitors (Klangio, AnthemScore, ScoreCloud, Chordify) survive by (a) framing output as "for personal use," (b) shifting liability to the user via ToS, (c) responding to takedowns, and in Chordify's case (d) holding actual licenses with publishers. This is a real but manageable risk if we follow the same playbook; it is **not** legal advice and a lawyer should review before launch in either market.

## Copyright & Fair Use

### The core legal question

Sheet music produced from an audio recording reproduces the underlying **musical composition** (notes, chords, rhythm, structure) — this is one of two distinct copyrights in any recorded song (composition vs. sound recording). Under 17 U.S.C. §106, the composition copyright owner has exclusive rights to **reproduce**, **distribute**, and **prepare derivative works**. A machine-generated transcription almost certainly hits all three when shared.

Sources are unanimous that:
- Transcribing a copyrighted recording into notation creates a **derivative work** (US Copyright Office; Music Publishers Association; MTNA copyright FAQ).
- Distribution / sale without a **print license** from the publisher is infringement.
- Creating a transcription purely for personal study is a **gray area** that is rarely enforced but is technically not authorized; "personal use" is *not* an enumerated exemption in US copyright law the way it is in some EU countries.

### Fair use (US, 17 U.S.C. §107)

Four-factor test as applied to our use case:
1. **Purpose and character** — A consumer using the app to learn a song themselves leans non-commercial and arguably transformative (audio → notation is a format shift for study). For *us as the service operator*, the use is commercial, which weighs against.
2. **Nature of the work** — Music is highly creative; weighs against fair use.
3. **Amount used** — Producing notation for an entire song captures the *entire composition*. Strongly against fair use.
4. **Effect on the market** — Sheet music IS a licensed market (print rights via Hal Leonard, Alfred, Music Sales, Sheet Music Plus, Musicnotes). Free auto-transcription substitutes directly. Strongest factor against.

Conclusion: a fair-use defense for the *service* generating distributable PDFs is weak. A fair-use defense for an *individual user* learning at home is plausible but untested.

### EU "private copying" exception

Most EU member states (under InfoSoc Directive 2001/29/EC Art. 5(2)(b)) allow personal-use copies for natural persons, often funded by levies on storage media. This is generally interpreted to permit *individual users* to transcribe a song they already own for their own study. It does NOT permit a third-party service to do the transcribing at scale, and it does not survive distribution.

### EU Copyright Directive Article 17 (DSM Directive 2019/790)

If we let users share their transcriptions on our platform, we likely become an "online content-sharing service provider" (OCSSP). That triggers:
- Best-effort obligation to obtain licenses from rightsholders.
- Best-effort to prevent availability of works rightsholders have specifically flagged ("upload filters" in practice).
- Expeditious takedown on notice.
- Exceptions for quotation, criticism, parody — **not** wholesale transcription.

If we keep output strictly user-private (no sharing feature), Article 17 does not directly apply, but national copyright laws still do.

### Distribution / derivative work issues

Selling, sharing publicly, posting to social, or even emailing transcribed scores of copyrighted songs is clearly infringing without a print license. Print licenses are administered by publishers directly or via Hal Leonard / Alfred / ArrangeMe. There is no statutory print-license analogue to §115 (mechanical) for sheet music — you must negotiate.

### Mechanical, performance, and print rights — what applies?

- **Mechanical (§115 / HFA / MLC):** Covers reproduction in audio formats. Transcription to *notation* is **not** a mechanical use, so §115 does not help us.
- **Performance (ASCAP / BMI / SESAC / GMR / PRS / GEMA):** Covers public performance. Our app does not publicly perform. **Not triggered.**
- **Print / Display rights:** This IS the relevant right. Held by publishers; no compulsory license. Klangio, etc. do not have these and operate in the gray zone.
- **Sound recording / master rights (RIAA labels):** We are not reproducing the master in our output, but we ARE processing it server-side if uploads occur, which is itself a (transient) reproduction.

### Precedent cases

- **No major reported lawsuit** has yet been filed specifically against an audio-to-sheet-music transcription service. The space has remained small and tolerated.
- **RIAA v. Suno / Udio (2024)** and **UMG / Concord / ABKCO v. Anthropic** are the closest analogs: both target AI services that *trained on* or *output* copyrighted material. The industry's appetite for AI-music litigation has clearly increased since 2024 (Music Business Worldwide; RIAA filings). A successful Klangio-style product at scale could become a target.
- **Tab/chord sites:** OLGA (On-Line Guitar Archive) was shut down in 2006 after MPA / NMPA pressure. Ultimate-Guitar and Chordify ultimately resolved this by striking licensing deals.
- **MP3tunes / EMI v. MP3tunes (2011, 2nd Cir. 2016):** Established that "red flag" knowledge of infringement at a service breaks DMCA safe harbor.
- **Viacom v. YouTube (2012, 2nd Cir.):** Confirmed safe harbor requires actual or red-flag knowledge of *specific* infringing items.

## How Existing Services Handle It

| Service | Approach |
| --- | --- |
| **Klangio** (klang.io, Germany) | Markets to musicians for personal practice. ToS pushes copyright responsibility onto users. Output is private to the user account. Does not appear to hold a blanket print license; relies on the personal-use framing and lack of distribution. |
| **AnthemScore** (Lunaverus, US) | Desktop software — user runs it on their own machine on files they already have. Vendor never sees the audio. This **dramatically** reduces vendor exposure: AnthemScore is more like a word processor than a service. |
| **ScoreCloud** (DoReMIR, Sweden) | Cloud-based but largely positioned for users transcribing their *own* singing/playing. Marketing emphasizes original composition, sidestepping (in marketing if not in reality) the copyrighted-input use case. |
| **Chordify** (Netherlands) | Most aggressive: actually displays chords for popular songs to all users. Operates under a **license deal with rightsholders** (announced licenses with major publishers and PROs over the years). Provides a `retract@chordify.net` opt-out for rightsholders. Not a model we can replicate cheaply — those licenses are expensive. |
| **Moises, LALAL.AI, Songscription.ai** | ToS grants limited personal-use license, disclaims warranties, requires user to represent they have rights to uploaded content. |

Common ToS pattern across all of them:
- User warrants they have rights to anything they upload.
- License granted to user is "limited, non-exclusive, non-transferable, personal, non-commercial."
- Service disclaims liability for user-side infringement.
- DMCA designated agent listed.
- Repeat-infringer policy stated.

## App Store / Play Store Policies

### Apple App Store

- **Guideline 5.2 (Intellectual Property):** "Make sure your app only includes content that you created or that you have a license to use." Apple does **not** pre-screen for music copyright but acts on credible complaints.
- DMCA-style takedowns via apple.com/legal/contact/copyright-infringement.html. AppleInsider (Nov 2024) reported Apple has been removing apps quickly on copyright claims, sometimes with limited appeal — a real operational risk for us.
- Music-recognition / processing apps (Shazam, SoundHound, Moises) are accepted, so the category is fine in principle.

### Google Play

- Play's "Intellectual Property" policy mirrors Apple's: don't infringe; respond to DMCA.
- Google's takedown response is generally slower but they have removed apps for copyright claims.

### Practical implication

Either store can pull the app on a single credible complaint. We need: (a) a registered DMCA agent, (b) a public takedown procedure, (c) responsive ops, (d) ToS that clearly puts upload responsibility on users.

## Privacy (GDPR, Audio Retention)

### GDPR applicability

Audio uploads are personal data **if** they contain identifiable voice (singing, talking, ambient identifying sounds). Pure music recordings without human voice may fall outside GDPR's "personal data" scope but in practice we should assume GDPR applies because:
- IP addresses, account IDs, device IDs are personal data on their own.
- We cannot reliably pre-classify whether an upload contains identifying audio.

### Required compliance items

- **Lawful basis** — likely Art. 6(1)(b) "performance of contract" for the core transcription function plus Art. 6(1)(a) consent for any optional analytics / training-data use.
- **Special-category caveat** — voice biometrics can be Art. 9 special-category data. Avoid using audio for any biometric/identification purpose.
- **Privacy notice** — must explain what audio is processed, why, retention period, sub-processors (cloud GPU provider, etc.), data subject rights.
- **Data subject rights** — access, deletion, portability within 30 days.
- **DPA** with any sub-processor (e.g., AWS, GCP, Modal, Replicate).
- **DPIA** likely required: large-scale processing of audio could be "systematic monitoring" or use "new technology" per Art. 35.
- **Cross-border transfers** — SCCs or adequacy decisions if processing outside EEA.

### Audio retention policy (recommended)

- **Default:** delete uploaded audio immediately after transcription completes. Keep only the resulting score (which the user owns the rendering of) and minimal metadata.
- If keeping audio for debugging / model improvement: retain max 30 days, encrypted at rest, access-logged, with explicit opt-in consent and a clear UI toggle.
- Never use uploaded user audio to train models without explicit, separately-presented consent (post-Suno/Udio, this is the single clearest litigation target).

### CCPA / CPRA (California)

Similar to GDPR in spirit; right to know, delete, opt out of "sale/share." Audio is personal information. Disclosures needed in privacy policy.

## Music Fingerprinting / Moderation

ACRCloud, AudD, and ShazamKit can identify uploaded recordings against catalogs of commercial music. Options:

1. **Don't fingerprint, don't ask.** "Willful blindness" can break DMCA safe harbor (per case law). However, simply not deploying detection technology is not itself willful blindness — courts require something closer to deliberate avoidance of specific known infringement.
2. **Fingerprint everything, allow only non-commercial.** Costs money per query, creates UX friction, and creates a record that we *knew* user X uploaded copyrighted track Y — that record becomes discoverable in litigation.
3. **Fingerprint to log & comply.** Identify uploads, attach metadata to the resulting score so we can honor takedowns at the per-song level.

Recommendation: **don't fingerprint at upload** for privacy and cost reasons, **but** support fingerprint-based takedown lookups if a rightsholder specifically asks us to suppress a song.

## DMCA Safe Harbor (US, 17 U.S.C. §512)

To qualify under §512(c) (user-stored content):
1. **Designated agent** registered with the US Copyright Office ($6 fee, renewable every 3 years).
2. **Repeat infringer policy** adopted, published, and reasonably enforced (account termination after N strikes).
3. **Notice-and-takedown** workflow with prompt action on valid notices.
4. **No actual or red-flag knowledge** of specific infringement; no willful blindness.
5. **No direct financial benefit** from infringement when we have the right and ability to control it. This is the trickiest prong: a paid-tier app *might* be argued to derive financial benefit from users transcribing hits.
6. **Accommodate standard technical measures.**

We probably qualify for §512(c) for *user-uploaded audio*, BUT the *output* (the score) is generated by our system, which is closer to §512(a) conduit / §512(b) caching analyses don't cleanly fit. Some courts would treat AI-generated derivatives as the service's own content, removing safe harbor entirely. This is unsettled law.

EU equivalent: Article 14 of the e-Commerce Directive 2000/31/EC (hosting safe harbor), now overlaid with Article 17 DSM Directive obligations. Article 17 effectively *removes* the hosting safe harbor for OCSSPs that host UGC of copyrighted works.

## Recommended Approach

Based on all of the above, the practical posture for Play by Ear Helper:

1. **Position as a personal-use tool.** No public sharing, no marketplace, no distribution feature in v1. Output is delivered to the uploading user only.
2. **Keep audio ephemeral.** Delete uploaded audio within minutes of transcription. Document this in the privacy policy and surface it in the UI ("we never keep your audio").
3. **No training on user uploads** without explicit, separate, opt-in consent — and even then, prefer not to.
4. **Strong ToS clauses:** user warrants they have the right to upload; user agrees output is for personal study only; service is provided "as is"; user indemnifies us; choice of law and arbitration clauses.
5. **Register a DMCA agent** with the US Copyright Office before launch. Publish the takedown procedure. Adopt and enforce a repeat-infringer policy.
6. **EU posture:** restrict the app to a no-sharing model so we are arguably not an OCSSP under Article 17. Publish a GDPR-compliant privacy policy and DPIA. Use an EU-resident sub-processor or SCCs.
7. **Be ready to remove songs.** Build a per-song suppression list keyed to ISRC / fingerprint so we can comply with rightsholder requests quickly without litigating each one.
8. **Avoid bait.** Don't market with major artists' names ("transcribe Taylor Swift!"). Use generic copy.
9. **Talk to a lawyer** before launch — specifically a US copyright/IP lawyer and an EU privacy/copyright lawyer. The cost of an hour of advice is far less than a single takedown war.
10. **Watch the AI-music litigation wave.** Suno/Udio outcomes and the Anthropic music-publishers case will materially change the risk profile in the next 12-24 months.

Risk rating overall: **moderate**. The activity (audio-to-notation) has been quietly tolerated for over a decade across multiple competitors. The main escalation risks are (a) growing big enough to attract NMPA's attention and (b) the broader anti-AI-music legal climate spilling over.

## Sources

- US Copyright Office — What Musicians Should Know about Copyright: https://www.copyright.gov/engage/musicians/
- US Copyright Office — Section 512 Resources: https://www.copyright.gov/512/
- 17 U.S.C. §512 (Cornell LII): https://www.law.cornell.edu/uscode/text/17/512
- Music Publishers Association — Quick Guide to Copyright: https://www.mpa.org/quick-guide-to-copyright/
- MTNA Copyright FAQs: https://www.mtna.org/MTNA/Learn/Copyright_FAQs.aspx
- ArrangeMe — Common Questions about Copyrighted Arrangements: https://blog.arrangeme.com/blog/common-questions-about-copyrighted-arrangements
- Copyright Alliance — DMCA Safe Harbor: https://copyrightalliance.org/education/copyright-law-explained/the-digital-millennium-copyright-act-dmca/dmca-safe-harbor/
- Fenwick — DMCA Safe Harbor Q&A (PDF): https://assets.fenwick.com/legacy/FenwickDocuments/DMCA-QA.pdf
- CRS — DMCA Safe Harbor Legal Overview: https://www.congress.gov/crs-product/IF11478
- EFF — DMCA: https://www.eff.org/issues/dmca
- Wikipedia — Online Copyright Infringement Liability Limitation Act: https://en.wikipedia.org/wiki/Online_Copyright_Infringement_Liability_Limitation_Act
- EU Commission — Guidance on Article 17: https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:52021DC0288
- Wikipedia — Directive on Copyright in the Digital Single Market: https://en.wikipedia.org/wiki/Directive_on_Copyright_in_the_Digital_Single_Market
- TermsFeed — Complying with Article 17: https://www.termsfeed.com/blog/eu-copyright-directive-article-17/
- Recording Academy — EU Copyright Reform: https://www.recordingacademy.com/advocacy/news/european-union-passes-game-changing-copyright-reform-heres-why-its-huge-deal
- IAPP — How rules on audio recording change under GDPR: https://iapp.org/news/a/how-do-the-rules-on-audio-recording-change-under-the-gdpr
- GDPR Advisor — Voice-to-Text Compliance: https://www.gdpr-advisor.com/gdpr-compliance-for-voice-to-text-services-and-transcription-platforms/
- Picovoice — GDPR, CCPA and Voice Recognition Privacy: https://picovoice.ai/blog/gdpr-ccpa-voice-recognition-privacy/
- TermsFeed — Privacy Policy for Audio Recordings: https://www.termsfeed.com/blog/privacy-policy-audio-capture-recordings/
- Apple — App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Apple — Copyright Infringement Contact: https://www.apple.com/legal/contact/copyright-infringement.html
- Apple — Music Dispute Forms: https://www.apple.com/legal/intellectual-property/dispute-forms/music/
- AppleInsider — App Store takedowns over copyright (Nov 2024): https://appleinsider.com/articles/24/11/20/apples-quick-app-store-takedowns-over-copyright-claims-are-a-nightmare-for-developers
- Klangio: https://klang.io/
- AnthemScore (Lunaverus): https://www.lunaverus.com/
- ScoreCloud: https://scorecloud.com/
- Chordify support — copyright Q&A: https://support.chordify.net/hc/en-us/articles/360001420738-Does-this-not-infringe-copyright
- Songscription.ai Terms of Service: https://www.songscription.ai/terms
- ASCAP Licensing FAQs: https://www.ascap.com/help/ascap-licensing
- ACRCloud (audio recognition / copyright compliance): https://www.acrcloud.com/
- AudD Music Recognition API: https://audd.io/
- Wang (2003) — Industrial-Strength Audio Search Algorithm (Shazam paper): https://www.ee.columbia.edu/~dpwe/papers/Wang03-shazam.pdf
- Milvus — Copyright issues in audio search: https://milvus.io/ai-quick-reference/how-are-copyright-issues-addressed-in-audio-search-implementations
- RIAA v. Udio complaint (2024): https://www.riaa.com/wp-content/uploads/2024/06/Udio-Complaint-6.24.241.pdf
- RIAA — Suno / Udio cases announcement: https://www.riaa.com/record-companies-bring-landmark-cases-for-responsible-ai-againstsuno-and-udio-in-boston-and-new-york-federal-courts-respectively/
- Music Ally — RIAA sues Suno and Udio: https://musically.com/2024/06/24/labels-body-riaa-sues-ai-music-firms-suno-and-udio-for-copyright-infringement/
- Music Business Worldwide — RIAA/NMPA amicus brief (Anthropic): https://www.musicbusinessworldwide.com/riaa-nmpa-and-more-file-amicus-brief-backing-music-publishers-against-anthropic-arguing-ai-companys-unlicensed-copying-is-inexcusable/
- Wikipedia — List of songs subject to plagiarism disputes: https://en.wikipedia.org/wiki/List_of_songs_subject_to_plagiarism_disputes

---

*This document is research-stage notes for project planning. It is **not legal advice**. Engage a qualified copyright/IP attorney (US) and a privacy/copyright attorney (EU) before launch.*
