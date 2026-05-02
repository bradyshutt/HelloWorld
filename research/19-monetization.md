# Monetization & Pricing

## Summary

The journaling/voice-AI market in 2025-2026 has converged on freemium subscriptions priced at roughly $50-$100/year for individuals, with a long tail of higher meeting/team tiers ($14-$30/user/month). Unit economics are dominated by speech-to-text, LLM, and (optionally) TTS costs, plus a 15-30% App Store cut; for a daily voice journaler the variable AI cost is on the order of $0.50-$1.00 per active user per month if you stay on cost-efficient models, but TTS narration can quickly 5-10x that. Lifetime deals and BYO-key models have proven viable for indie players (Voicenotes, AudioPen) but most scaled apps default to recurring subscriptions with annual discounts and team tiers. A $7-$10/month price point with an opt-out trial is the most defensible mainstream wedge.

## Pricing Models in Market

- **Day One** – Free Basic; Silver $49.99/yr; Gold $74.99/yr (adds AI features). Android Premium is just $24.99/yr, suggesting platform-level price discrimination. ([dayoneapp.com](https://dayoneapp.com/plans/))
- **Otter.ai** – Free (300 min/mo); Pro $8.33/mo annual ($16.99 monthly); Business $20/user/mo; Enterprise custom. Student/teacher 20% discount. ([Sonix breakdown](https://sonix.ai/resources/otter-ai-pricing/), [Otter pricing](https://otter.ai/pricing))
- **Rosebud** – Free basic; $12.99/mo Premium unlocks long-term memory and voice/call modes. 7-day free trial. Raised $6M in 2025. ([TechCrunch](https://techcrunch.com/2025/06/04/rosebud-lands-6m-to-scale-its-interactive-ai-journaling-app/))
- **AudioPen** – Free (10 notes, 3 min each); Prime $99/yr; previously $120 lifetime deal. Web/Chrome only. ([audiopen.ai/prime](https://www.audiopen.ai/prime))
- **Voicenotes** – $14.99/mo or $99.99/yr (raised from $79 in Oct 2025); $50 lifetime "Believer" deal retired Nov 2024; team plan with no per-seat pricing. ([voicenotes.com/pricing](https://voicenotes.com/pricing), [Lifetimo](https://lifetimo.com/deal/voicenotes-deal/))
- **Granola** – Free Basic (last 30 days only); Business $14/user/mo; Enterprise $35+/user/mo with SSO + training opt-out. No annual billing or minute packs. ([granola.ai/pricing](https://www.granola.ai/pricing))
- **Bonsai** – The agency/freelancer SaaS (~$17/mo, [hellobonsai.com](https://www.hellobonsai.com/pricing)) is not a journaling app; only useful as a comp for higher-priced "all-in-one" framing.

Common patterns: hard paywalls or opt-out trials at $7-$13/mo, ~40% annual discount, free tier capped by minutes/recordings, team/family rare in journaling but standard in meeting apps.

## Unit Economics

Per active user with ~5 minutes of voice/day (~150 min/mo):

- **STT** – OpenAI Whisper / GPT-4o-mini-transcribe at $0.003-$0.006/min ⇒ **$0.45-$0.90/mo**. Google/AWS/Azure are 3-4x more expensive at $0.017-$0.024/min. ([VocaFuse comparison](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/))
- **LLM** – GPT-4o-mini at $0.15/M input + $0.60/M output. A daily 500-word entry plus summary/narration prompt is ~1k in / 500 out ⇒ **<$0.02/mo**. ([OpenAI pricing](https://openai.com/api/pricing/))
- **TTS narration (optional)** – ElevenLabs starts at $5/mo for ~30 min of audio; per-character credits make a 2-min daily narration cost roughly **$2-$5/mo** at retail rates, far more than STT+LLM combined. ([ElevenLabs](https://elevenlabs.io/pricing))
- **Store cut** – Apple/Google take **30%** standard, **15%** under the Small Business Program (≤$1M net proceeds) and after year-1 of any subscription. ([Apple SBP](https://developer.apple.com/app-store/small-business-program/), [RevenueCat](https://www.revenuecat.com/blog/engineering/small-business-program/))

A $7.99/mo plan nets ~$6.79 (15%) or ~$5.59 (30%); AI COGS of $0.50-$1.00 yields ~85% gross margin without TTS, dropping to ~50-60% if every user gets daily TTS narration on premium voices.

## Trade-offs

- **Freemium vs hard paywall** – RevenueCat's 2025 data shows median freemium converts at ~2.2% vs ~12.1% for hard paywalls; opt-out trials hit ~49% vs ~18% opt-in. ([RevenueCat State of Subs 2025](https://www.revenuecat.com/state-of-subscription-apps-2025/))
- **Lifetime deals** – Cash up front and word-of-mouth, but cap LTV and create perpetual COGS liability as AI costs evolve; Voicenotes retired theirs partly to fund infra.
- **BYO-key** – Eliminates pass-through cost risk and lets power users access frontier models, but adds onboarding friction and breaks the App Store IAP requirement on iOS for "digital content." Better for web/desktop or as a power-user tier. ([BYOKList](https://byoklist.com/))
- **TTS cost** – Premium narration is the single biggest margin lever; making it opt-in, weekly (not daily), or lower-tier voice models is essential.

## Recommendations

1. **Anchor at $7.99/mo or $59/yr** with an opt-out 7-day trial — undercuts Day One Gold and AudioPen, sits below Rosebud, and supports 80%+ gross margin on text features.
2. **Free tier**: ~3-5 entries/week, basic transcription + summary, no narration; mirrors Otter/Rosebud and creates a clear upgrade trigger.
3. **Make TTS narration a Pro-tier or credit-metered feature** — use OpenAI/ElevenLabs Flash/Turbo voices (0.5 credits/char) by default and reserve premium voices for higher tiers or weekly "epic recap" episodes.
4. **Skip lifetime; add a Family plan** ($99-$119/yr for 4-5 seats) once you have proof of retention — Apple Family Sharing on auto-renewables is well supported.
5. **Plan for a 15% effective store fee** (Small Business Program from day 1; year-2 subs drop to 15% anyway) and offer a web/Stripe path for power users to recover margin.
6. **Optional BYOK power tier** on web only — appeals to the AudioPen/Voicenotes lifetime-deal demographic without iOS IAP conflicts.

Sources:
- [Day One Plans](https://dayoneapp.com/plans/)
- [Otter.ai Pricing](https://otter.ai/pricing) / [Sonix breakdown](https://sonix.ai/resources/otter-ai-pricing/)
- [Rosebud TechCrunch funding](https://techcrunch.com/2025/06/04/rosebud-lands-6m-to-scale-its-interactive-ai-journaling-app/)
- [AudioPen Prime](https://www.audiopen.ai/prime)
- [Voicenotes Pricing](https://voicenotes.com/pricing)
- [Granola Pricing](https://www.granola.ai/pricing)
- [OpenAI API Pricing](https://openai.com/api/pricing/)
- [Whisper / STT comparison](https://vocafuse.com/blog/best-speech-to-text-api-comparison-2025/)
- [ElevenLabs Pricing](https://elevenlabs.io/pricing)
- [Apple Small Business Program](https://developer.apple.com/app-store/small-business-program/)
- [RevenueCat State of Subscription Apps 2025](https://www.revenuecat.com/state-of-subscription-apps-2025/)
- [BYOKList](https://byoklist.com/)
