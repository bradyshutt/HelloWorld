# Cost Analysis

## Summary

For the MVP using open-source models (Demucs v4 + Basic Pitch) on a serverless GPU, the marginal cost per 3.5-minute song is roughly **$0.005 to $0.02** of compute, with storage/bandwidth adding under $0.001/song if Cloudflare R2 is used. Commercial APIs are dramatically more expensive: a single song through Klangio is roughly **$2-$4**, AudioShake stem separation alone is **$0.10-$1.00/song**, and LALAL.AI vocals/instrumental separation costs **$0.50-$0.60/song**. The break-even where running our own GPU pipeline beats commercial APIs occurs almost immediately - even 100 songs/month on Replicate ($1-2/mo) is cheaper than a $7.50/mo LALAL.AI Lite plan. At 10,000 transcriptions/month, the open-source pipeline costs roughly **$80-$200/mo** in GPU compute (vs. $20,000+ for Klangio), making vertical integration the obvious long-term play. Mobile on-device inference (Core ML / ONNX) is effectively free per inference but costs ~2-4 weeks of engineering and yields lower-quality stems.

## Cloud GPU Pricing (2026)

All prices are on-demand, single-GPU, USD. Spot/community rates roughly 40-50% lower.

### Hourly On-Demand (Reserved/Always-On)

| Provider          | GPU                | $/hour  | $/second   | Notes |
|-------------------|--------------------|---------| -----------|-------|
| AWS EC2 g5.xlarge | A10G (24GB)        | $1.006  | $0.000279  | 4 vCPU, 16 GB RAM |
| AWS EC2 g6.xlarge | L4 (24GB)          | ~$0.805 | ~$0.000224 | Better $/perf for inference vs g5 |
| AWS EC2 g6e.xlarge| L40S (48GB)        | $1.861  | $0.000517  | High-VRAM workloads |
| GCP G2 (L4)       | L4 (24GB)          | ~$0.71  | ~$0.000197 | Comparable to g6 |
| Lambda Labs       | A100 80GB          | $1.29   | $0.000358  | No spot, regional limits |
| Lambda Labs       | H100 PCIe          | $2.49   | $0.000692  | |
| Lambda Labs       | H100 SXM           | $3.29   | $0.000914  | |
| RunPod Secure     | RTX 4090 (24GB)    | $0.69   | $0.000192  | |
| RunPod Community  | RTX 4090 (24GB)    | $0.34   | $0.000094  | Spot - preemptible |
| RunPod Secure     | A100 80GB          | $1.89   | $0.000525  | |
| RunPod Community  | A100 80GB          | $1.19   | $0.000331  | |
| fal.ai            | H100               | $1.89   | $0.000525  | Output-priced for many models |

### Per-Second Serverless (Pay only for inference time)

| Provider  | GPU         | $/second    | Implied $/hour | Notes |
|-----------|-------------|-------------|----------------|-------|
| Replicate | T4          | $0.000225   | $0.81          | Cheapest entry; 16GB |
| Replicate | L40S        | $0.000975   | $3.51          | 48GB |
| Replicate | A100 80GB   | $0.001400   | $5.04          | |
| Modal     | T4          | $0.000164   | $0.59          | $30/mo free credits |
| Modal     | L4          | $0.000222   | $0.80          | |
| Modal     | A10         | $0.000306   | $1.10          | |
| Modal     | L40S        | $0.000542   | $1.95          | |
| Modal     | A100 40GB   | $0.000583   | $2.10          | |
| Modal     | H100        | $0.001097   | $3.95          | |

**Key takeaway:** for sub-30-second jobs, serverless (Modal/Replicate) wins by avoiding idle billing; for sustained throughput, RunPod Community A100 ($1.19/hr) or AWS g6 reserved ($0.40-0.60/hr w/ 1-yr commit) is cheapest.

## Audio ML Inference Performance (Reference)

- **Demucs v4 (HTDemucs)** on RTX 3090 / A100: ~5 sec wall-clock for a full 3-min track (FP16). On L4/A10: budget ~10-15 sec. On T4: ~25-40 sec.
- **Basic Pitch** (Spotify) is small (~17 MB) and runs near real-time on CPU; on a GPU it adds ~2-5 sec for a 3-min track.
- **MT3 / YourMT3+** is heavier - typically ~30-60 sec for 3-min on A100, can need ~10GB VRAM, processes audio in 20-sec chunks.

## Estimated Cost per Song

Assume the canonical MVP pipeline: **Demucs v4 separation + Basic Pitch transcription** on a 3.5-minute (210-sec) song. Add ~5 sec overhead (cold-start padding, file I/O).

### Open-source pipeline on serverless GPU

| Setup                              | Total inference seconds | Rate ($/sec) | Cost/song |
|------------------------------------|-------------------------|--------------|-----------|
| Modal L4 (Demucs+BP)               | ~20 s                   | $0.000222    | **$0.0044** |
| Modal A10                          | ~12 s                   | $0.000306    | **$0.0037** |
| Replicate T4                       | ~45 s                   | $0.000225    | **$0.010**  |
| Replicate A100 80GB                | ~10 s                   | $0.001400    | **$0.014**  |
| Modal H100                         | ~7 s                    | $0.001097    | **$0.0077** |

**MVP recommendation: Modal L4 or A10** at roughly **$0.004-$0.005/song** of compute. Add ~50% buffer for cold starts, container init, and queueing overhead → budget **$0.007-$0.010/song**.

If using **MT3 instead of Basic Pitch** (better polyphonic accuracy), inference time roughly triples → **$0.015-$0.04/song**.

### Storage and bandwidth (per song)

- Average 3.5-min MP3 @ 192 kbps ≈ **5 MB**; WAV/FLAC ≈ 35-40 MB.
- **Cloudflare R2:** $0.015/GB/mo storage, **$0 egress**. 5 MB file stored 1 month = $0.000075. Egress free.
- **AWS S3:** $0.023/GB/mo + **$0.09/GB egress**. 5 MB file = $0.000115 storage + $0.00045 egress = $0.0006 if downloaded once.
- **Verdict:** R2 saves ~10x on bandwidth-heavy workloads. Storage cost is negligible per song; bandwidth dominates only if users re-download stems.

### Commercial APIs (per 3.5-min song)

| Service     | Pricing model                                | Cost per 3.5-min song |
|-------------|----------------------------------------------|-----------------------|
| **Klangio** | ~$4 per individual transcription; subs ~$15-50/mo for 50-250 tickets | **~$2-$4** end-user; API price requires direct contact |
| **LALAL.AI**| ~$0.15/stem-minute (Pro $15/mo for 250 fast min); 4-stem at high quality consumes ~8-12 min from quota | **$0.53** (vocals/instr only) - **$2.10** (4-stem hi-q) |
| **AudioShake (API)** | "cents per minute" tier; one-off ~$1/min web; T&A 1.5 credits/min | **~$0.10-$0.50** API rate; **~$3.50** at one-off web rate |
| **Moises**  | No public per-track API; subscriptions $3.99-$9.99/mo unlimited | Effectively unmetered for end users; not a B2B option |

**Note:** Commercial APIs only do *part* of the pipeline (separation OR transcription). To replicate our full pipeline you'd need both (e.g. AudioShake + Klangio), stacking costs to **$2-$4.50/song**.

## Scaling Scenarios

Assumptions: 3.5-min average song, full pipeline (Demucs + Basic Pitch). Storage on R2. Modal L4 serverless at $0.005/song compute + $0.002/song overhead = **~$0.007/song marginal**.

### 100 transcriptions/month (early MVP)

- **Open-source on Modal:** 100 × $0.007 = **$0.70 compute + $0.10 storage/bandwidth ≈ $0.80/mo**.
- **Modal $30/mo credits cover all of it for free.**
- **Commercial API:** Klangio Universe sub at $50-70/mo, or ~$400/mo if paying per-song. LALAL.AI Pro $15/mo (caps at ~70 songs of 4-stem hi-q).
- **Verdict:** open-source is free-to-near-free at this scale.

### 1,000 transcriptions/month (growing user base)

- **Open-source on Modal:** 1,000 × $0.007 = **$7 compute**. Add storage ($1) + bandwidth (free on R2) = **~$10/mo all-in**.
- **Open-source on RunPod always-on RTX 4090** ($0.69/hr): one GPU running 24/7 costs $497/mo and can process ~17,000 songs/mo at ~10 sec each → only worth it past ~5k songs/mo. **Stay serverless.**
- **Commercial API:** AudioShake API ~$0.10/min × 3.5 min = $0.35/song → **$350/mo** (separation only). With Klangio for transcription, **$2,000-$4,000/mo total**.
- **Verdict:** open-source is **30-400x cheaper**.

### 10,000 transcriptions/month (real product)

- **Open-source serverless:** 10,000 × $0.007 = **$70 compute**. Add R2 storage at 50 GB ≈ $1/mo. **~$75-$100/mo all-in.**
- **Open-source dedicated:** 1× RTX 4090 on RunPod 24/7 = $497/mo, handles ~17k jobs/mo with headroom. Add 1 spare for redundancy + traffic spikes → **$1,000/mo capped, lower per-song marginal cost**.
- **Hybrid (recommended):** Reserved baseline + serverless burst → roughly **$200-$400/mo**.
- **Commercial API:** AudioShake at ~$0.10-$0.35/song = **$1,000-$3,500/mo** (sep only). Klangio/Tony at retail ~$20,000/mo. Negotiated enterprise deals ~30-50% cheaper.
- **Verdict:** serverless open-source still wins; consider reserved capacity once peak QPS justifies it.

### Frontier scale: 100,000+/month

- Demucs in TensorRT FP16 at ~2.7 sec GPU compute per 3-min song means a single A100 can do ~1,000 songs/hour. 100k songs/mo = ~100 GPU-hours = **~$130-$300/mo** at A100 spot rates.
- Storage and egress (R2) become the dominant cost only if average file size is large or users repeatedly download.

## Mobile vs. Server Inference Trade-offs

| Dimension              | On-device (Core ML / ONNX)                | Server-side (cloud GPU)             |
|------------------------|-------------------------------------------|-------------------------------------|
| Per-inference cost     | **$0** (uses user's battery)              | $0.005-$0.04                        |
| Engineering effort     | High - Core ML conversion, quantization, model size limits, separate pipelines per OS | Low - Python on Linux GPU |
| Model selection        | Limited: Basic Pitch yes; Demucs requires aggressive quantization, MT3 not feasible | Any model |
| Latency                | 10-30 sec on modern phones for Basic Pitch; minutes for Demucs | 5-20 sec end-to-end |
| Privacy story          | Strong - audio never leaves device        | Weak - need ToS, retention policy   |
| Quality                | Lower (smaller/quantized models)          | Best available                       |
| Offline support        | Yes                                       | No                                   |

**Recommendation:** server-side for MVP. Revisit on-device after PMF; consider Basic Pitch on-device for "instant preview" UX with server-side high-quality re-transcribe in background.

## Recommendation

1. **Build MVP on Modal serverless L4 / A10** with Demucs v4 + Basic Pitch. Marginal cost ~$0.007/song; first 100-1000 users essentially free under the $30/mo credit.
2. **Use Cloudflare R2** for audio storage and stem delivery. Egress savings vs S3 are significant once we ship stems back to users.
3. **Skip commercial APIs.** They are 100-1000x more expensive per song and only solve part of the pipeline. Useful only for very early prototyping (<50 songs total).
4. **Plan a migration to RunPod Community / reserved A100** once steady-state load exceeds ~5,000 songs/month; expected savings ~50-70%.
5. **Defer mobile inference** until product-market fit; the engineering cost dwarfs the GPU bill at MVP scale.
6. **Budget for ops:** factor in ~$50-$100/mo for monitoring, queue (e.g. SQS / Cloudflare Queues), and a small CPU API server (~$20/mo) on top of inference compute.

### Headline numbers

- **MVP cost per song:** ~$0.007 (compute + storage)
- **At 1k songs/mo:** ~$10/mo total infra
- **At 10k songs/mo:** ~$100/mo serverless or ~$300/mo on reserved hardware
- **Commercial-API equivalent at 10k/mo:** $1,000-$20,000/mo

## Sources

- [AWS EC2 G5 Instances](https://aws.amazon.com/ec2/instance-types/g5/)
- [AWS EC2 G6e Instances](https://aws.amazon.com/ec2/instance-types/g6e/)
- [g5.xlarge pricing - Vantage](https://instances.vantage.sh/aws/ec2/g5.xlarge)
- [g6e.xlarge pricing - Vantage](https://instances.vantage.sh/aws/ec2/g6e.xlarge)
- [RunPod Pricing](https://www.runpod.io/pricing)
- [Lambda Labs Pricing](https://lambda.ai/pricing)
- [Modal Pricing](https://modal.com/pricing)
- [Replicate Pricing](https://replicate.com/pricing)
- [fal.ai](https://fal.ai/)
- [Cloudflare R2 Pricing](https://developers.cloudflare.com/r2/pricing/)
- [LALAL.AI Pricing](https://www.lalal.ai/pricing/)
- [AudioShake Developers](https://developer.audioshake.ai/)
- [AudioShake Pricing breakdown - Oreate AI](https://www.oreateai.com/blog/unpacking-audioshake-ai-pricing-what-you-need-to-know/b64a1eb29db9373de0f82a976554b4c3)
- [Klangio Subscriptions](https://klang.io/help/klangio-subscriptions/)
- [Moises pricing](https://studio.moises.ai/billing/pricing)
- [Demucs v4 TensorRT benchmarks](https://huggingface.co/MansfieldPlumbing/Demucs_v4_TRT)
- [Demucs Apple Silicon performance writeup](https://medium.com/@andradeolivier/i-ported-demucs-to-apple-silicon-it-separates-a-7-minute-song-in-12-seconds-6c4e5cffb5c3)
- [Spotify Basic Pitch GitHub](https://github.com/spotify/basic-pitch)
- [MT3 GitHub](https://github.com/magenta/mt3)
- [GPU Cloud Pricing Comparison 2026 - Spheron](https://www.spheron.network/blog/gpu-cloud-pricing-comparison-2026/)
- [Cheapest cloud GPU providers 2026 - Northflank](https://northflank.com/blog/cheapest-cloud-gpu-providers)
- [H100 rental price comparison 2026 - IntuitionLabs](https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison)
- [Modal Pricing 2026 breakdown - Morph](https://www.morphllm.com/modal-pricing)
