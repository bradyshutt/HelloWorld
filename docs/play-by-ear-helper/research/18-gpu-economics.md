# Self-Hosted GPU vs Cloud Inference Economics

## Summary
For Play by Ear Helper, where each transcription needs ~10-30 GPU-seconds of Demucs + ~1-3 CPU-seconds of Basic Pitch on a 3-minute song, **serverless GPU (Modal or RunPod) is the right choice for 0-3,000 transcriptions/day** because per-second billing and scale-to-zero crush idle costs at low volume. **Reserved cloud instances (Lambda, AWS g6/g5)** become cheaper around **3,000-10,000/day**, where utilization on a single GPU exceeds ~25-30%. **Self-hosted/colocated RTX 4090 or 5090** hardware breaks even against Lambda H100 rentals in roughly **6-9 months** of continuous use, so it makes sense only at **10,000+ transcriptions/day** sustained, or as a developer's local machine. **CPU-only inference is feasible for Basic Pitch (3-5s for a 3-min song) but not Demucs (45-120s)**, so the realistic CPU-only budget option means swapping Demucs for Spleeter or skipping separation. **On-device (Core ML / TFLite)** is currently feasible for Basic Pitch on iOS/Android but not for Demucs at acceptable latency on mid-tier phones — revisit in 12-18 months as Apple Neural Engine and Tensor Core mobile chips improve.

## Pricing Comparison (May 2026)

### Serverless GPU platforms (per-second, scale-to-zero)
| Provider | GPU | $/hour | $/second | Cold start | Notes |
|---|---|---|---|---|---|
| Modal | T4 16GB | ~$0.59 | $0.000164 | 2-4 s | Warm pool of base containers; NVMe-cached weights |
| Modal | L4 24GB | ~$0.80 | $0.000222 | 2-4 s | Best inference value for medium models |
| Modal | A10G 24GB | ~$1.10 | $0.000306 | 2-4 s | |
| Modal | A100 40GB | ~$2.10 | $0.000583 | 2-4 s | |
| Modal | A100 80GB | ~$2.50 | $0.000694 | 2-4 s | |
| Modal | H100 80GB | ~$3.95 | $0.001097 | 2-4 s | GPU memory snapshots can give 10x faster cold start (alpha) |
| RunPod Serverless | RTX 4090 24GB | ~$0.69-2.12 ($/hr equiv) | $0.00019-$0.00059 | <2 s w/ FlashBoot, 6-12 s otherwise | Cheapest 4090 serverless; Community vs Secure tiers |
| RunPod Serverless | A100 80GB | ~$1.89 (Secure) | $0.000525 | 1-2 s w/ FlashBoot | 95th-percentile cold start <2.3 s with FlashBoot |
| RunPod Serverless | H100 80GB | ~$2.59-2.99 | $0.000719-$0.000831 | 1-2 s w/ FlashBoot | |
| Replicate | A100 80GB | $5.04 | $0.001400 | 8-15 s typical, 60+ s for custom models | Highest cold start; easiest deployment |
| Replicate | H100 80GB | $5.99 | $0.001664 | 8-15 s | |
| fal.ai | A100 80GB | ~$0.99 (custom deploy) | $0.000275 | Sub-second on hot models | Mostly per-output billing for packaged models |
| fal.ai | H100 80GB | ~$1.89 (custom deploy) | $0.000525 | Sub-second on hot models | |
| Beam | A100 40GB | ~$1.40 | $0.000389 | 2-3 s | Hot reload, Python-first SDK |
| Beam | H100 80GB | ~$3.15 | $0.000875 | 2-3 s | |
| Banana | — | — | — | — | **Sunset March 31, 2024 — no longer available** |

### Cloud GPU rental (hourly, on-demand and reserved)
| Provider | Instance / GPU | $/hour on-demand | 1-yr reserved | 3-yr reserved |
|---|---|---|---|---|
| AWS | g6.xlarge (1× L4 24GB) | $0.8048 | ~$0.55 (~32% off) | ~$0.32 (~60% off) |
| AWS | g5.xlarge (1× A10G 24GB) | $1.006 | ~$0.65 | ~$0.38 |
| AWS | g5.4xlarge | $1.624 | — | — |
| AWS | p4d.24xlarge (8× A100 40GB) | ~$32.77 | — | up to 62% off |
| AWS | p5.48xlarge (8× H100 80GB) | ~$98.32 | — | up to 62% off |
| GCP | g2-standard-8 (1× L4) | ~$0.71 | — | — |
| GCP | a2-highgpu-1g (1× A100 40GB) | ~$3.67 | up to 57% off w/ CUD | |
| GCP | a2-highgpu-8g (1× A100 40GB share) | ~$3.28/GPU-hr | | |
| GCP | a2-ultragpu (1× A100 80GB) | ~$5.07 | | |
| Azure | NC24ads A100 v4 (1× A100 80GB) | ~$3.67 | up to ~50% off RI | |
| Azure | NC96ads A100 v4 (4× A100 80GB) | ~$31.93 raw VM / $39.91 ML | | |
| Lambda Cloud | 1× A100 40GB | $1.29 | — | — |
| Lambda Cloud | 1× A100 80GB | $1.48 | — | — |
| Lambda Cloud | 1× H100 PCIe | $2.49-2.86 | — | — |
| Lambda Cloud | 1× H100 SXM | $3.29-3.78 | — | — |
| Lambda Cloud | 1× B200 | $6.08 | — | — |

### Self-hosted / colocated dedicated hardware
| Option | Up-front | Monthly | Notes |
|---|---|---|---|
| Buy RTX 4090 (used) | ~$1,099-2,400 | electricity ~$25-50 (US, $0.14-0.20/kWh, 50% util) | 24 GB VRAM, 450 W TDP |
| Buy RTX 5090 (new) | ~$2,900-4,500 | electricity ~$30-60 (575 W TDP, 50% util) | 32 GB VRAM, 575 W TDP |
| Rent dedicated 1× RTX 4090 server | $0 | $409-479/mo | Hostkey / GPU-Mart, 24 GB VRAM |
| Rent dedicated 2× RTX 4090 server | $0 | $609-859/mo | |
| Rent dedicated 1× RTX 5090 server | $0 | $479-670/mo | Spot-style providers as low as $96/mo (preemptible) |
| Colocate own 4090 box | ~$2,500 build | $50-150/mo (1U/2U + power) | DIY route — biggest savings if utilization is high |

## Cost per Transcription (3-min song)

Assumed pipeline per song:
- Demucs htdemucs_ft separation: ~6 s on H100 / ~10 s on A100 / ~12-15 s on RTX 4090 / ~8 s on Apple Silicon M-series / 90-180 s on a modern x86 CPU
- Basic Pitch transcription: ~1-3 s on CPU, sub-second on GPU
- Total: assume **20 GPU-seconds** for a comfortable upper bound that includes I/O and overhead, plus **2 CPU-seconds** of pre/post.

| Approach | Per-transcription compute cost | Notes |
|---|---|---|
| Modal H100 (20 s) | 20 × $0.001097 = **$0.0219** | Plus ~$0.001 CPU/RAM and 2-4 s cold start every cluster expansion |
| Modal A100 80GB (30 s) | 30 × $0.000694 = **$0.0208** | Slightly slower, similar cost |
| Modal L4 (45 s) | 45 × $0.000222 = **$0.0100** | Best Modal price/transcription if quality holds at L4 |
| RunPod Serverless RTX 4090 (35 s incl. cold start) | 35 × $0.00019 = **$0.0067** (community), $0.0207 (secure) | Cheapest serverless GPU per song; 24 GB is plenty for Demucs |
| RunPod Serverless H100 (22 s) | 22 × $0.000719 = **$0.0158** | |
| Replicate A100 80GB (40 s incl. higher cold start) | 40 × $0.0014 = **$0.056** | Easiest to deploy; 2-3× the cost |
| fal.ai H100 custom deploy (22 s) | 22 × $0.000525 = **$0.0116** | If you want their fast network/CDN |
| AWS g6.xlarge L4 on-demand (45 s) | 45/3600 × $0.8048 = **$0.0101** | Only competitive if instance stays busy |
| AWS g6.xlarge L4 3-yr RI (45 s, 100% util) | 45/3600 × $0.32 = **$0.004** | Requires near-constant utilization to realize |
| Lambda H100 PCIe (22 s, 100% util) | 22/3600 × $2.86 = **$0.0175** | At 50% util, doubles to ~$0.035 |
| Lambda H100 PCIe (22 s, 25% util) | $0.07 | Idle time eats the budget at low volume |
| Self-hosted 4090, 80% utilization, 3-yr amortization | (~$2,000 / 3 yr / 365 d / 24 h) + electricity ≈ **$0.10/hr all-in** → 22 s @ 4090 = $0.0006 | Unbeatable per-song, but only if you actually fill the hours |
| CPU-only (no Demucs, Basic Pitch only, 3 s on AWS c7i.large $0.085/hr) | $0.000071 | Drops separation entirely; quality regression for polyphonic input |
| CPU-only Demucs (120 s on AWS c7i.2xlarge $0.357/hr) | **$0.0119** | Surprisingly competitive in dollars but kills latency |
| On-device (iPhone 15 Pro / Pixel 8) | **$0** marginal | Requires Core ML / TFLite ports; current Demucs port runs ~34× realtime on M-series, mobile NPU 5-10× realtime is plausible |

## Breakeven Analysis

### Serverless GPU vs reserved hourly instance
Lambda H100 PCIe at $2.86/hr = $0.000794/s. Modal H100 at $0.001097/s. Modal is 38% more expensive per active second. Lambda is cheaper **only when utilization ≥ ~72%** of the hour (otherwise idle pay erases the per-second savings). For Play by Ear Helper at 22 GPU-seconds/song:
- **22 s × N songs/hour ≥ 0.72 × 3,600 s** → N ≥ **118 songs/hour** → **~2,800/day** of constant load.
- Below ~2,800/day, Modal/RunPod serverless wins.
- At 2,800-10,000/day, a single reserved H100 (or A100) on Lambda saturates and beats serverless.

Same math vs RunPod 4090 serverless ($0.00019/s) is harder to beat:
- A self-hosted/dedicated RTX 4090 ($479/mo dedicated rental ≈ $0.665/hr ≈ $0.000185/s) only wins above ~97% utilization. **RunPod 4090 serverless is a near-perfect match** for sub-10k/day workloads.

### Self-hosted RTX 4090 vs Lambda H100
- Buy used 4090: ~$1,500. Build cost: ~$2,000 chassis + PSU + CPU. Total **~$3,500**.
- Power: 450 W × 50% util × 24 h × 30 d × $0.15/kWh ≈ **$24/mo**. Bandwidth + colo ~$100/mo.
- 3-year all-in: $3,500 + 36 × $124 ≈ **$7,964**, or **$0.30/hr** average.
- Lambda H100 at $2.86/hr × 24 × 30 × 36 = **$74,131** if on 24/7.
- Crossover: self-host pays back in **~3 months at 100% utilization**, **~6 months at 50%**, **~12 months at 25%**.
- Note that an RTX 4090 has roughly 1/3 the H100's BF16 throughput on Demucs-style workloads, so for fair comparison you'd need ~3 × 4090s ≈ $9-10k all-in to match one H100. Even then, payback < 12 months at high utilization.

### Self-hosted vs RunPod Community 4090 ($0.34/hr)
- $0.34/hr × 8,760 = $2,978/yr. Buying a 4090 + chassis ($3,500) takes ~14 months to break even on hardware alone, and ignores depreciation, downtime, and ops effort. For most teams, **RunPod Community 4090 wins** until you're saturating multiple GPUs simultaneously.

### CPU vs GPU
- AWS c7i.2xlarge: $0.357/hr, runs Demucs at ~120 s/song = $0.0119 per song.
- Modal H100 serverless: $0.0219 per song but **5× faster wall-clock** (20 s vs 120 s).
- For interactive UX (user waiting <10 s), **GPU is required**. For batch / async ("we'll email you the score in 5 minutes"), CPU is viable and competitive.

## Recommendation by Stage

### MVP (0-100 transcriptions/day) — **RunPod Serverless RTX 4090 (Community tier) or Modal L4**
- **Why:** Scale-to-zero kills idle cost. At 100/day × 22 s = 2,200 GPU-s/day. RunPod Community 4090 at $0.00019/s = **$0.42/day = $12.60/mo**. Modal L4 = **$14.70/mo**. Replicate A100 = $5.60/day = **$168/mo** — pay for the easy deploy if engineering time is the constraint.
- Cold start matters less here because users are already accustomed to a few-second wait. Use Modal `min_containers=1` during business hours if you want sub-2s p99 latency for **~$60/mo extra**.
- **Avoid:** AWS g5/g6 on-demand (idle dominates), self-hosted (over-engineering for the volume), Replicate for production unless engineering is single-handed and cost-insensitive.

### Growth (100-10,000/day) — **Modal H100 or RunPod Serverless 4090, with reserved instance for the steady-state floor**
- **100-1,000/day:** Stay fully serverless. At 1,000/day × 22 s on RunPod 4090 Secure = 22,000 s/day × $0.00059 = **$13/day = $390/mo**. Modal H100 = $24/day = **$720/mo**.
- **1,000-3,000/day:** Still serverless, but consider Modal `min_containers` warm pool for latency. Cost will be ~$1,000-2,500/mo.
- **3,000-10,000/day:** Hybrid — keep one Lambda H100 PCIe ($2.86/hr × 730 = **$2,088/mo**) as the always-on baseline handling the steady load, and burst overflow to Modal/RunPod serverless. At ~3,000/day a single H100 hits ~25% utilization; at 10,000/day it's ~80% — perfect saturation point. Total bill ~$2,500-3,500/mo for 10k/day vs ~$5,000-6,500/mo on pure serverless.

### Scale (10,000+ transcriptions/day) — **Reserved Lambda / Lambda 1-year contracts, or self-hosted/colocated**
- At 10k/day = 220k GPU-s/day. One H100 at full utilization handles 86k s/day — **need ~3-4 H100s** worth of capacity. Lambda 1-yr commitments ~$1.99-2.50/hr/H100, totaling **$4,300-5,500/mo for 3 H100s**.
- Alternative: rent or buy 6-8 RTX 4090 / 5090 nodes. Hostkey 2× 4090 dedicated at $609/mo × 4 nodes = **$2,436/mo** with redundancy. Or buy 8 used 4090s + chassis + colo for ~$25-35k up front, then $400-600/mo running cost — **payback ~6-9 months** at this scale.
- At **50,000+/day (sustained)**, self-hosted/colocated wins decisively; expect ~70-85% lower TCO than any cloud option, at the cost of ops headcount.

### Cross-cutting
- **CPU fallback:** Always have a CPU-only path for Basic Pitch (Spotify ships an ONNX runtime that runs <5 s on a laptop CPU). Lets you keep service alive when GPU pool is exhausted, and is ~free per song.
- **On-device (iOS/Android Core ML / TFLite):** Track this for V2. A Core ML port of Basic Pitch already exists; Demucs Apple Silicon port runs at 34× realtime on M-series. iPhone 15 Pro / Pixel 8 ANEs can plausibly hit 5-10× realtime on a quantized Demucs in 2026, making fully on-device pipelines feasible. This eliminates per-inference cost and gives offline mode, but development cost is ~2-3 engineer-months extra and quality at INT8 needs validation.
- **Don't use Replicate at any scale beyond demo** — its $0.0014/s A100 rate and 60+s custom-model cold starts make it 2-3× more expensive than alternatives.
- **Don't use Banana** — sunset March 2024.

## Sources
- Modal pricing 2026: https://modal.com/pricing and https://www.morphllm.com/modal-pricing
- Modal cold start docs: https://modal.com/docs/guide/cold-start
- RunPod pricing 2026: https://www.runpod.io/pricing and https://docs.runpod.io/serverless/pricing
- RunPod FlashBoot cold start: https://www.runpod.io/blog/introducing-flashboot-serverless-cold-start
- Replicate / serverless comparison: https://hostfleet.net/serverless-gpu-pricing-matrix-2026/
- 9 Best Serverless GPU Providers 2026: https://blog.premai.io/9-best-serverless-gpu-providers-for-llm-inference-2026/
- fal.ai pricing: https://fal.ai/pricing and https://cloudgpuprices.com/vendors/fal-ai
- Banana sunset: https://thinkpeak.ai/banana-serverless-gpu-pricing-2026/
- AWS EC2 on-demand pricing: https://aws.amazon.com/ec2/pricing/on-demand/
- AWS GPU pricing breakdown: https://wring.co/blog/aws-gpu-instance-pricing-guide
- GCP GPU pricing: https://cloud.google.com/compute/gpus-pricing
- Cloud GPU comparison AWS/Azure/GCP: https://www.cloudzero.com/blog/cloud-gpu-pricing-comparison/
- Lambda Labs pricing: https://lambda.ai/pricing and https://computeprices.com/providers/lambda
- H100 cross-cloud comparison: https://intuitionlabs.ai/articles/h100-rental-prices-cloud-comparison
- A100 pricing 2026: https://jarvislabs.ai/blog/a100-price
- RTX 4090 / 5090 retail: https://bestvaluegpu.com/history/new-and-used-rtx-4090-price-history-and-specs/ and https://bestvaluegpu.com/history/new-and-used-rtx-5090-price-history-and-specs/
- Dedicated 4090/5090 server pricing: https://www.gpu-mart.com/pricing and https://hostkey.com/gpu-dedicated-servers/
- RTX 4090 power consumption: https://www.ecoenergygeek.com/rtx-4090-power-consumption/
- Demucs CPU vs Apple Silicon benchmark: https://medium.com/@andradeolivier/i-ported-demucs-to-apple-silicon-it-separates-a-7-minute-song-in-12-seconds-6c4e5cffb5c3
- Demucs TensorRT optimization: https://huggingface.co/MansfieldPlumbing/Demucs_v4_TRT
- Whisper serverless cold start (Spheron): https://www.spheron.network/blog/whisper-v4-asr-gpu-cloud-production-guide/
- Mobile transcription / Core ML / TFLite: https://www.ionio.ai/blog/running-transcription-models-on-the-edge-a-practical-guide-for-devices and https://github.com/umitkacar/awesome-mobile-ai
- Basic Pitch: https://github.com/spotify/basic-pitch and https://github.com/sevagh/basicpitch.cpp
- Spheron 2026 GPU comparison: https://www.spheron.network/blog/gpu-cloud-pricing-comparison-2026/
- Synpix 2026 cloud GPU pricing: https://www.synpixcloud.com/blog/cloud-gpu-pricing-comparison-2026
- Northflank cheapest cloud GPU 2026: https://northflank.com/blog/cheapest-cloud-gpu-providers
